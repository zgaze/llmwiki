---
source: http://blog.csdn.net/pbymw8iwm/article/details/7974789
---
功能描述：根据文件描述词来操作文件的特性。

#include <unistd.h>
#include <fcntl.h> 
int fcntl(int fd, int cmd); 
int fcntl(int fd, int cmd, long arg); 
int fcntl(int fd, int cmd, struct flock \*lock);

\[描述\]
fcntl()针对(文件)描述符提供控制。参数fd是被参数cmd操作(如下面的描述)的描述符。针对cmd的值，fcntl能够接受第三个参数int arg。

\[返回值\]
fcntl()的返回值与命令有关。如果出错，所有命令都返回－1，如果成功则返回某个其他值。下列三个命令有特定返回值：F\_DUPFD , F\_GETFD , F\_GETFL以及F\_GETOWN。
    F\_DUPFD   返回新的文件描述符
    F\_GETFD   返回相应标志
    F\_GETFL , F\_GETOWN   返回一个正的进程ID或负的进程组ID

fcntl函数有5种功能： 
1\. 复制一个现有的描述符(cmd=F\_DUPFD). 
2\. 获得／设置文件描述符标记(cmd=F\_GETFD或F\_SETFD). 
3\. 获得／设置文件状态标记(cmd=F\_GETFL或F\_SETFL). 
4\. 获得／设置异步I/O所有权(cmd=F\_GETOWN或F\_SETOWN). 
5\. 获得／设置记录锁(cmd=F\_GETLK , F\_SETLK或F\_SETLKW).

**1\. cmd值的F\_DUPFD ：** 
F\_DUPFD    返回一个如下描述的(文件)描述符：
        ·最小的大于或等于arg的一个可用的描述符
        ·与原始操作符一样的某对象的引用
        ·如果对象是文件(file)的话，则返回一个新的描述符，这个描述符与arg共享相同的偏移量(offset)
        ·相同的访问模式(读，写或读/写)
        ·相同的文件状态标志(如：两个文件描述符共享相同的状态标志)
        ·与新的文件描述符结合在一起的close-on-exec标志被设置成交叉式访问execve(2)的系统调用

实际上调用dup(oldfd)；
等效于
        fcntl(oldfd, F\_DUPFD, 0);

而调用dup2(oldfd, newfd)；
等效于
        close(oldfd)；
        fcntl(oldfd, F\_DUPFD, newfd)；

**2\. cmd值的F\_GETFD和F\_SETFD：**      
F\_GETFD    取得与文件描述符fd联合的close-on-exec标志，类似FD\_CLOEXEC。如果返回值和FD\_CLOEXEC进行与运算结果是0的话，文件保持交叉式访问exec()，否则如果通过exec运行的话，文件将被关闭(arg 被忽略)        F\_SETFD    设置close-on-exec标志，该标志以参数arg的FD\_CLOEXEC位决定，应当了解很多现存的涉及文件描述符标志的程序并不使用常数 FD\_CLOEXEC，而是将此标志设置为0(系统默认，在exec时不关闭)或1(在exec时关闭)    

在修改文件描述符标志或文件状态标志时必须谨慎，先要取得现在的标志值，然后按照希望修改它，最后设置新标志值。不能只是执行F\_SETFD或F\_SETFL命令，这样会关闭以前设置的标志位。 

**3. cmd值的F\_GETFL和F\_SETFL：**   
F\_GETFL    取得fd的文件状态标志，如同下面的描述一样(arg被忽略)，在说明open函数时，已说明
了文件状态标志。不幸的是，三个存取方式标志 (O\_RDONLY , O\_WRONLY , 以及O\_RDWR)并不各占1位。(这三种标志的值各是0 , 1和2，由于历史原因，这三种值互斥 — 一个文件只能有这三种值之一。) 因此首先必须用屏蔽字O\_ACCMODE相与取得存取方式位，然后将结果与这三种值相比较。       
F\_SETFL    设置给arg描述符状态标志，可以更改的几个标志是：O\_APPEND，O\_NONBLOCK，O\_SYNC 和 O\_ASYNC。而fcntl的文件状态标志总共有7个：O\_RDONLY , O\_WRONLY , O\_RDWR , O\_APPEND , O\_NONBLOCK , O\_SYNC和O\_ASYNC

