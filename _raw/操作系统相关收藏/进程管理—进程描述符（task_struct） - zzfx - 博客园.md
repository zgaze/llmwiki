---
source: https://www.cnblogs.com/feng9exe/p/6883614.html
---
http://blog.csdn.net/qq\_26768741/article/details/54348586

当把一个程序加载到内存当中，此时，这个时候就有了进程，关于进程，有一个相关的叫做进程控制块（PCB），这个是系统为了方便进行管理进程所设置的一个数据结构，通过PCB，就可以记录进程的特征以及一些信息。 
内核当中使用进程描述符task\_struct。 
这个task\_struct就是一个定义的一个结构体，通过这个结构体，可以对进程的所有的相关的信息进行维护，对进程进行管理。

接下来我们需要对task\_struct结构体当中的成员进行一些分析。

|     |
| --- |
| linux内核版本 |
| Linux version 2.6.32-431.el6.i686 |

# **1 task\_struct**

* * *

## **1.1 进程状态**

* * *

volatile long state;
int exit\_state;\`

表示进程的状态， 在进程执行的时候，它会有一个状态，这个状态对于进程来说是很重要的一个属性。进程主要有以下几个状态。

state可能的取值 
![[./_resources/进程管理—进程描述符（task_struct）_-_zzfx_-_博客园.resources/SouthEast.png]]

    这些状态就不再一一说明了，后续进程篇会有专门的说明。

## **1.2 进程标识符（PID）**

* * *

pid\_t pid;
pid\_t tgid;

每个进程都有进程标识符、用户标识符、组标识符，进程标识符对于每一个进程来说都是唯一的。内核通过进程标识符来对不同的进程进行识别，一般来说，行创建的进程都是在前一个进程的基础上PID加上1作为本进程的PID。为了linux平台兼容性，PID一般最大为32767。

## **1.3 进程内核栈**

* * *

void \*stack

stack用来维护分配给进程的内核栈，内核栈的意义在于，进程task\_struct所占的内存是由内核动态分配的，确切的说就是内核根本不给task\_struct分配内存，只给内核栈分配8KB内存，并且一部分会提供给task\_struct使用。 
task\_struct结构体大约占用的大小为1K左右，根据内核版本的不同，大小也会有差异。 
所以，也就可以知道内核栈最大也就是7KB，否则，内核栈会覆盖task\_struct结构。

## **1.4 标记**

* * *

unsigned int flags
用来反映一个进程的状态信息，但不是运行状态，用于内核识别进程当前的状态，flags的取值如下：

|     |     |
| --- | --- |
| 可使用的标记 | 功能  |
| PF\_FORKNOEXEC | 进程刚创建，但还没执行。 |
| PF\_SUPERPRIV | 超级用户特权。 |
| PF\_DUMPCORE | 关于核心。 |
| PF\_SIGNALED | 进程被信号(signal)杀出。 |
| PF\_EXITING | 进程开始关闭。 |

## **1.5 表示进程亲属关系的成员**

* * *

`struct task_struct *real_parent;`
`struct task_struct *parent;`
`struct list_head children;`
`struct list_head sibling;`
`struct task_struct *group_leader;`

linux系统当中，考虑到进程的派生，所以进程之间会存在父进程和子进程这样的关系，当然，对于同一个父进程派生出来的进程，他们的关系当然是兄弟进程了。

|     |     |
| --- | --- |
| 成员  | 功能  |
| real\_parent | 指向父进程的指针，如果父进程不存在了，则指向PID为1的进程 |
| parent | 指向父进程的，值与real——parent相同，需要向它的父进程发送信号 |
| children | 表示链表的头部，链表中的所有元素都是它的子进程 |
| sibling | 用于当前进程插入兄弟链表当中 |
| group\_leader | 指向进程组的领头进程 |

## **1.6 ptrace系统调用**

* * *

unsigned int ptrace;
`struct list_head ptraced;`
`struct list_head ptrace_entry;`

首先我们要清楚ptrace是什么东西，ptrace是一种提供父进程控制子进程运行，并且可以检查和改变它的核心image。当trace设置为0时不需要被跟踪。

## **1.7 性能诊断工具——Performance Event**

* * *

`#ifdef CONFIG_PERF_EVENTS`
`#ifndef __GENKSYMS__`
void \* \_\_reserved\_perf\_\_;
`#else`
struct perf\_event\_context \*perf\_event\_ctxp;
`#endif`
struct mutex perf\_event\_mutex;
struct list\_head perf\_event\_list;
`#endif`

