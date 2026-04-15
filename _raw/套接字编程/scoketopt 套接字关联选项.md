---
source: http://blog.csdn.net/piaojun_pj/article/details/6098438
---
功能描述：
        获取或者设置与某个套接字关联的选 项。选项可能存在于多层协议中，它们总会出现在最上面的套接字层。当操作套接字选项时，选项位于的层和选项的名称必须给出。为了操作套接字层的选项，应该 将层的值指定为SOL\_SOCKET。为了操作其它层的选项，控制选项的合适协议号必须给出。例如，为了表示一个选项由TCP协议解析，层应该设定为协议 号TCP。

用法：
#include
#include
int getsockopt(int sock, int level, int optname, void \*optval, socklen\_t \*optlen);
int setsockopt(int sock, int level, int optname, const void \*optval, socklen\_t optlen);
参数：
sock：将要被设置或者获取选项的套接字。
level：选项所在的协议层。
optname：需要访问的选项名。
optval：对于getsockopt()，指向返回选项值的缓冲。对于setsockopt()，指向包含新选项值的缓冲。
optlen：对于getsockopt()，作为入口参数时，选项值的最大长度。作为出口参数时，选项值的实际长度。对于setsockopt()，现选项的长度。

返回说明：
成功执行时，返回0。失败返回-1，errno被设为以下的某个值
EBADF：sock不是有效的文件描述词
EFAULT：optval指向的内存并非有效的进程空间
EINVAL：在调用setsockopt()时，optlen无效
ENOPROTOOPT：指定的协议层不能识别选项
ENOTSOCK：sock描述的不是套接字

参数详细说明：
level指定控制套接字的层次.可以取三种值:
1)SOL\_SOCKET:通用套接字选项.
2)IPPROTO\_IP:IP选项.
3)IPPROTO\_TCP:TCP选项.
optname指定控制的方式(选项的名称),我们下面详细解释

```
optval获得或者是设置套接字选项.根据选项名称的数据类型进行转换

选项名称　　　　　　　　说明　　　　　　　　　　　　　　　　　　数据类型
========================================================================
　　　　　　　　　　　　SOL_SOCKET
------------------------------------------------------------------------
SO_BROADCAST　　　　　　允许发送广播数据　　　　　　　　　　　　int
SO_DEBUG　　　　　　　　允许调试　　　　　　　　　　　　　　　　int
SO_DONTROUTE　　　　　　不查找路由　　　　　　　　　　　　　　　int
SO_ERROR　　　　　　　　获得套接字错误　　　　　　　　　　　　　int
SO_KEEPALIVE　　　　　　保持连接　　　　　　　　　　　　　　　　int
SO_LINGER　　　　　　　 延迟关闭连接　　　　　　　　　　　　　　struct linger
SO_OOBINLINE　　　　　　带外数据放入正常数据流　　　　　　　　　int
SO_RCVBUF　　　　　　　 接收缓冲区大小　　　　　　　　　　　　　int
SO_SNDBUF　　　　　　　 发送缓冲区大小　　　　　　　　　　　　　int
SO_RCVLOWAT　　　　　　 接收缓冲区下限　　　　　　　　　　　　　int
SO_SNDLOWAT　　　　　　 发送缓冲区下限　　　　　　　　　　　　　int
SO_RCVTIMEO　　　　　　 接收超时　　　　　　　　　　　　　　　　struct timeval
SO_SNDTIMEO　　　　　　 发送超时　　　　　　　　　　　　　　　　struct timeval
SO_REUSERADDR　　　　　 允许重用本地地址和端口　　　　　　　　　int
SO_TYPE　　　　　　　　 获得套接字类型　　　　　　　　　　　　　int
SO_BSDCOMPAT　　　　　　与BSD系统兼容　　　　　　　　　　　　　 int
========================================================================
　　　　　　　　　　　　IPPROTO_IP
------------------------------------------------------------------------
IP_HDRINCL　　　　　　　在数据包中包含IP首部　　　　　　　　　　int
IP_OPTINOS　　　　　　　IP首部选项　　　　　　　　　　　　　　　int
IP_TOS　　　　　　　　　服务类型
IP_TTL　　　　　　　　　生存时间　　　　　　　　　　　　　　　　int
========================================================================
　　　　　　　　　　　　IPPRO_TCP
------------------------------------------------------------------------
TCP_MAXSEG　　　　　　　TCP最大数据段的大小　　　　　　　　　　 int
TCP_NODELAY　　　　　　 不使用Nagle算法　　　　　　　　　　　　 int
========================================================================
```

