---
source: http://blog.csdn.net/houlaizhe221/article/details/6580775
---
<http://blog.csdn.net/pbymw8iwm/article/details/7974789>
<http://blog.csdn.net/houlaizhe221/article/details/6580775>

非阻塞IO 和阻塞IO：

       在网络编程中对于一个网络句柄会遇到阻塞IO 和非阻塞IO 的概念, 这里对于这两种socket 先做一下说明：
       基本概念：

              阻塞IO::

                     socket 的阻塞模式意味着必须要做完IO 操作（包括错误）才会

                     返回。

              非阻塞IO::

                     非阻塞模式下无论操作是否完成都会立刻返回，需要通过其他方

                     式来判断具体操作是否成功。

IO模式设置：

                                               SOCKET
       对于一个socket 是阻塞模式还是非阻塞模式有两种方式来处理::

       方法1、fcntl 设置;用F\_GETFL获取flags,用F\_SETFL设置flags|O\_NONBLOCK;

       方法2、recv,send 系列的参数。(读取，发送时，临时将sockfd或filefd设置为非阻塞)

                                            方法一的实现

 fcntl 函数可以将一个socket 句柄设置成非阻塞模式:       flags = fcntl(sockfd, F\_GETFL, 0);                       //获取文件的flags值。

      fcntl(sockfd, F\_SETFL, flags | O\_NONBLOCK);   //设置成非阻塞模式；

      flags  = fcntl(sockfd,F\_GETFL,0);

      fcntl(sockfd,F\_SETFL,flags&~O\_NONBLOCK);    //设置成阻塞模式；

      设置之后每次的对于sockfd 的操作都是非阻塞的。

                                           方法二的实现

    recv, send 函数的最后有一个flag 参数可以设置成MSG\_DONTWAIT

             临时将sockfd 设置为非阻塞模式,而无论原有是阻塞还是非阻塞。

    recv(sockfd, buff, buff\_size,MSG\_DONTWAIT);     //非阻塞模式的消息发送

    send(scokfd, buff, buff\_size, MSG\_DONTWAIT);   //非阻塞模式的消息接受

                                                    普通文件

        对于文件的阻塞模式还是非阻塞模式::

        方法1、open时，使用O\_NONBLOCK；

        方法2、fcntl设置，使用F\_SETFL，flags|O\_NONBLOCK；

                                              消息队列

        对于消息队列消息的发送与接受::

        //非阻塞  msgsnd(sockfd,msgbuf,msgsize(不包含类型大小),IPC\_NOWAIT)

        //阻塞     msgrcv(scokfd,msgbuf,msgsize(\*\*),msgtype,IPC\_NOWAIT);

                                                                  读                

阻塞与非阻塞读的区别:  //阻塞和非阻塞的区别在于没有数据到达的时候是否立刻返回．

读(read/recv/msgrcv):

       读的本质来说其实不能是读,在实际中, 具体的接收数据不是由这些调用来进行,是由于系统底层自动完成的。read 也好,recv 也好只负责把数据从底层缓冲copy 到我们指定的位置.

       对于读来说(read, 或者recv) ::

阻塞情况下::

       在阻塞条件下，read/recv/msgrcv的行为::

       1、如果没有发现数据在网络缓冲中会一直等待，

       2、当发现有数据的时候会把数据读到用户指定的缓冲区，但是如果这个时候读到的数据量比较少，比参数中指定的长度要小，read 并不会一直等待下去，而是立刻返回。

       read 的原则::是数据在不超过指定的长度的时候有多少读多少，没有数据就会一直等待。

       所以一般情况下::我们读取数据都需要采用循环读的方式读取数据，因为一次read 完毕不能保证读到我们需要长度的数据，

       read 完一次需要判断读到的数据长度再决定是否还需要再次读取。

非阻塞情况下::

       在非阻塞的情况下，read 的行为::

       1、如果发现没有数据就直接返回，

       2、如果发现有数据那么也是采用有多少读多少的进行处理．

             所以::read 完一次需要判断读到的数据长度再决定是否还需要再次读取。