Performance Event是性能诊断工具，这些成员用来帮助它进行分析进程性能问题。

## **1.8 进程调度**

* * *

int prio, static\_prio, normal\_prio;
unsigned int rt\_priority;

|     |     |
| --- | --- |
| 成员  | 功能  |
| static\_prio | 保存静态优先级，可以通过nice系统进行修改 |
| rt\_priority | 保存实时优先级 |
| normal\_prio | 保存静态优先级和调度策略 |
| prio | 保存动态优先级 |

调度进程利用这部分信息决定系统当中的那个进程最应该运行，并且结合进程的状态信息保证系统运作高效。
提到进程调度，当然还需要说明一下进程调度策略，我们来看下关于调度策略的成员：

unsigned int policy;
const struct sched\_class \*sched\_class;
struct sched\_entity se;
struct sched\_rt\_entity rt;

|     |     |
| --- | --- |
| 成员  | 功能  |
| policy | 调度策略 |
| sched\_class | 调度类 |
| se  | 普通进程的一个调用的实体，每一个进程都有其中之一的实体 |
| rt  | 实时进程的调用实体，每个进程都有其中之一的实体 |
| cpus\_allowed | 用于控制进程可以在处理器的哪里运行 |

policy表示进程的调度策略，主要有以下五种：

|     |     |
| --- | --- |
| 种类  | 功能  |
| SCHED\_NORMAL | 用于普通进程 |
| SCHED\_BATCH | 普通进程策略的分化版本，采用分时策略 |
| SCHED\_IDLE | 优先级最低，系统空闲时才跑这类进程 |
| SCHED\_FIFO | 先入先出的调度算法 |
| SCHED\_RR | 实时调度算法，采用时间片，相同优先级的任务当用完时间片就会放到队列的尾部，保证公平性，同时，高优先级的任务抢占低优先级的任务。 |
| SCHED\_DEADLINE | 新支持的实时调度策略，正对突发性计算 |

说完了调度策略，我们再来看一下调度类。

|     |     |
| --- | --- |
| 调度类 | 功能  |
| idle\_sched\_class | 每一个cpu的第一个pid=0的线程，是一个静态的线程 |
| stop\_sched\_class | 优先级最高的线程，会中断所有其他的线程，而且不会被其他任务打断 |
| rt\_sched\_slass | 作用在实时线程 |
| fair\_sched\_class | 作用的一般线程 |

它们的优先级顺序为Stop>rt>fair>idle

## **1.9进程的地址空间**

* * *

struct mm\_struct \*mm, \*active\_mm;

|     |     |
| --- | --- |
| 成员  | 功能  |
| mm  | 进程所拥有的用户空间的内存描述符 |
| active\_mm | 指向进程运行时使用的内存描述符，对于普通的进程来说，mm和active\_mm是一样的，但是内核线程是没有进程地址空间的，所以内核线程的mm是空的，所以需要初始化内核线程的active\_mm |

对于内核线程切记是没有地址空间的。
后续会有专门的博客来叙述

## **1.10 判断标志**

* * *

//用于进程判断标志
int exit\_state;
int exit\_code, exit\_signal;
int pdeath\_signal; /\* The signal sent when the parent dies \*/
/\* ??? \*/
unsigned int personality;
unsigned did\_exec:1;
unsigned in\_execve:1; /\* Tell the LSMs that the process is doing an
`* execve */`
unsigned in\_iowait:1;

/\* Revert to default priority/policy when forking \*/
unsigned sched\_reset\_on\_fork:1;