1.closesocket（一般不会立即关闭而经历TIME\_WAIT的过程）后想继续重用该socket：
BOOL bReuseaddr=TRUE;
setsockopt(s,SOL\_SOCKET ,SO\_REUSEADDR,(const char\*)&bReuseaddr,sizeof(BOOL));

2\. 如果要已经处于连接状态的soket在调用closesocket后强制关闭，不经历TIME\_WAIT的过程：
BOOL bDontLinger = FALSE;
setsockopt(s,SOL\_SOCKET,SO\_DONTLINGER,(const char\*)&bDontLinger,sizeof(BOOL));

3.在send(),recv()过程中有时由于网络状况等原因，发收不能预期进行,而设置收发时限：
int nNetTimeout=1000;//1秒
//发送时限
setsockopt(socket，SOL\_S0CKET,SO\_SNDTIMEO，(char \*)&nNetTimeout,sizeof(int));
//接收时限
setsockopt(socket，SOL\_S0CKET,SO\_RCVTIMEO，(char \*)&nNetTimeout,sizeof(int));

4.在send()的时候，返回的是实际发送出去的字节(同步)或发送到socket缓冲区的字节
(异步);系统默认的状态发送和接收一次为8688字节(约为8.5K)；在实际的过程中发送数据
和接收数据量比较大，可以设置socket缓冲区，而避免了send(),recv()不断的循环收发：
// 接收缓冲区
int nRecvBuf=32\*1024;//设置为32K
setsockopt(s,SOL\_SOCKET,SO\_RCVBUF,(const char\*)&nRecvBuf,sizeof(int));
//发送缓冲区
int nSendBuf=32\*1024;//设置为32K
setsockopt(s,SOL\_SOCKET,SO\_SNDBUF,(const char\*)&nSendBuf,sizeof(int));

5\. 如果在发送数据的时，希望不经历由系统缓冲区到socket缓冲区的拷贝而影响
程序的性能：
int nZero=0;
setsockopt(socket，SOL\_S0CKET,SO\_SNDBUF，(char \*)&nZero,sizeof(nZero));

6.同上在recv()完成上述功能(默认情况是将socket缓冲区的内容拷贝到系统缓冲区)：
int nZero=0;
setsockopt(socket，SOL\_S0CKET,SO\_RCVBUF，(char \*)&nZero,sizeof(int));

7.一般在发送UDP数据报的时候，希望该socket发送的数据具有广播特性：
BOOL bBroadcast=TRUE;
setsockopt(s,SOL\_SOCKET,SO\_BROADCAST,(const char\*)&bBroadcast,sizeof(BOOL));

8.在client连接服务器过程中，如果处于非阻塞模式下的socket在connect()的过程中可以设置connect()延时,直到accpet()被呼叫(本函数设置只有在非阻塞的过程中有显著的作用，在阻塞的函数调用中作用不大)
BOOL bConditionalAccept=TRUE;
setsockopt(s,SOL\_SOCKET,SO\_CONDITIONAL\_ACCEPT,(const char\*)&bConditionalAccept,sizeof(BOOL));

9.如果在发送数据的过程中(send()没有完成，还有数据没发送)而调用了closesocket(),以前我们一般采取的措施是"从容关闭"shutdown(s,SD\_BOTH),但是数据是肯定丢失了，如何设置让程序满足具体应用的要求(即让没发完的数据发送出去后在关闭socket)？
struct linger {
u\_short l\_onoff;
u\_short l\_linger;
};
linger m\_sLinger;
m\_sLinger.l\_onoff=1;//(在closesocket()调用,但是还有数据没发送完毕的时候容许逗留)
// 如果m\_sLinger.l\_onoff=0;则功能和2.)作用相同;
m\_sLinger.l\_linger=5;//(容许逗留的时间为5秒)
setsockopt(s,SOL\_SOCKET,SO\_LINGER,(const char\*)&m\_sLinger,sizeof(linger));