对于读而言::   阻塞和非阻塞的区别在于没有数据到达的时候是否立刻返回．
       recv 中有一个MSG\_WAITALL 的参数::

       recv(sockfd, buff, buff\_size, MSG\_WAITALL),
       在正常情况下recv 是会等待直到读取到buff\_size 长度的数据，但是这里的WAITALL 也只是尽量读全，在有中断的情况下recv 还是可能会被打断，造成没有读完指定的buff\_size的长度。

       所以即使是采用recv + WAITALL 参数还是要考虑是否需要循环读取的问题，在实验中对于多数情况下recv (使用了MSG\_WAITALL)还是可以读完buff\_size，

       所以相应的性能会比直接read 进行循环读要好一些。

注意::      //使用MSG\_WAITALL时，sockfd必须处于阻塞模式下，否则不起作用。

               //所以MSG\_WAITALL不能和MSG\_NONBLOCK同时使用。

       要注意的是使用MSG\_WAITALL的时候，sockfd 必须是处于阻塞模式下，否则WAITALL不能起作用。

                                                                         写 

阻塞与非阻塞写的区别:     //
写(send/write/msgsnd)::

       写的本质也不是进行发送操作,而是把用户态的数据copy 到系统底层去,然后再由系统进行发送操作,send，write返回成功，只表示数据已经copy 到底层缓冲,而不表示数据已经发出,更不能表示对方端口已经接收到数据.
       对于write(或者send)而言，

阻塞情况下::                 //阻塞情况下，write会将数据发送完。(不过可能被中断)

       在阻塞的情况下，是会一直等待，直到write 完，全部的数据再返回．这点行为上与读操作有所不同。

        原因::

              读，究其原因主要是读数据的时候我们并不知道对端到底有没有数据，数据是在什么时候结束发送的，如果一直等待就可能会造成死循环，所以并没有去进行这方面的处理；

              写，而对于write, 由于需要写的长度是已知的，所以可以一直再写，直到写完．不过问题是write 是可能被打断吗，造成write 一次只write 一部分数据, 所以write 的过程还是需要考虑循环write, 只不过多数情况下一次write 调用就可能成功.

非阻塞写的情况下::     //

       非阻塞写的情况下，是采用可以写多少就写多少的策略．与读不一样的地方在于，有多少读多少是由网络发送的那一端是否有数据传输到为标准，但是对于可以写多少是由本地的网络堵塞情况为标准的，在网络阻塞严重的时候，网络层没有足够的内存来进行写操作，这时候就会出现写不成功的情况，阻塞情况下会尽可能(有可能被中断)等待到数据全部发送完毕， 对于非阻塞的情况就是一次写多少算多少,没有中断的情况下也还是会出现write 到一部分的情况.

功能描述：根据文件描述词来操作文件的特性。
#include <unistd.h>
#include <fcntl.h>
int fcntl(int fd, int cmd);
int fcntl(int fd, int cmd, long arg);
int fcntl(int fd, int cmd, struct flock \*lock);

fcntl(fd, F\_GETFL)
fcntl(fd, F\_SETFL, val)

cmd值的F\_GETFL和F\_SETFL：

F\_GETFL    取得fd的文件状态标志。
F\_SETFL    设置给arg描述符状态标志，可以更改的几个标志是：O\_APPEND，O\_NONBLOCK，O\_SYNC 和 O\_ASYNC。
而fcntl的文件状态标志总共有7个：O\_RDONLY , O\_WRONLY , O\_RDWR , O\_APPEND , O\_NONBLOCK , O\_SYNC和O\_ASYNC
可更改的几个标志如下面的描述：
    O\_NONBLOCK   非阻塞I/O，如果read(2)调用没有可读取的数据，或者如果write(2)操作将阻塞，则read或write调用将返回-1和EAGAIN错误
    O\_APPEND     强制每次写(write)操作都添加在文件大的末尾，相当于open(2)的O\_APPEND标志
    O\_DIRECT     最小化或去掉reading和writing的缓存影响。系统将企图避免缓存你的读或写的数据。如果不能够避免缓存，那么它将最小化已经被缓存了的数据造成的影响。如果这个标志用的不够好，将大大的降低性能
    O\_ASYNC      当I/O可用的时候，允许SIGIO信号发送到进程组，例如：当有数据可以读的时候
在修改文件描述符标志或文件状态标志时必须谨慎，先要取得现在的标志值，然后按照希望修改它，最后设置新标志值。不能只是执行F\_SETFD或F\_SETFL命令，这样会关闭以前设置的标志位。