可更改的几个标志如下面的描述：
    O\_NONBLOCK   非阻塞I/O，如果read(2)调用没有可读取的数据，或者如果write(2)操作将阻塞，则read或write调用将返回-1和EAGAIN错误
    O\_APPEND     强制每次写(write)操作都添加在文件大的末尾，相当于open(2)的O\_APPEND标志
    O\_DIRECT     最小化或去掉reading和writing的缓存影响。系统将企图避免缓存你的读或写的数据。如果不能够避免缓存，那么它将最小化已经被缓存了的数据造成的影响。如果这个标志用的不够好，将大大的降低性能
    O\_ASYNC      当I/O可用的时候，允许SIGIO信号发送到进程组
 在修改文件描述符标志或文件状态标志时必须谨慎，先要取得现在的标志值，然后按照希望修改它，最后设置新标志值。不能只是执行F\_SETFD或F\_SETFL命令，这样会关闭以前设置的标志位。

**4. cmd值的F\_GETOWN和F\_SETOWN：**   
F\_GETOWN   取得当前正在接收SIGIO或者SIGURG信号的进程id或进程组id，进程组id返回的是负值(arg被忽略)     
F\_SETOWN   设置将接收SIGIO和SIGURG信号的进程id或进程组id，进程组id通过提供负值的arg来说明(arg绝对值的一个进程组ID)，否则arg将被认为是进程id

 **5. cmd值的F\_GETLK, F\_SETLK或F\_SETLKW：** 获得／设置记录锁的功能，成功则返回0，若有错误则返回-1，错误原因存于errno。F\_GETLK    通过第三个参数arg(一个指向flock的结构体)取得第一个阻塞lock description指向的锁。取得的信息将覆盖传到fcntl()的flock结构的信息。如果没有发现能够阻止本次锁(flock)生成的锁，这个结构将不被改变，除非锁的类型被设置成F\_UNLCK    
F\_SETLK    按照指向结构体flock的指针的第三个参数arg所描述的锁的信息设置或者清除一个文件的segment锁。F\_SETLK被用来实现共享(或读)锁(F\_RDLCK)或独占(写)锁(F\_WRLCK)，同样可以去掉这两种锁(F\_UNLCK)。如果共享锁或独占锁不能被设置，fcntl()将立即返回EAGAIN     
F\_SETLKW   除了共享锁或独占锁被其他的锁阻塞这种情况外，这个命令和F\_SETLK是一样的。如果共享锁或独占锁被其他的锁阻塞，进程将等待直到这个请求能够完成。当fcntl()正在等待文件的某个区域的时候捕捉到一个信号，如果这个信号没有被指定SA\_RESTART, fcntl将被中断

当一个共享锁被set到一个文件的某段的时候，其他的进程可以set共享锁到这个段或这个段的一部分。共享锁阻止任何其他进程set独占锁到这段保护区域的任何部分。如果文件描述符没有以读的访问方式打开的话，共享锁的设置请求会失败。

独占锁阻止任何其他的进程在这段保护区域任何位置设置共享锁或独占锁。如果文件描述符不是以写的访问方式打开的话，独占锁的请求会失败。

结构体flock的指针：
struct flcok 
{ 
short int l\_type; /\* 锁定的状态\*/

//以下的三个参数用于分段对文件加锁，若对整个文件加锁，则：l\_whence=SEEK\_SET, l\_start=0, l\_len=0
short int l\_whence; /\*决定l\_start位置\*/ 
off\_t l\_start; /\*锁定区域的开头位置\*/ 
off\_t l\_len; /\*锁定区域的大小\*/

pid\_t l\_pid; /\*锁定动作的进程\*/ 
};

l\_type 有三种状态： 
F\_RDLCK   建立一个供读取用的锁定 
F\_WRLCK   建立一个供写入用的锁定 
F\_UNLCK   删除之前建立的锁定

l\_whence 也有三种方式： 
SEEK\_SET   以文件开头为锁定的起始位置 
SEEK\_CUR   以目前文件读写位置为锁定的起始位置 
SEEK\_END   以文件结尾为锁定的起始位置