|     |     |
| --- | --- |
| 成员  | 功能  |
| exit\_state | 进程终止的状态 |
| exit\_code | 设置进程的终止代号 |
| exit\_signal | 设置为-1的时候表示是某个线程组当中的一员，只有当线程组的最后一个成员终止时，才会产生型号给父进程 |
| pdeath\_signal | 用来判断父进程终止时的信号 |

## **1.11 时间与定时器**

* * *

关于时间，一个进程从创建到终止叫做该进程的生存期，进程在其生存期内使用CPU时间，内核都需要进行记录，进程耗费的时间分为两部分，一部分是用户模式下耗费的时间，一部分是在系统模式下耗费的时间。

//描述CPU时间的内容
cputime\_t utime, stime, utimescaled, stimescaled;
cputime\_t gtime;
cputime\_t prev\_utime, prev\_stime;
unsigned long nvcsw, nivcsw; /\* context switch counts \*/
struct timespec start\_time; /\* monotonic time \*/
struct timespec real\_start\_time; /\* boot based time \*/
struct task\_cputime cputime\_expires;
struct list\_head cpu\_timers\[3\];

|     |     |
| --- | --- |
| 成员  | 属性  |
| utime/stime | 用于记录进程在用户状态/内核态下所经过的定时器 |
| prev\_utime/prev\_stime | 记录当前的运行时间 |
| utimescaled/stimescaled | 分别记录进程在用户态和内核态的运行的时间 |
| gtime | 记录虚拟机的运行时间 |
| nvcsw/nicsw | 是自愿/非自愿上下文切换计数 |
| start\_time/real\_start\_time | 进程创建时间，real还包括了进程睡眠时间 |
| cputime\_expires | 用来统计进程或进程组被跟踪的处理器时间，三个成员对应的是下面的cpu\_times\[3\]的三个链表 |

然后接下来我们来看一下进程的定时器，一共是三种定时器。

|     |     |     |
| --- | --- | --- |
| 定时器类型 | 解释  | 更新时刻 |
| ITIMER\_REAL | 实时定时器 | 实时更新，不在乎进程是否运行 |
| ITIMER\_VIRTUAL | 虚拟定时器 | 只在进程运行用户态时更新 |
| ITIMER\_PROF | 概况定时器 | 进程运行于用户态和系统态进行更新 |

进程总过有三种定时器，这三种定时器的特征有到期时间，定时间隔，和要触发的时间，

## **1.12 信号处理**

* * *

struct signal\_struct \*signal;
struct sighand\_struct \*sighand;

sigset\_t blocked, real\_blocked;
sigset\_t saved\_sigmask; /\* restored if set\_restore\_sigmask() was used \*/
struct sigpending pending;

unsigned long sas\_ss\_sp;
size\_t sas\_ss\_size;

关于信号处理：

|     |     |
| --- | --- |
| 成员  | 功能  |
| signal | 指向进程信号描述符 |
| sighand | 指向进程信号处理程序描述符 |
| blocked | 表示被阻塞信号的掩码 |
| pending | 存放私有挂起信号的数据结构 |
| sas\_ss\_sp | 信号处理程序备用堆栈的地址 |

## **1.13 文件系统信息**

* * *

//文件系统信息结构体
`/* filesystem information */`
struct fs\_struct \*fs;

//打开文件相关信息结构体
`/* open file information */`
struct files\_struct \*files;

进程可以用来打开和关闭文件，文件属于系统资源，task\_struct有两个来描述文件资源，他们会描述两个VFS索引节点，两个节点分别是root和pwd，分别指向根目录和当前的工作目录。

|     |     |
| --- | --- |
| 成员  | 功能  |
| struct fs\_struct \*fs | 进程可执行镜像所在的文件系统 |
| struct files\_struct \*files | 进程当前打开的文件 |

## **1.14 其他**

* * *

`struct task_struct {`
//进程状态（-1就绪态，0运行态，>0停止态）
volatile long state; /\* -1 unrunnable, 0 runnable, >0 stopped \*/

