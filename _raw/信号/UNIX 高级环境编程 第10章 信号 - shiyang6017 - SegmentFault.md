---
source: https://segmentfault.com/a/1190000007517934
---
10.1 引言
**定义**：
　　 信号是软件中断，其提供了一种处理异步事件的方法。

## 10.2 信号概念

**信号处理方式**：

1. 忽略信号
	

		捕捉信号
	
		执行默认系统动作，对于大多数系统默认动作是终止该进程
	**注意：SIGKILL和SIGSTOP 信号不能被忽略，也不能被捕捉。**
	

## 10.3 函数 `signal`

    #include <signal>
    void (*signal(int sig, void(*handler)(int)))(int);

**函数说明：**

1. sig: 信号类型
	

		handler： 处理sig信号的函数的指针
	
		signal的返回值： **原先**处理sig信号的函数的指针
	

**信号的状态**
1.进程启动
　　当执行一个程序时，所有信号的状态都是系统默认的。 除非调用exec的进程忽略该信号。确切地讲，exec函数将原先设置为要捕捉的信号都更改为默认动作，其他信号的状态则不变。（课本举了一个例子: 在bash下，执行"./a.out &"， 产生的后台进程自动忽略SIGINT与SIGSTOP）
2.进程创建
　　当一个进程调用fork时，其子进程继承父进程的信号处理方式

## 10.5 中断的系统调用

对于Linux操作系统而言，如果进程在执行一个低速系统调用而阻塞期间捕捉到一个信号，则该系统调用：

1. 在信号处理程序完成后，自动重启.
	(SA\_RESTART置位后，如 write， read，open，wait， waitpid， fcntl等)
	

		系统调用失败，返回EINTR（无论SA\_RESTART是否置位，如pause, sigsuspend，nanosleep等）
	**注意：sleep 永远不会被重启，返回一个成功值：剩余的休眠时间**
	

## 10.6 可重入函数

问题：
如果进程正在执行malloc，在其堆中分配另外的存储空间，而此时由于捕捉到信号而插入执行该信号处理程序，其中又调用malloc，那么可能就会出现严重错误。所以，对于malloc而言，它就**不是**可重入代码

不可重入函数的特征：

1. 使用了静态数据结构，或全局数据结构
	

		调用malloc或free函数
	
		它们是标准I/O（很多标准I/O实现以不可重入方式使用了全局数据结构）
	

举例：在信号处理程序中调用prinf函数，当main函数也使用了printf时，则不保证出现期望的结果
**注意**：由于一个线程只有一个errno值，而信号处理程序可能改变errno值。 因此，在信号处理程序中首先应该保存原先的errno。待处理完信号后，恢复原有的errno。比如：SIGCHLD的信号处理程序 往往会调用waitpid函数，显然waitpid可能会产生ECHILD，从而覆盖了原有的errno。

    void handler(int sig){
        int errno_saved = errno;
        // proccess sig
        errno = errno_saved; 
    }

理解longjmp与siglongjmp为什么不是可重入的。

## 10.7 `SIGCLD` 语义

在 linux 3.10.0 中语义与SIGCHLD 相同

## 10.8 可靠信号术语和语义

1. 信号的产生（generation）： 造成信号的发生（硬件异常，软件条件）
	
2. 信号的传递（delivery）：对信号调用信号处理程序或采用系统默认处理方式
	
3. 信号的未决（pending）：信号已经产生，当还没有传递的中间状态
	

多数unix操作系统并不对信号进行排队，而是只传递一次信号

## 10.9 函数`kill`和`raise`

    int kill(pid_t pid,int signo)
    
    int raise(int signo); 等价于 kill(getpid(), signo)

理解不同pid的值代表的含义。对于非超级用户，发送者的实际用户ID或有效用户ID必须等于接受者的实际用户ID或有效用户ID

## 10.10 函数 `alarm`和 `pause`

    unsigned int alarm(unsigned int sec);

理解第二次设置alarm时，alarm函数的返回值。

    int pause(void);

只有执行了一个信号处理程序并从其返回时，pause才返回，pause返回-1，errno设置为EINTR

## 11.11 信号集

    #include <signal.h>int sigemptyset (sigset_t* set); //置0int sigfillset (sigset_t* set); //置1int sigaddset (sigset_t* set, int sig);
    int sigdelset (sigset_t* set, int sig);
                                          // return 0 if success, else return -1int sigismember(const sigset_t* set, int sig); // true or false 

