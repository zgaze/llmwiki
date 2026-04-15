---

tags: 
  - 分布式
  - kafka

---
### kafka

#### 组织结构

kafka使用topic组织数据，一个topic包含多个partition，每个partition可能多副本，其中有一个leader，所有的读写都是通过这个副本。follower副本的作用体现在leader崩溃。 副本都被保存在broker上，但并非一一对应，一个broker可以存储多个topic的数据。

#### replica

每个partition可能多副本，其中有一个leader，记录在zk，所有的读写都是通过这个副本。follower副本的作用体现在leader崩溃。

* follower作为消费者，从leader 里消费数据，更新自身，保存和leader一致。通过offset可以看是不是最新的。
* leader崩溃后，只有最新的才能成为leader。
* 副本数量不能超过broker数量
* 一个Partition只能被一个Consumer消费，Partition的个数决定了最大并行度。

#### broker

broker就是kafka server，负责处理客户端、replica、controller发送的请求。包含元数据请求、消费请求、写入请求。

* broker写用mmap，发用sendfile。
* broker和客户端，broker和broker都使用同一协议，请求类型不同。

#### ISR

* 跟上节奏就加入ISR，跟不上节奏就踢出ISR。kafka的消费复制不是同步，也不是完全异步，而是通过ISR+异步的方式实现的。为了需求效率和可靠性的平衡。
* replica.lag.time.max，落后一段时间的follower被提出ISR，加入OSR。all只需要所有ISR里面的commit就可以，lag追上leader的follower会重新加入ISR。

> producer 发送一条消息到leader节点后， 只有当ISR中所有Replica都向Leader发送ACK确认这条消息时，leader才commit，这时候producer才能认为这条消息commit了，正是因为如此，kafka客户端的写性能取决于ISR集合中的最慢的一个broker的接收消息的性能，如果一个点性能太差，就必须尽快的识别出来，然后从ISR集合中踢出去，以免造成性能问题。

#### controller

Kafka集群中多个broker，有一个会被选举为controller leader，负责管理整个集群中分区和副本的状态，管理集群brokers，partition leader，topic等。

* broker join/broker failure：通过监控 zk 的 /brokers/ids 目录变化，就会知道哪些 broker 掉线了，进而触发 broker 的下线操作/上线操作，可能某些分区需要重新选leader。
* topic creation：监控/brokers/topics这个目录来判断是否有新的 topic 需要创建
* topic deletion：监控 zk 的 /admin/delete\_topics 节点来触发 topic 删除操作
* partition reassignment：监控 zk 的 /admin/reassign\_partitions 节点来触发 Partition 的副本迁移操作
* preferred replica leader election：监控 zk 的 /admin/preferred\_replica\_election 节点来触发最优 leader 选举操作，该操作的目的选举 Partition 的第一个 replica 作为 leader
* topic partition expansion：监控 zk 的 /brokers/topics/<topic> 数据内容的变化，来触发 Topic 的 Partition 扩容操作
* controller leader election：集群中所有 broker 会监听 zk 的 /controller节点，如果该节点消失，所有的 broker 都回去抢占 controller 节点，抢占成功的，就成了最新的 controller
* controlled shutdown：Controller 通过处理 ControlledShudownRequest 请求来优雅地关闭一个 broker 节点，主动关闭与直接 kill 的区别，它可以减少 Partition 的不可用时间，因为一个 broker 的 zk 临时节点消失是需要一定时间的
* cluster metadata updates:producer 或 consumer 可以通过 MetadataRequest 请求从集群任何一台 broker 上查询到某个 Partition 的 metadata 信息，如果一个 Partition 的 leader 或 isr 等信息变化，Controller 会广播到集群的所有 broker 上，这样每台 Broker 都会有该 Partition 的最新 Metadata 信息

controller负责协调管理，具体功能由其他实现，比如： ReplicaManager负责管理当前broker所有分区和副本的信息，会处理KafkaController发起的一些请求，副本状态的切换，添加/读取消息等

#### kafka在zk注册的信息

* /brokers/ids/\[id\] 记录集群中的broker id
* /brokers/topics/\[topic\]/partitions 记录了topic所有分区分配信息以及AR集合
* /brokers/topics/\[topic\]/partitions/\[partition\_id\]/state 记录了某partition的leader副本所在brokerId,leader\_epoch, ISR集合,zk 版本信息
* /controller\_epoch 记录了当前Controller Leader的年代信息
* /controller 记录了当前Controller Leader的id，也用于Controller Leader的选择
* /admin/reassign\_partitions 记录了需要进行副本重新分配的分区
* /admin/preferred\_replica\_election：记录了需要进行"优先副本"选举的分区，优先副本在创建分区的时候第一个副本
* /admin/delete\_topics 记录删除的topic
* /isr\_change\_notification 记录一段时间内ISR列表变化的分区信息
* /config 记录的一些配置信息

### 物理存储

* kafka的基本存储单元是partition，log.dirs指定存储路径。
* 每个partition分成若干个片段,满1G或者过去一周,就换新的。这样就可以根据策略删除过期或者容量达到限制的文件。
* kafka的消息包含了offset、消息大小、校验和、版本号、压缩算法、时间戳等等。

#### 日志和索引数据

        

```
ba.cost.budget.shbt-engine.bool-0
ba.cost.budget.shbt-engine.bool-1
ba.cost.budget.shbt-engine.bool-2
...
```

* partition数据存在topicname+index，index为从0上升的数字，一个partition一个目录。
* partition数据里包含三类：xx.timeindex,xx.log,xx.index，分别是时间索引，log原始数据和索引。会有多个这样的文件。过期的被删除，然后又新建。index用来二分查找，找到对应文件的偏移offset->文件名和偏移，log用来顺序写入。

#### 偏移数据

* 旧版本的offset数据存在kafka，效率太低。新版本kafka专门位offset创建了一个\_\_consumer\_offsets的topic。
* 以消费的Group，Topic，以及Partition做为组合 Key，将offset写入上述的topic中。另外borker内存中，也会存储一份。

#### CAP

* kafka的三个副本挂的只剩一个了，也可以继续工作。如果没有完全同步的副本，leader挂了，可以允许不完全同步的副本成为leader，实现了高可用，但是会造成数据丢失。需要的话，可以禁用不完全首领选举。
* min.insync.replicas,最少同步副本，对于一个包含3副本的topic，最少写入几个副本，才会被认为正确commit。
* commit之后的消费，才对消费者可见。
* 只有禁用不完全首领选举，acks设置为all，然后客户端producer发现错误会重试，才能保证数据完整性。
* 重试一些情况下，会导致消息重复。不是幂等的。
* kafka只能保证at\_least\_once最少一次，即数据不会丢失，但是可能会重复。要做到exactly\_once刚好一次，需要借助外部系统，比如对消息里key做去重。