//进程内核栈
void \*stack;

//有几个进程只在使用此结构
atomic\_t usage;

//标记
unsigned int flags; /\* per process flags, defined below \*/

//ptrace系统调用，关于实现断点调试，跟踪进程运行。
unsigned int ptrace;

//锁的深度
int lock\_depth; /\* BKL lock depth \*/

//SMP实现无加锁的进程切换
`#ifdef CONFIG_SMP`
`#ifdef __ARCH_WANT_UNLOCKED_CTXSW`
int oncpu;
`#endif`
`#endif`

//关于进程调度
int prio, static\_prio, normal\_prio;

//优先级
unsigned int rt\_priority;

//关于进程
const struct sched\_class \*sched\_class;
struct sched\_entity se;
struct sched\_rt\_entity rt;

//preempt\_notifier结构体链表
`#ifdef CONFIG_PREEMPT_NOTIFIERS`
/\* list of struct preempt\_notifier: \*/
struct hlist\_head preempt\_notifiers;
`#endif`

/\*
`* fpu_counter contains the number of consecutive context switches`
`* that the FPU is used. If this is over a threshold, the lazy fpu`
`* saving becomes unlazy to save the trap. This is an unsigned char`
`* so that after 256 times the counter wraps and the behavior turns`
`* lazy again; this to deal with bursty apps that only use FPU for`
`* a short time`
`*/`

//FPU使用计数
unsigned char fpu\_counter;

//块设备I/O层的跟踪工具
`#ifdef CONFIG_BLK_DEV_IO_TRACE`
unsigned int btrace\_seq;
`#endif`
//进程调度策略相关的字段
unsigned int policy;

cpumask\_t cpus\_allowed;

//RCU同步原语
`#ifdef CONFIG_TREE_PREEMPT_RCU`
int rcu\_read\_lock\_nesting;
char rcu\_read\_unlock\_special;
struct rcu\_node \*rcu\_blocked\_node;
struct list\_head rcu\_node\_entry;
`#endif /* #ifdef CONFIG_TREE_PREEMPT_RCU */`

`//用于调度器统计进程运行信息`
`#if defined(CONFIG_SCHEDSTATS) || defined(CONFIG_TASK_DELAY_ACCT)`
struct sched\_info sched\_info;
`#endif`

`//用于构架进程链表`
struct list\_head tasks;
struct plist\_node pushable\_tasks;

//关于进程的地址空间，指向进程的地址空间。（链表和红黑树）
struct mm\_struct \*mm, \*active\_mm;

`/* task state */`
//进程状态参数
int exit\_state;

//退出信号处理
int exit\_code, exit\_signal;

//接收父进程终止的时候会发送信号
int pdeath\_signal; /\* The signal sent when the parent dies \*/
/\* ??? \*/
unsigned int personality;
unsigned did\_exec:1;
unsigned in\_execve:1; /\* Tell the LSMs that the process is doing an
`* execve */`
unsigned in\_iowait:1;

/\* Revert to default priority/policy when forking \*/
unsigned sched\_reset\_on\_fork:1;

//进程pid，父进程ppid。
pid\_t pid;
pid\_t tgid;

//防止内核堆栈溢出
`#ifdef CONFIG_CC_STACKPROTECTOR`
/\* Canary value for the -fstack-protector gcc feature \*/
unsigned long stack\_canary;
`#endif`

/\*
`* pointers to (original) parent process, youngest child, younger sibling,`
`* older sibling, respectively. (p->father can be replaced with`
`* p->real_parent->pid)`
`*/`

//这部分是用来进行维护进程之间的亲属关系的。
//初始化父进程
struct task\_struct \*real\_parent; /\* real parent process \*/
//接纳终止的进程
struct task\_struct \*parent; /\* recipient of SIGCHLD, wait4() reports \*/
/\*
`* children/sibling forms the list of my natural children`
`*/`
//维护子进程链表
struct list\_head children; /\* list of my children \*/
//兄弟进程链表
struct list\_head sibling; /\* linkage in my parent's children list \*/
//线程组组长
struct task\_struct \*group\_leader; /\* threadgroup leader \*/