## 11.12 函数 `sigpromask`

    #include <signal>int sigpromask(int how, const sigset_t*restrict set, sigset_t*restrict oset);
                                         // return 0 if success, else return -1

		若oset是非空指针，那么进程的当前信号屏蔽字通过oset返回
	
		若set是一个非空指针，则参数how指示如何修改当前信号屏蔽字。
	
	     SIG_BLOCK: 当前信号屏蔽字并上set指向的信号屏蔽字
	    
	     SIG_UNBLOCK: 当前信号屏蔽字去掉set
	     
	     SIG_SETMASK: set替换当前信号屏蔽字
	
		如果set为空，那么how的参数没有意义
	

    　　　sigpromask(0, NULL, oset); // 获取当前进程的信号屏蔽字

## 10.13 函数 `sigpending`

		返回未决信号
	

    int sigpending(sigset_t* set);

		注意第三个point
	

    #include "apue.h"static void handler(int sig){
        pid_t pid;
        /*
         * point 1
         * saved the orignal errno
         * */
        int err_saved = errno;
        while ((pid = wait(NULL)) > 0)
        {
             /* point 2
              *  printf is not Re-entrant code, so the line of code below is a bad example.
              * */
            printf("child %d has finished.\n", pid);
        }
        if (errno != ECHILD)
            err_sys("wait err.");
        errno = err_saved;
    }
    
    void example();
    int main(){
        /*other parts*/
        example();
        /*other parts*/
        return 0;
    }

    void example(){
        sigset_t oldset, newset, pendingset;
        sigemptyset(&newset);
        sigaddset(&newset, SIGCHLD);
        sigaddset(&newset, SIGUSR1);
    
        void (*old_sigchild_handler)(int);
        void (*old_sigusr1_handler)(int);
        if ((old_sigchild_handler =  signal(SIGCHLD, handler)) == SIG_ERR)
            err_sys("signal err.");
        if ((old_sigusr1_handler=signal(SIGCHLD, handler)) == SIG_ERR)
               err_sys("signal err.");
    
        if (sigprocmask(SIG_BLOCK, &newset, &oldset) < 0)
            err_sys("sigpromask err.");
    
        int chld_num = 4;
        while (chld_num--){
            pid_t pid;
            if ((pid = fork()) == 0)
                exit(0);
        }
        sleep(1);
        sigpending(&pendingset);
        if (sigismember(&pendingset, SIGCHLD))
              printf("pending sigchld."); // will print
        if (sigismember(&pendingset, SIGUSR1))
                printf("pending sigusr1."); // will not print
        if (sigprocmask(SIG_SETMASK, &oldset, NULL) < 0)
               err_sys("sigpromask err.");
        /*
         * point 3
         *  if (sigpromask(SIG_UNBLOCK, &newset, NULL) < 0)
               err_sys("sigpromask err.");
         *  This is a bad way to reset the mask set.
         *  because the code BEFORE sample function may also mask SIGCHLD or SIGURS1 signal.
         *  so what you do here will badly affect other function.
         * */
        if (signal(SIGCHLD, old_sigchild_handler) == SIG_ERR)
             err_sys("signal err.");
         if (signal(SIGCHLD, old_sigusr1_handler) == SIG_ERR)
                err_sys("signal err.");
    }

## 10.14 函数 `sigaction`

		检查或修改与指定信号关联的处理动作
	

    int sigaction(int signo, const struct sigaction* act, struct sigaction* oact);

		数据结构 `struct sigaction`
	

    struct sigaction{
        void (*sa_handler)(int);
        sigset_t sa_mask;
        int sa_flags; 
        void (*sa_sigaction)(int, siginfo_t*, void*); 
    }

		要点
	

　　　　`sa_handler`: 用来处理signo的信号捕捉函数

