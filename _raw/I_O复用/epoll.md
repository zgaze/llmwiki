---
source: http://blog.csdn.net/ljx0305/article/details/4065058
---
epoll的接口非常简单，一共就三个函数：
1\. int epoll\_create(int size);
自从linux2.6.8之后，size参数是被忽略的。

2\. int epoll\_ctl(int epfd, int op, int fd, struct epoll\_event \*event);
epoll的事件注册函数，它不同与select()是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。第一个参数是epoll\_create()的返回值，第二个参数表示动作，用三个宏来表示：
EPOLL\_CTL\_ADD：注册新的fd到epfd中；
EPOLL\_CTL\_MOD：修改已经注册的fd的监听事件；
EPOLL\_CTL\_DEL：从epfd中删除一个fd；
第三个参数是需要监听的fd，第四个参数是告诉内核需要监听什么事，struct epoll\_event结构如下：

typedef union epoll\_data {
    void \*ptr;
    int fd;
    \_\_uint32\_t u32;
    \_\_uint64\_t u64;
} epoll\_data\_t;

struct epoll\_event {
    \_\_uint32\_t events; /\* Epoll events \*/
    epoll\_data\_t data; /\* User data variable \*/
};

events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里。

```
事件值          枚举值           触发条件

EPOLLIN       0x1               新连接，断连接，可读
EPOLLPRI      0x2               有带外数据到来
EPOLLOUT      0x4               新连接，可写
EPOLLERR      0x8               写已关闭   socket pipe broken
EPOLLHUP      0x10              断连接，收到RST包。在注册事件的时候这个事件是默认添加
EPOLLET       0x80000000        边沿触发模式
EPOLLONESHOT  0x40000000        只监听一次事件模式
对端异常断开连接（拔网线）          没触发任何事件
```

epoll使用的资料网上一大把，EPOLLIN(读)监听事件的类型，大家一般使用起来一般没有什么疑问，无非是监听某个端口，一旦客户端连接有数据发送，它马上通知服务端有数据，一般用一个回调的读函数，从这个相关的socket接口读取数据就行了。但是有关EPOLLOUT(写)监听的使用，网上的资料却讲得不够明白，理解起来有点麻烦。因为监听一般都是被动操作，客户端有数据上来需要读写(被动的读操作，EPOLIN监听事件很好理解，但是服务器给客户发送数据是个主动的操作，写操作如何监听呢？
  如果将客户端的socket接口都设置成 EPOLLIN | EPOLLOUT(读，写)两个操作都设置，那么这个写操作会一直监听，有点影响效率。经过查阅大量资料，我终于明白了EPOLLOUT(写)监听的使用场，一般说明主要有以下三种使用场景:
  1： 对客户端socket只使用EPOLLIN(读)监听，不监听EPOLLOUT(写)，写操作一般使用socket的send操作
  2：客户端的socket初始化为EPOLLIN(读)监听，有数据需要发送时，对客户端的socket修改为EPOLLOUT(写)操作，这时EPOLL机制会回调发送数据的函数，发送完数据之后，再将客户端的socket修改为EPOLL(读)监听

 3：对客户端socket使用EPOLLIN 和 EPOLLOUT两种操作，这样每一轮epoll\_wait循环都会回调读，写函数，这种方式效率不是很好

3\. int epoll\_wait(int epfd, struct epoll\_event \* events, int maxevents, int timeout);
等待事件的产生，类似于select()调用。参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个 maxevents的值不能大于创建epoll\_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。
        基本的语法为：nfds = epoll\_wait(kdpfd, events, maxevents, -1);
其中:
       kdpfd  为用epoll\_create创建之后的句柄，
       events 是一个epoll\_event\*的指针，当epoll\_wait这个函数操作成功之后，epoll\_events里面将储存所有的读写事件。
       max\_events是当前需要监听的所有socket句柄数。这个 maxevents的值不能大于创建epoll\_create()时的size
        timeout是epoll\_wait的超时，
                      为0  , 表示马上返回，
                      为-1 , 表示一直等下去，直到有事件范围，
                      为任意正整数的时候表示等这么长的时间，如果一直没有事件，则返回。
                             一般如果网络主循环是单独的线程的话，可以用-1来等，这样可以保证一些效率(等效于阻塞)
                      如果是和主逻辑在同一个线程的话，则可以用0来保证主循环的效率。

4、关于ET、LT两种工作模式：
可以得出这样的结论:
ET模式仅当状态发生变化的时候才获得通知,这里所谓的状态的变化并不包括缓冲区中还有未处理的数据,也就是说,如果要采用ET模式,需要一直read/write直到出错为止,很多人反映为什么采用ET模式只接收了一部分数据就再也得不到通知了,大多因为这样;而LT模式是只要有数据没有处理就会一直通知下去的. 
LT(level triggered)是epoll**缺省的工作方式**，并且同时支持block和no-block socket.
ET (edge-triggered)是高速工作方式，只支持no-block socket，它效率要比LT更高.

Epoll的ET模式是边沿触发模式，其中有两个事件：EPOLLOUT和EPOLLIN事件。

#### **EPOLLOUT事件**

只有在连接是触发一次，表示该套接字进入可写状态。在下面的几种情况下也可以触发：
1、某次write写操作，写满了缓冲区，返回错误码为EAGAIN。
2、对端读取了一些数据，缓冲区又变为可写状态了，此时会触发EPOLLOUT事件。
3、也可以通过epoll\_ctl重新设置一下event，设置成与原来的触发状态一致即可，当然需要包含EPOLLOUT事件，设置后会立马触发一次EPOLLOUT事件。

#### **EPOLLIN事件**

只有当该套接字对端的数据可读时才会触发，触发一次后需要不断读取对端的数据直到返回EAGAIN错误码为止。