/\*
`* ptraced is the list of tasks this task is using ptrace on.`
`* This includes both natural children and PTRACE_ATTACH targets.`
`* p->ptrace_entry is p's link on the p->parent->ptraced list.`
`*/`

//ptrace，系统调用，关于断点调试。
struct list\_head ptraced;
struct list\_head ptrace\_entry;

//PID与PID散列表的联系
/\* PID/PID hash table linkage. \*/
struct pid\_link pids\[PIDTYPE\_MAX\];

//维护一个链表，里面有该进程所有的线程
struct list\_head thread\_group;

//do\_fork()函数
struct completion \*vfork\_done; /\* for vfork() \*/
int \_\_user \*set\_child\_tid; /\* CLONE\_CHILD\_SETTID \*/
int \_\_user \*clear\_child\_tid; /\* CLONE\_CHILD\_CLEARTID \*/

//描述CPU时间的内容
//utime是用户态下的执行时间
//stime是内核态下的执行时间
cputime\_t utime, stime, utimescaled, stimescaled;
cputime\_t gtime;
cputime\_t prev\_utime, prev\_stime;

//上下文切换计数
unsigned long nvcsw, nivcsw; /\* context switch counts \*/
struct timespec start\_time; /\* monotonic time \*/
struct timespec real\_start\_time; /\* boot based time \*/
`/* mm fault and swap info: this can arguably be seen as either mm-specific or thread-specific */`

//缺页统计
unsigned long min\_flt, maj\_flt;

struct task\_cputime cputime\_expires;
struct list\_head cpu\_timers\[3\];

`/* process credentials */`

`//进程身份凭据`
const struct cred \*real\_cred; /\* objective and real subjective task
`* credentials (COW) */`
const struct cred \*cred; /\* effective (overridable) subjective task
`* credentials (COW) */`
struct mutex cred\_guard\_mutex; /\* guard against foreign influences on
`* credential calculations`
`* (notably. ptrace) */`
struct cred \*replacement\_session\_keyring; /\* for KEYCTL\_SESSION\_TO\_PARENT \*/

//去除路径以后的可执行文件名称，进程名
char comm\[TASK\_COMM\_LEN\]; /\* executable name excluding path
`- access with [gs]et_task_comm (which lock`
`it with task_lock())`
`- initialized normally by setup_new_exec */`
`/* file system info */`

//文件系统信息
int link\_count, total\_link\_count;
`#ifdef CONFIG_SYSVIPC`
`/* ipc stuff */`
`//进程通信`
struct sysv\_sem sysvsem;
`#endif`
`#ifdef CONFIG_DETECT_HUNG_TASK`
`/* hung task detection */`
unsigned long last\_switch\_count;
`#endif`

`//该进程在特点CPU下的状态`
`/* CPU-specific state of this task */`
struct thread\_struct thread;

//文件系统信息结构体
`/* filesystem information */`
struct fs\_struct \*fs;

//打开文件相关信息结构体
`/* open file information */`
struct files\_struct \*files;
`/* namespaces */`
`//命名空间：`
struct nsproxy \*nsproxy;
`/* signal handlers */`

//关于进行信号处理
struct signal\_struct \*signal;
struct sighand\_struct \*sighand;

sigset\_t blocked, real\_blocked;
sigset\_t saved\_sigmask; /\* restored if set\_restore\_sigmask() was used \*/
struct sigpending pending;

unsigned long sas\_ss\_sp;
size\_t sas\_ss\_size;
int (\*notifier)(void \*priv);
void \*notifier\_data;
sigset\_t \*notifier\_mask;

//进程审计
struct audit\_context \*audit\_context;
`#ifdef CONFIG_AUDITSYSCALL`
uid\_t loginuid;
unsigned int sessionid;
`#endif`
seccomp\_t seccomp;

`#ifdef CONFIG_UTRACE`
struct utrace \*utrace;
unsigned long utrace\_flags;
`#endif`