　　　　`sa_mask`: 在进程接收到信号signo之后，调用信号捕捉函数之前，这一信号集要加入到进程的信号屏蔽字中。
　　　　　　　　仅当从信号捕捉函数返回时再将进程的信号屏蔽字恢复为原先值。这样,在调用信号捕捉函数时就能阻塞某些信号.
　　　　　　　　注意，在信号处理函数被调用时，操作系统建立的新信号屏蔽字包括正被传递的信号。
　　　　　　　
　　　　`sa_flags`:成员用于指定信号处理的行为，它可以是以下值的“按位或”组合。

                [SA_RESTART]：使被信号打断的系统调用自动重新发起。
                [SA_SIGINFO]：使用 sa_sigaction 成员而不是 sa_handler 作为信号处理函数。
                [SA_INTERRUPT]: 由此信号中断的系统调用不自动重启。
                SA_ONSTACK：系统将在调用[[sigalstack|sigalstack]]替代信号栈上运行信号句柄；否则使用用户栈来交付信号。
                SA_NOCLDSTOP：使父进程在它的子进程暂停或继续运行时不会收到 SIGCHLD 信号。
                SA_NOCLDWAIT：使父进程在它的子进程退出时不会收到 SIGCHLD 信号，这时子进程如果退出也不会成为僵尸进程。
                SA_NODEFER：使对信号的屏蔽无效，即在信号处理函数执行期间仍能发出这个信号。
                SA_RESETHAND：信号处理之后重新设置为默认的处理方式。
    

		如果设置了`SA_SIGINFO`标志，那么按下列方式调用信号处理程序
	

      void handler(int signo, siginfo_t *info, void* context)

		注意 `struct siginfo` 数据结构在不同平台的实现
	

    typedef struct
      {
        int si_signo;        /* Signal number.  */
        int si_errno;        /* If non-zero, an errno value associated with
                       this signal, as defined in <errno.h>.  */
        int si_code;        /* Signal code.  */
    
        union
          {
        int _pad[__SI_PAD_SIZE];
    
         /* kill().  */
        struct
          {
            __pid_t si_pid;    /* Sending process ID.  */
            __uid_t si_uid;    /* Real user ID of sending process.  */
          } _kill;
    
        /* POSIX.1b timers.  */
        struct
          {
            int si_tid;        /* Timer ID.  */
            int si_overrun;    /* Overrun count.  */
            sigval_t si_sigval;    /* Signal value.  */
          } _timer;
    
        /* POSIX.1b signals.  */
        struct
          {
            __pid_t si_pid;    /* Sending process ID.  */
            __uid_t si_uid;    /* Real user ID of sending process.  */
            sigval_t si_sigval;    /* Signal value.  */
          } _rt;
    
        /* SIGCHLD.  */
        struct
          {
            __pid_t si_pid;    /* Which child.  */
            __uid_t si_uid;    /* Real user ID of sending process.  */
            int si_status;    /* Exit value or signal.  */
            __sigchld_clock_t si_utime;
            __sigchld_clock_t si_stime;
          } _sigchld;
    
        /* SIGILL, SIGFPE, SIGSEGV, SIGBUS.  */
        struct
          {
            void *si_addr;    /* Faulting insn/memory ref.  */
          } _sigfault;
    
        /* SIGPOLL.  */
        struct
          {
            long int si_band;    /* Band event for SIGPOLL.  */
            int si_fd;
          } _sigpoll;
    
        /* SIGSYS.  */
        struct
          {
            void *_call_addr;    /* Calling user insn.  */
            int _syscall;    /* Triggering system call number.  */
            unsigned int _arch; /* AUDIT_ARCH_* of syscall.  */
          } _sigsys;
          } _sifields;
      } siginfo_t __SI_ALIGNMENT;
    

		`struct siginfo`数据结构中需要特别关注的一个成员数据结构:
	

    typedef union sigval
    {
        int sival_int;
        void *sival_ptr;
    } sigval_t

		用户可以通用`sigqueue`函数对该成员进行赋值。需要注意的是不能通过如下方式访问该成员
	

    info->_sifields._rt.si_sigval; // error

		因为GNU-C库对该成员进行了宏定义
	

    # define si_value    _sifields._rt.si_sigval

		所以直接使用`info->si_value`就可以访问该成员数据结构了。
	

    void sigusr1_handler(int signal){
        int save_err = errno; 
        assert (signal == SIGUSR1);
        printf ("receiving SIGUSR1.\n");
        errno = save_err;
    }
    
    int main(){
        struct sigaction saction;
        saction.sa_handler = sigusr1_handler;
        sigemptyset(&saction.sa_mask);
        saction.sa_flags = 0;
       
        if (sigaction(SIGUSR1, &saction, NULL) < 0){
            fprintf(stderr, "sigaction err %s", strerror(errno));
            exit(0);
        }
    
        sigset_t mask_set, old_mask_set;
    
        sigemptyset(&mask_set);
        sigaddset(&mask_set,SIGUSR1);
        if (sigprocmask(SIG_BLOCK, &mask_set, &old_mask_set) < 0){
            fprintf(stderr, "sigpromask err : %s", strerror(errno));
            exit(0);
        }
        raise (SIGUSR1);
        // no response
        sigset_t pending_sig;
        if (sigpending(&pending_sig) < 0){
            fprintf(stderr, "sigpending err : %s", strerror(errno));
            exit(0);
        }
        if (sigismember(&pending_sig,SIGUSR1)){
            printf("it is a SIGUSR1.\n");
        }
        // print "it is a SIGUSR1."
        if (sigprocmask(SIG_SETMASK, &old_mask_set, NULL) < 0){
            fprintf(stderr, "sigpromask err : %s", strerror(errno));
            exit(0);
        }
        // print "receiving SIGURS1"
        return 0;
    }
    

