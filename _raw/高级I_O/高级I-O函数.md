---
---
~~[Edit](http://maxiang.info/#/?provider=evernote&guid=99a48e50-18d5-421b-be3e-b85be529c803&notebook=%E9%AB%98%E7%BA%A7I%2FO)~~

## 高级I/O函数

### 套接字超时限制

#### 调用alarm，使用SIGALRM

* 为connect设置超时，connect默认超时时间75s

* 为recvfrom设计超时时间

#### 使用select

#### 使用SO\_RCVTIMEO选项

### recv和send函数

函数原型：

    #include <sys/types.h>
    #include <sys/socket.h>
    ssize_t recv(int sockfd, void *buf, size_t len, int flags);
    ssize_t send(int sockfd, const void *buf, size_t len, int flags);
    

参数说明：

* 前三个参数和write、read一样
* flags为0，就跟write的作用一样，可以设置为一下：

|     |     |     |     |
| :--- | ---: | :---: | --- |
| flags | 说明  | recv | send |
| MSG\_DONTROUTE | 绕过路由表查找（无需查找） |     | y   |
| MSG\_DONTWAIT | 本次操作非阻塞 | y   | y   |
| MSG\_OOB | 发送或接受外带收据 | y   | y   |
| MSG\_PEEK | 窥看外来消息（读取后不丢弃用） | y   |     |
| MSG\_WAITALL | 等待所有数据（读到N才返回） | y   |     |

### readv和writev函数

函数原型：

    #include <sys/uio.h>
    ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
    ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
    

可以读入到或者写出自多个缓冲区，分散读、集中写；可以分别利用堆区和栈区接收、发送数据。

    struct msghdr {
        void       *msg_name;   /* optional address */
        socklen_t   msg_namelen;/* size of address */
        struct iovec *msg_iov;  /* scatter/gather array */
        size_t     msg_iovlen; /* # elements in msg_iov */
        void     *msg_control; /* ancillary data, see below */
        size_t    msg_controllen; /* ancillary data buffer len */
        int       msg_flags; /* flags (unused) */
    };
    #include <sys/types.h>
    #include <sys/socket.h>
    ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
    ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
    

最通用的两个I/O函数，其他的都可以使用这个代替。

#### 参数说明

* `msg_name` 和 `msg_namelen`适用于套接字未连接的场合（如未连接的UDP套接字），类似于`sendto`和`recvfrom`的第五个和第6个参数，或者说`accept`的最后两个参数。
* `msg_iov`和`msg_iov`指定输入或输出缓冲区数组；类似`readv`的第二个和第三个参数；
* msg\_flags 不使用了，最起码sendmsg不使用。

#### 辅助数据

辅助数据可以通过`msg_control`和`msg_controllen`两个成员发送和接收。

%23%23%20%u9AD8%u7EA7I/O%u51FD%u6570%0A@%28%u9AD8%u7EA7I/O%29%0A%0A%23%23%23%20%u5957%u63A5%u5B57%u8D85%u65F6%u9650%u5236%0A%23%23%23%23%20%u8C03%u7528alarm%uFF0C%u4F7F%u7528SIGALRM%0A-%20%u4E3Aconnect%u8BBE%u7F6E%u8D85%u65F6%uFF0Cconnect%u9ED8%u8BA4%u8D85%u65F6%u65F6%u95F475s%0A-%20%u4E3Arecvfrom%u8BBE%u8BA1%u8D85%u65F6%u65F6%u95F4%0A%0A%23%23%23%23%20%u4F7F%u7528select%0A%23%23%23%23%20%u4F7F%u7528SO\_RCVTIMEO%u9009%u9879%0A%0A%23%23%23%20recv%u548Csend%u51FD%u6570%0A%u51FD%u6570%u539F%u578B%uFF1A%0A%60%60%60%0A%23include%20%3Csys/types.h%3E%0A%23include%20%3Csys/socket.h%3E%0Assize\_t%20recv%28int%20sockfd%2C%20void%20\*buf%2C%20size\_t%20len%2C%20int%20flags%29%3B%0Assize\_t%20send%28int%20sockfd%2C%20const%20void%20\*buf%2C%20size\_t%20len%2C%20int%20flags%29%3B%0A%60%60%60%0A%u53C2%u6570%u8BF4%u660E%uFF1A%0A-%20%u524D%u4E09%u4E2A%u53C2%u6570%u548Cwrite%u3001read%u4E00%u6837%0A-%20flags%u4E3A0%uFF0C%u5C31%u8DDFwrite%u7684%u4F5C%u7528%u4E00%u6837%uFF0C%u53EF%u4EE5%u8BBE%u7F6E%u4E3A%u4E00%u4E0B%uFF1A%0A%0A%7C%20flags%20%20%20%20%20%20%7C%20%20%20%20%20%u8BF4%u660E%7C%20%20recv%20%20%20%7C%20%20send%20%20%7C%0A%7C%20%3A--------%20%7C%20--------%3A%7C%20%3A------%3A%20%7C%20%20%20%20%20%20%20%7C%0A%7C%20MSG\_DONTROUTE%20%20%7C%20%20%20%u7ED5%u8FC7%u8DEF%u7531%u8868%u67E5%u627E%uFF08%u65E0%u9700%u67E5%u627E%uFF09%20%7C%20%20%20%20%7C%20%20%20y%20%20%20%20%7C%0A%7CMSG\_DONTWAIT%20%20%20%20%7C%u672C%u6B21%u64CD%u4F5C%u975E%u963B%u585E%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7Cy%7Cy%7C%0A%7CMSG\_OOB%20%20%20%20%20%20%20%20%20%7C%u53D1%u9001%u6216%u63A5%u53D7%u5916%u5E26%u6536%u636E%20%20%20%20%20%20%20%20%20%20%7Cy%7Cy%7C%0A%7CMSG\_PEEK%20%20%20%20%20%20%20%20%7C%u7AA5%u770B%u5916%u6765%u6D88%u606F%uFF08%u8BFB%u53D6%u540E%u4E0D%u4E22%u5F03%u7528%uFF09%7Cy%7C%7C%0A%7CMSG\_WAITALL%20%20%20%20%20%7C%u7B49%u5F85%u6240%u6709%u6570%u636E%uFF08%u8BFB%u5230N%u624D%u8FD4%u56DE%uFF09%20%7C%20%20%20y%20%20%20%7C%0A%0A%23%23%23%20readv%u548Cwritev%u51FD%u6570%0A%u51FD%u6570%u539F%u578B%uFF1A%0A%60%60%60%0A%23include%20%3Csys/uio.h%3E%0Assize\_t%20readv%28int%20fd%2C%20const%20struct%20iovec%20\*iov%2C%20int%20iovcnt%29%3B%0Assize\_t%20writev%28int%20fd%2C%20const%20struct%20iovec%20\*iov%2C%20int%20iovcnt%29%3B%0A%60%60%60%0A%u53EF%u4EE5%u8BFB%u5165%u5230%u6216%u8005%u5199%u51FA%u81EA%u591A%u4E2A%u7F13%u51B2%u533A%uFF0C%u5206%u6563%u8BFB%u3001%u96C6%u4E2D%u5199%uFF1B%u53EF%u4EE5%u5206%u522B%u5229%u7528%u5806%u533A%u548C%u6808%u533A%u63A5%u6536%u3001%u53D1%u9001%u6570%u636E%u3002%0A%0A%23%23%23%20%0A%60%60%60%0Astruct%20msghdr%20%7B%0A%20%20%20%20void%20%20%20%20%20%20%20\*msg\_name%3B%20%20%20/\*%20optional%20address%20\*/%0A%20%20%20%20socklen\_t%20%20%20msg\_namelen%3B/\*%20size%20of%20address%20\*/%0A%20%20%20%20struct%20iovec%20\*msg\_iov%3B%20%20/\*%20scatter/gather%20array%20\*/%0A%20%20%20%20size\_t%20%20%20%20%20msg\_iovlen%3B%20/\*%20%23%20elements%20in%20msg\_iov%20\*/%0A%20%20%20%20void%20%20%20%20%20\*msg\_control%3B%20/\*%20ancillary%20data%2C%20see%20below%20\*/%0A%20%20%20%20size\_t%20%20%20%20msg\_controllen%3B%20/\*%20ancillary%20data%20buffer%20len%20\*/%0A%20%20%20%20int%20%20%20%20%20%20%20msg\_flags%3B%20/\*%20flags%20%28unused%29%20\*/%0A%7D%3B%0A%23include%20%3Csys/types.h%3E%0A%23include%20%3Csys/socket.h%3E%0Assize\_t%20recvmsg%28int%20sockfd%2C%20struct%20msghdr%20\*msg%2C%20int%20flags%29%3B%0Assize\_t%20sendmsg%28int%20sockfd%2C%20const%20struct%20msghdr%20\*msg%2C%20int%20flags%29%3B%0A%60%60%60%0A%u6700%u901A%u7528%u7684%u4E24%u4E2AI/O%u51FD%u6570%uFF0C%u5176%u4ED6%u7684%u90FD%u53EF%u4EE5%u4F7F%u7528%u8FD9%u4E2A%u4EE3%u66FF%u3002%0A%23%23%23%23%20%u53C2%u6570%u8BF4%u660E%0A-%20%60msg\_name%60%20%u548C%20%60msg\_namelen%60%u9002%u7528%u4E8E%u5957%u63A5%u5B57%u672A%u8FDE%u63A5%u7684%u573A%u5408%uFF08%u5982%u672A%u8FDE%u63A5%u7684UDP%u5957%u63A5%u5B57%uFF09%uFF0C%u7C7B%u4F3C%u4E8E%60sendto%60%u548C%60recvfrom%60%u7684%u7B2C%u4E94%u4E2A%u548C%u7B2C6%u4E2A%u53C2%u6570%uFF0C%u6216%u8005%u8BF4%60accept%60%u7684%u6700%u540E%u4E24%u4E2A%u53C2%u6570%u3002%0A-%20%60msg\_iov%60%u548C%60msg\_iov%60%u6307%u5B9A%u8F93%u5165%u6216%u8F93%u51FA%u7F13%u51B2%u533A%u6570%u7EC4%uFF1B%u7C7B%u4F3C%60readv%60%u7684%u7B2C%u4E8C%u4E2A%u548C%u7B2C%u4E09%u4E2A%u53C2%u6570%uFF1B%0A-%20msg\_flags%20%u4E0D%u4F7F%u7528%u4E86%uFF0C%u6700%u8D77%u7801sendmsg%u4E0D%u4F7F%u7528%u3002%0A%0A%23%23%23%23%20%u8F85%u52A9%u6570%u636E%0A%u8F85%u52A9%u6570%u636E%u53EF%u4EE5%u901A%u8FC7%60msg\_control%60%u548C%60msg\_controllen%60%u4E24%u4E2A%u6210%u5458%u53D1%u9001%u548C%u63A5%u6536%u3002%0A%0A%0A