`//线程跟踪组`
`/* Thread group tracking */`
u32 parent\_exec\_id;
u32 self\_exec\_id;
`/* Protection of (de-)allocation: mm, files, fs, tty, keyrings, mems_allowed,`
`* mempolicy */`
spinlock\_t alloc\_lock;

//中断
`#ifdef CONFIG_GENERIC_HARDIRQS`
/\* IRQ handler threads \*/
struct irqaction \*irqaction;
`#endif`

`//task_rq_lock函数所使用的锁`
/\* Protection of the PI data structures: \*/
spinlock\_t pi\_lock;

//基于PI协议的等待互斥锁
`#ifdef CONFIG_RT_MUTEXES`
/\* PI waiters blocked on a rt\_mutex held by this task \*/
struct plist\_head pi\_waiters;
/\* Deadlock detection and priority inheritance handling \*/
struct rt\_mutex\_waiter \*pi\_blocked\_on;
`#endif`

`//死锁检测`
`#ifdef CONFIG_DEBUG_MUTEXES`
/\* mutex deadlock detection \*/
struct mutex\_waiter \*blocked\_on;
`#endif`

`//中断`
`#ifdef CONFIG_TRACE_IRQFLAGS`
unsigned int irq\_events;
int hardirqs\_enabled;
unsigned long hardirq\_enable\_ip;
unsigned int hardirq\_enable\_event;
unsigned long hardirq\_disable\_ip;
unsigned int hardirq\_disable\_event;
int softirqs\_enabled;
unsigned long softirq\_disable\_ip;
unsigned int softirq\_disable\_event;
unsigned long softirq\_enable\_ip;
unsigned int softirq\_enable\_event;
int hardirq\_context;
int softirq\_context;
`#endif`

`//lockdep`
`#ifdef CONFIG_LOCKDEP`
`# define MAX_LOCK_DEPTH 48UL`
u64 curr\_chain\_key;
int lockdep\_depth;
unsigned int lockdep\_recursion;
struct held\_lock held\_locks\[MAX\_LOCK\_DEPTH\];
gfp\_t lockdep\_reclaim\_gfp;
`#endif`

`//日志文件`
`/* journalling filesystem info */`

void \*journal\_info;

`/* stacked block device info */`
//块设备链表
struct bio \*bio\_list, \*\*bio\_tail;

`/* VM state */`
//虚拟内存状态，内存回收
struct reclaim\_state \*reclaim\_state;

//存放块设备I/O流量信息
struct backing\_dev\_info \*backing\_dev\_info;

//I/O调度器所用信息
struct io\_context \*io\_context;

unsigned long ptrace\_message;
siginfo\_t \*last\_siginfo; /\* For ptrace use. \*/

//记录进程I/O计数
struct task\_io\_accounting ioac;
`#if defined(CONFIG_TASK_XACCT)`
u64 acct\_rss\_mem1; /\* accumulated rss usage \*/
u64 acct\_vm\_mem1; /\* accumulated virtual memory usage \*/
cputime\_t acct\_timexpd; /\* stime + utime since last update \*/
`#endif`

//CPUSET功能
`#ifdef CONFIG_CPUSETS`
nodemask\_t mems\_allowed; /\* Protected by alloc\_lock \*/
`#ifndef __GENKSYMS__`
/\*
`* This does not change the size of the struct_task(2+2+4=4+4)`
`* so the offsets of the remaining fields are unchanged and`
`* therefore the kABI is preserved. Only the kernel uses`
`* cpuset_mem_spread_rotor and cpuset_slab_spread_rotor so`
`* it is safe to change it to use shorts instead of ints.`
\*/
unsigned short cpuset\_mem\_spread\_rotor;
unsigned short cpuset\_slab\_spread\_rotor;
int mems\_allowed\_change\_disable;
`#else`
int cpuset\_mem\_spread\_rotor;
int cpuset\_slab\_spread\_rotor;
`#endif`
`#endif`