## 10.15 函数`sigsetjmp`和`siglongjmp`

		这两个函数要解决的问题：在信号处理程序中经常调用setjmp和longjmp函数以返回到程序的主函数中。但是存在一个问题：
	当捕捉到一个信号时，进入信号捕捉函数，此时当前信号被自动地加入到进程的信号屏蔽字中。如果用longjmp跳出信号处理程序。那么当前信号会永久的加入到信号屏蔽字中（至少在linux 3.10上 是这样的）。
	

    int sigsetjmp(sigjmp_buf env, int savemask);
    /* 若直接调用，返回0；若从siglongjmp调用返回，则返回非0
     - 若savamask非0，则sigsetjmp在env中保存进程的当前信号屏蔽字，则siglongjmp从其中恢复保存的信号屏蔽字。
     */void siglongjmp(sigjmp_buf env, int val);

		注意 图10-20中的代码在(linux 3.10. gcc 4.8.5) 平台上的的测试结果和课本不同。
	

## 10.16 函数 `sigsuspend`

		此函数解决如下伪代码的问题： 在3和4之间存在窗口期，可能导致进程一直暂停
	

    屏蔽信号signo； //1
    critical region of code; //2
    解除信号屏蔽；//3
    pause(); // 4, 等待signo的到来

    int sigsuspend(const sigset_t *sigmask);
    //将进程的屏蔽字设置为sigmask，然后暂停。直到有除sigmask之外的信号传递，唤醒进程

		注意从sisuspend返回时，将进程的信号屏蔽字恢复为原来的值。
	
		`sigsuspend`的另外两个应用:
	
	       1.等待一个信号处理程序设置一个全局变量；
	       2.实现父子进程之间的同步
	    
	

## 10.17 函数 `abort`

    void abort(void); // raise(SIGABRT), kill(getpid(), SIGABRT);

    class A{
    public:
        ~A(){ std::cout << "A destruct done." << std::endl; }
    };
    class B{
    public:
        ~B(){ std::cout << "B destruct done." << std::endl; }
    };
    
    A a;
    int main(){
        B b; 
        // static B b; different 
        /*
        exit(0); // only print "A destruct done."
        */
    
        /*
        abort(); // aborted(core dump) 
        */ 
    }
    

## 10.18 函数 `system`

		`POSIX.1`要求`system`忽略`SIGINT`和`SIGQUIT`,阻塞`SIGCHLD`.
	
		结合`system`函数的具体实现理解“阻塞 `SIGCHLD`”
	
		由system执行的命令可能是交互式命令，以及因为system的调用者在程序执行时放弃了控制，等待该命令的结束。
	所以对于system的调用者应该`system`忽略`SIGINT`和`SIGQUIT`
	
		`system`的返回值： shell的终止状态，而不是cmd的返回值
	

## 10.19 函数 `sleep`和`nanosleep`和`clock_nanosleep`

## 10.20 函数`sigqueue`

**使用排队信号的一个操作：**

1. 使用`sigaction`函数安装信号处理程序时指定`SA_SIGINFO`.
	
2. 在`sigaction`结构的`sa_sigaction`成员中提供信号处理程序.
	
3. 使用`sigqueue`函数发送信号。（使用`kill`也可以,可靠信息的现实取决于可靠信号本身，而不是函数）
	

    int sigqueue(pid_t pid, int signo, const union sigval value);

**和kill的比较**

1. `sigqueue`信号只能把信号发送给单个进程
	
2. `sigqueue`使用`value`参数向信号处理程序传递整数和指针值
	
3. 其他都一样，即两者都可以有效传递可靠信号
	

**实时信号范围：`[SIGRTMIN, SIGRTMAX]`**，默认行为终止进程。
**注意： 在`Linux`上,即使调用者没有使用`SA_SIGINFO`,也对可靠信号进行排队**
