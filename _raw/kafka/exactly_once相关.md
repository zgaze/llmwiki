---
---
## kafka幂等和事务

kafka 0.11.0.0后新支持的语义。

### 幂等基础：at\_least\_once

* kafka的三个副本挂的只剩一个了，也可以继续工作。如果没有完全同步的副本，leader挂了，可以允许不完全同步的副本成为leader，实现了高可用，但是会造成数据丢失。需要的话，可以禁用不完全首领选举。只有禁用不完全首领选举，acks设置为all，然后客户端producer发现错误会重试，才能保证数据完整性。
* min.insync.replicas,最少同步副本，对于一个包含3副本的topic，最少写入几个副本，才会被认为正确commit。commit之后的消费，才对消费者可见。
* 重试一些情况下，会导致消息重复。不是幂等的。
* kafka只能保证at\_least\_once最少一次，即数据不会丢失，但是可能会重复。要做到exactly\_once刚好一次，需要借助外部系统，比如对消息里key做去重。

### 幂等性

即producer的生产不会丢失，不会重复，即Exactly Once,需要在配置重试和enable.idempotence = true
at least once + 幂等 = exactly once

#### 幂等限制

* 只能保证 Producer 在单个会话内不丟不重，如果 Producer 出现意外挂掉再重启是无法保证的（幂等性情况下，是无法获取之前的状态信息，因此是无法做到跨会话级别的不丢不重）;
* 幂等性不能跨多个 Topic-Partition，只能保证单个 partition 内的幂等性，当涉及多个 Topic-Partition 时，这中间的状态并没有同步。
* 如果需要跨会话、跨多个 topic-partition 的情况，需要使用 Kafka 的事务性来实现。
* 需要producer设置enable.idempotence = true。

#### 幂等的实现

在分区的维度上去做，重复数据的判断让 partition 的 leader 去判断处理，前提是 Producer 请求需要把唯一键值告诉 leader；client + partition。 Producer 在实现时有以下两个重要机制：

* PID（Producer ID），用来标识每个 producer client,procucer创建时产生，Producer 故障后重新启动后会被分配一个新的 PID，这也是幂等性无法做到跨会话的一个原因。
* sequence numbers，client 发送的每条消息都会带相应的 sequence number，Server 端就是根据这个值来判断 数据是否重复。序列号将被持久化存储topic中，因此即使leader replica失败，接管的任何其他broker也将能感知到消息是否重复。
* kafka recordbatch包含以下2个字段，用来支撑幂等性:producer id,first sequence, producer epoch,是用来实现事务的，在幂等的基础上。
* pid段：/latest\_producer\_id\_block 存在zk。{"version":1,"broker":35,"block\_start":"4000","block\_end":"4999"}

#### 消费不能ExactlyOnce

在Consumer中，没有配置可以保证Exactly-Once语义。若要达到这个目标，需要在at-least-once的基础上实现幂等。这点和Producer是类似的，区别是Consumer的幂等性需要用户自己来完成。

* 如果offset是自动提交的，那么Consumer将不能再次消费这些数据（除非重启Consumer，并通过seek(TopicPartition, long)重置offset）。它表现出at-most-once语义。
* 在捕获异常后，如果手动提交offset，表现出at-most-once语义。如果不提交offset，Consumer可重复消费该消息，表现出at-least-once语义。

有两种常用的方法在Kafka之上来获得恰好一次的语义：

* 将偏移量存储在与派生状态相同的DB中，并在事务中更新两者。重新启动时，从DB读取当前偏移量，然后从偏移位置开始读取卡夫卡。
* 以幂等的方式将状态更新和偏移量一起写入。例如，如果您的派生状态是一个key和一个跟踪出现次数的计数器，则将偏移量与计数值一起存储，并忽略任何偏移量<=当前存储值的任何更新。

#### server端处理如何保证幂等性

* 先根据 batch 的 sequence number 信息检查这个 batch 是否重复（batchMetadata 会缓存最新 5个 batch 的数据，根据 batchMetadata 缓存的 batch 数据来判断这个 batch 是否为重复的数据。），如果有重复，这里当做写入成功返回
* 检查pid是否在缓存中存在：存在的话，检查PID epoch 与 server 端记录的是否相同；不存在的话，要seq从0开始才行，否则抛出异常。
* 检查无误后，写入

#### 幂等简单总结

* Server 端验证 batch 的 sequence number 值，不连续时，直接返回异常；
* Client 端请求重试时，batch 在 reenqueue 时会根据 sequence number 值放到合适的位置（有序保证之一）；
* Sender 线程发送时，在遍历 queue 中的 batch 时，会检查这个 batch 是否是重试的 batch，如果是的话，只有这个 batch 是最旧的那个需要重试的 batch，才允许发送，否则本次发送跳过这个 Topic-Partition 数据的发送等待下次发送。

### 事务

Kafka支持两种事务，单独的producer事务和接收-处理-发送事务，不支持单纯的consumer事务。 producer发的多条消息组成一个事务这些消息需要对consumer同时可见或者同时不可见。

#### 重要基础

* 因为producer发送消息可能是分布式事务，所以引入了常用的2PC，所以有事务协调者(Transaction Coordinator)。Transaction Coordinator和之前为了解决脑裂和惊群问题引入的Group Coordinator在选举和failover上面类似。
* 事务管理中事务日志是必不可少的，kafka使用一个内 部topic来保存事务日志，这个设计和之前使用内部topic保存位点的设计保持一致。事务日志是Transaction Coordinator管理的状态的持久化，因为不需要回溯事务的历史状态，所以事务日志只用保存最近的事务状态。
* 因为事务存在commit和abort两种操作，而客户端又有read committed和read uncommitted两种隔离级别，所以消息队列必须能标识事务状态，这个被称作Control Message。
* producer挂掉重启或者漂移到其它机器需要能关联的之前的未完成事务所以需要有一个唯一标识符来进行关联，这个就是TransactionalId，一个producer挂了，另一个有相同TransactionalId的producer能够接着处理这个事务未完成的状态。注意不要把TransactionalId和数据库事务中常见的transaction id搞混了，kafka目前没有引入全局序，所以也没有transaction id，这个TransactionalId是用户提前配置的。
* TransactionalId能关联producer，也需要避免两个使用相同TransactionalId的producer同时存在，所以引入了producer epoch来保证对应一个TransactionalId只有一个活跃的producer epoch

#### 事务实现

### 总结

* PID与Sequence Number的引入实现了写操作的幂等性
* 写操作的幂等性结合At Least Once语义实现了单一 Session 内的Exactly Once语义
* Transaction Marker与PID提供了识别消息是否应该被读取的能力，从而实现了事务的隔离性
* Offset 的更新标记了消息是否被读取，从而将对读操作的事务处理转换成了对写（Offset）操作的事务处理
* Kafka 事务的本质是，将一组写操作（如果有）对应的消息与一组读操作（如果有）对应的 Offset 的更新进行同样的标记（即Transaction Marker）来实现事务中涉及的所有读写操作同时对外可见或同时对外不可见
* Kafka 只提供对 Kafka 本身的读写操作的事务性，不提供包含外部系统的事务性