`//Control Groups`
`#ifdef CONFIG_CGROUPS`
/\* Control Group info protected by css\_set\_lock \*/
struct css\_set \*cgroups;
/\* cg\_list protected by css\_set\_lock and tsk->alloc\_lock \*/
struct list\_head cg\_list;
`#endif`

`//futex同步机制`
`#ifdef CONFIG_FUTEX`
struct robust\_list\_head \_\_user \*robust\_list;
`#ifdef CONFIG_COMPAT`
struct compat\_robust\_list\_head \_\_user \*compat\_robust\_list;
`#endif`
struct list\_head pi\_state\_list;
struct futex\_pi\_state \*pi\_state\_cache;
`#endif`

`//关于内存检测工具Performance Event`
`#ifdef CONFIG_PERF_EVENTS`
`#ifndef __GENKSYMS__`
void \* \_\_reserved\_perf\_\_;
`#else`
struct perf\_event\_context \*perf\_event\_ctxp;
`#endif`
struct mutex perf\_event\_mutex;
struct list\_head perf\_event\_list;
`#endif`

//非一致内存访问
`#ifdef CONFIG_NUMA`
struct mempolicy \*mempolicy; /\* Protected by alloc\_lock \*/
short il\_next;
`#endif`

//文件系统互斥资源
atomic\_t fs\_excl; /\* holding fs exclusive resources \*/

//RCU链表
struct rcu\_head rcu;

/\*
`* cache last used pipe for splice`
`*/`

//管道
struct pipe\_inode\_info \*splice\_pipe;

//延迟计数
`#ifdef CONFIG_TASK_DELAY_ACCT`
struct task\_delay\_info \*delays;
`#endif`
`#ifdef CONFIG_FAULT_INJECTION`
int make\_it\_fail;
`#endif`
struct prop\_local\_single dirties;
`#ifdef CONFIG_LATENCYTOP`
int latency\_record\_count;
struct latency\_record latency\_record\[LT\_SAVECOUNT\];
`#endif`
/\*
`* time slack values; these are used to round up poll() and`
`* select() etc timeout values. These are in nanoseconds.`
`*/`

//time slack values,常用于poll和select函数
unsigned long timer\_slack\_ns;
unsigned long default\_timer\_slack\_ns;

//socket控制消息
struct list\_head \*scm\_work\_list;
`#ifdef CONFIG_FUNCTION_GRAPH_TRACER`

//ftrace跟踪器
/\* Index of current stored adress in ret\_stack \*/
int curr\_ret\_stack;
/\* Stack of return addresses for return function tracing \*/
struct ftrace\_ret\_stack \*ret\_stack;
/\* time stamp for last schedule \*/
unsigned long long ftrace\_timestamp;
/\*
`* Number of functions that haven't been traced`
`* because of depth overrun.`
`*/`
atomic\_t trace\_overrun;
/\* Pause for the tracing \*/
atomic\_t tracing\_graph\_pause;
`#endif`
`#ifdef CONFIG_TRACING`
/\* state flags for use by tracers \*/
unsigned long trace;
/\* bitmask of trace recursion \*/
unsigned long trace\_recursion;
`#endif /* CONFIG_TRACING */`
/\* reserved for Red Hat \*/
unsigned long rh\_reserved\[2\];
`#ifndef __GENKSYMS__`
struct perf\_event\_context \*perf\_event\_ctxp\[perf\_nr\_task\_contexts\];
`#ifdef CONFIG_CGROUP_MEM_RES_CTLR /* memcg uses this to do batch job */`
struct memcg\_batch\_info {
int do\_batch; /\* incremented when batch uncharge started \*/
struct mem\_cgroup \*memcg; /\* target memcg of uncharge \*/
unsigned long bytes; /\* uncharged usage \*/
unsigned long memsw\_bytes; /\* uncharged mem+swap usage \*/
} memcg\_batch;
`#endif`
`#endif`
};

如果需要，可从github处取走注释源码：<https://github.com/wsy081414/C_linux_practice/blob/master/task_struct.h>
