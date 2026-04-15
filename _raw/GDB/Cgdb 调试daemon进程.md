---
source: http://www.cnblogs.com/habibah-chang/p/3510529.html
---
缺省gdb是调试主进程的，可是现在采用daemon模式工作的程序那么多，主进程通常很快就结束了，子进程才是真正干活的。怎么跟踪调试子进程呢？
在
则gdb就可以调试子进程了。gdb里面执行：
set follow-fork-mode child
set follow-fork-mode parent

<https://www.ibm.com/developerworks/cn/linux/l-cn-gdbmp/> 
gdb 调试多进程

   如果想知道程序现在运行到了哪里，同样可以使用“backtrace ”命令。当然也可以使用“step”命令对程序进行单步调试。
