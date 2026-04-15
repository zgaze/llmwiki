---
source: https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c
---
## 12 个答案

排序：

65
[![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.svg]]](https://stackoverflow.com/posts/2180788/timeline)

无需调用不可移植的 API，例如`sendfile`，也无需使用外部实用程序。在 70 年代有效的相同方法现在仍然有效：

`#include <fcntl.h> #include <unistd.h> #include <errno.h> int cp(const char *to, const char *from) {     int fd_to, fd_from;     char buf[4096];     ssize_t nread;     int saved_errno;     fd_from = open(from, O_RDONLY);     if (fd_from < 0)         return -1;     fd_to = open(to, O_WRONLY | O_CREAT | O_EXCL, 0666);     if (fd_to < 0)         goto out_error;     while (nread = read(fd_from, buf, sizeof buf), nread > 0)     {         char *out_ptr = buf;         ssize_t nwritten;         do {             nwritten = write(fd_to, out_ptr, nread);             if (nwritten >= 0)             {                 nread -= nwritten;                 out_ptr += nwritten;             }             else if (errno != EINTR)             {                 goto out_error;             }         } while (nread > 0);     }     if (nread == 0)     {         if (close(fd_to) < 0)         {             fd_to = -1;             goto out_error;         }         close(fd_from);         /* Success! */         return 0;     }   out_error:     saved_errno = errno;     close(fd_from);     if (fd_to >= 0)         close(fd_to);     errno = saved_errno;     return -1; }`

[分享](https://stackoverflow.com/a/2180788)
[改进这个答案](https://stackoverflow.com/posts/2180788/edit)

[2010 年 2 月 1 日 23:45编辑](https://stackoverflow.com/posts/2180788/revisions)

于 2010 年 2 月 1 日 23:16 回答
[![dc6be45ddcfd038ba5e1e5fca319c7fa?s=64&d=identicon&r=PG](https://www.gravatar.com/avatar/dc6be45ddcfd038ba5e1e5fca319c7fa?s=64&d=identicon&r=PG)](https://stackoverflow.com/users/134633/caf)
[咖啡馆](https://stackoverflow.com/users/134633/caf)
**226k**3636枚金徽章311311银徽章457457铜牌

		1
	
	@Caf：OMG....转到.... :) 无论如何，您的代码比我的更理智... ;) 具有读/写功能的旧循环是最便携的... +1 来自我...
	– [t0mm13b](https://stackoverflow.com/users/206367/t0mm13b)
	[2010 年 2 月 2 日 0:03](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2128154_2180788)
	

		22
	
	我发现受控使用`goto`对于将错误处理路径整合到一个地方很有用。
	– [咖啡馆](https://stackoverflow.com/users/134633/caf)
	[2010 年 2 月 2 日 0:36](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2128280_2180788)
	
		10
	
	不可用于一般用途。文件的副本不仅仅是数据流。稀疏文件或扩展属性怎么样？这就是为什么 Windows API 如此丑陋的原因再次击败了 Linux
	– [洛萨](https://stackoverflow.com/users/155082/lothar)
	[2012 年 7 月 17 日 21:22](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment15243986_2180788)
	
		1
	
	您`EINTR`在`write()`循环中处理，但不在`read()`循环中。
	– [乔纳森·莱因哈特](https://stackoverflow.com/users/119527/jonathon-reinhart)
	[2015 年 4 月 22 日 17:33](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment47737633_2180788)
	
		4
	
	@Lothar Unix 文件在概念上只是一个字节序列。权限、ACL 等元数据与数据的实际复制正交处理。正如他们应该的那样。特定于应用程序的文件格式是应用程序的问题。正如他们应该的那样。
	– [wcochran](https://stackoverflow.com/users/398461/wcochran)
	[2018 年 2 月 20 日 19:11](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment84790646_2180788)
	

[显示另外**5**条评论](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#)

27
[![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.1.svg]]](https://stackoverflow.com/posts/2180157/timeline)

API 中没有内置的等效 CopyFile 函数。但是[sendfile](http://docs.oracle.com/cd/E23823_01/html/816-5172/sendfile-3ext.html#scrolltoc)可用于在内核模式下复制文件，这是一种比打开文件、循环读取文件并将输出写入另一个文件更快更好的解决方案（出于多种原因）。

更新：

从 Linux 内核版本 2.6.33 开始，要求输出为`sendfile`套接字的限制被取消，原始代码可以在 Linux 上运行，但是，从 OS X 10.9 Mavericks 开始，`sendfile`在 OS X 上现在要求输出为一个套接字，代码将不起作用！

以下代码片段应该适用于大多数 OS X（截至 10.5）、（免费）BSD 和 Linux（截至 2.6.33）。所有平台的实现都是“零拷贝”，这意味着所有这些都是在内核空间中完成的，并且没有缓冲区或数据进出用户空间的复制。几乎是您可以获得的最佳性能。

`#include <fcntl.h> #include <unistd.h> #if defined(__APPLE__) || defined(__FreeBSD__) #include <copyfile.h> #else #include <sys/sendfile.h> #endif int OSCopyFile(const char* source, const char* destination) {         int input, output;         if ((input = open(source, O_RDONLY)) == -1)     {         return -1;     }         if ((output = creat(destination, 0660)) == -1)     {         close(input);         return -1;     }     //Here we use kernel-space copying for performance reasons #if defined(__APPLE__) || defined(__FreeBSD__)     //fcopyfile works on FreeBSD and OS X 10.5+      int result = fcopyfile(input, output, 0, COPYFILE_ALL); #else     //sendfile will work with non-socket output (i.e. regular file) on Linux 2.6.33+     off_t bytesCopied = 0;     struct stat fileinfo = {0};     fstat(input, &fileinfo);     int result = sendfile(output, input, &bytesCopied, fileinfo.st_size); #endif     close(input);     close(output);     return result; }`

_编辑_`creat()`：用我们希望指定标志的调用替换目标的打开`O_TRUNC`。请参阅下面的评论。

[分享](https://stackoverflow.com/a/2180157)
[改进这个答案](https://stackoverflow.com/posts/2180157/edit)

[于 2017 年 7 月 7 日 9:46编辑](https://stackoverflow.com/posts/2180157/revisions)
[![5b36bf29056af38269b2147d5c36b94e?s=64&d=identicon&r=PG](https://www.gravatar.com/avatar/5b36bf29056af38269b2147d5c36b94e?s=64&d=identicon&r=PG)](https://stackoverflow.com/users/146377/patrick-schl%c3%bcter)
[帕特里克·施吕特](https://stackoverflow.com/users/146377/patrick-schl%c3%bcter)
**11k**11 个金徽章4040个银质徽章4747枚铜牌

于 2010 年 2 月 1 日 21:17 回答
[![fb75fbfd9dd8d93d49ff88c152d82c92?s=64&d=identicon&r=PG](https://www.gravatar.com/avatar/fb75fbfd9dd8d93d49ff88c152d82c92?s=64&d=identicon&r=PG)](https://stackoverflow.com/users/17027/mahmoud-al-qudsi)
[Mahmoud Al-Qudsi](https://stackoverflow.com/users/17027/mahmoud-al-qudsi)
**27.2k**1212个金徽章8383枚银质徽章123123枚铜牌

		4
	
	根据手册页，输出参数`sendfile`必须是套接字。你确定这行得通吗？
	– [杰伊·康罗德](https://stackoverflow.com/users/1891/jay-conrod)
	[2010 年 2 月 1 日 21:27](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127169_2180157)
	

		1
	
	对于 Linux，Jay Conrod 是对的——of`out_fd`可以`sendfile`是 2.4 内核中的常规文件，但它现在必须支持`sendpage`内部内核 API（本质上意味着管道或套接字）。 `sendpage`在不同的 UNIX 上以不同的方式实现 - 它没有标准语义。
	– [咖啡馆](https://stackoverflow.com/users/134633/caf)
	[2010 年 2 月 1 日 21:45](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127304_2180157) ![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.2.svg]]
	
		1
	
	Linux 下的原型与 OSX 不同，因此当我看到您的实现并看到 sendfile 的额外参数时，您会认为（我也认为）......它取决于平台 - 值得牢记！
	– [t0mm13b](https://stackoverflow.com/users/206367/t0mm13b)
	[2010 年 2 月 1 日 21:59](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127415_2180157) ![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.3.svg]]
	
		1
	
	仅供参考 - 您可以使用 if (PathsMatch(source, destination)) return 1; 节省大量工作。/\* 其中 PathsMatch 是区域设置的适当路径比较例程 \*/，否则我想第二次打开会失败。
	– [底座](https://stackoverflow.com/users/20481/plinth)
	[2010 年 2 月 2 日 1:44](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2128580_2180157)
	
		1
	
	+1 [man sendfile](http://linux.die.net/man/2/sendfile)说，从 2.6.33 开始，再次支持此功能。 `sendfile()`优于，`CopyFile()`因为它允许偏移。这对于从文件中剥离标题信息很有用。
	– [朴实无华的噪音](https://stackoverflow.com/users/1880339/artless-noise)
	[2013 年 11 月 25 日 22:41](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment30126599_2180157)
	

[显示另外**11**条评论](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#)

22
[![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.4.svg]]](https://stackoverflow.com/posts/2180347/timeline)

直接使用 fork/execl 运行 cp 为您完成工作。这与系统相比具有优势，因为它不容易受到 Bobby Tables 攻击，并且您不需要对参数进行相同程度的清理。此外，由于 system() 要求您将命令参数拼凑在一起，因此由于 sprintf() 检查草率，您不太可能遇到缓冲区溢出问题。

直接调用 cp 而不是编写它的优点是不必担心目标路径中存在的元素。在滚动你自己的代码中这样做是容易出错且乏味的。

我用 ANSI C 写了这个例子，只去掉了最简单的错误处理，除了它是直截了当的代码。

`void copy(char *source, char *dest) {     int childExitStatus;     pid_t pid;     int status;     if (!source || !dest) {         /* handle as you wish */     }     pid = fork();     if (pid == 0) { /* child */         execl("/bin/cp", "/bin/cp", source, dest, (char *)0);     }     else if (pid < 0) {         /* error - couldn't start process - you decide how to handle */     }     else {         /* parent - wait for child - this has all error handling, you          * could just call wait() as long as you are only expecting to          * have one child process at a time.          */         pid_t ws = waitpid( pid, &childExitStatus, WNOHANG);         if (ws == -1)         { /* error - handle as you wish */         }         if( WIFEXITED(childExitStatus)) /* exit code in childExitStatus */         {             status = WEXITSTATUS(childExitStatus); /* zero is normal exit */             /* handle non-zero as you wish */         }         else if (WIFSIGNALED(childExitStatus)) /* killed */         {         }         else if (WIFSTOPPED(childExitStatus)) /* stopped */         {         }     } }`

[分享](https://stackoverflow.com/a/2180347)
[改进这个答案](https://stackoverflow.com/posts/2180347/edit)

[2012 年 7 月 23 日 18:42编辑](https://stackoverflow.com/posts/2180347/revisions)
[![642e5a139877e650b37ebdd6e4628bc5?s=64&d=identicon&r=PG&f=1](https://www.gravatar.com/avatar/642e5a139877e650b37ebdd6e4628bc5?s=64&d=identicon&r=PG&f=1)](https://stackoverflow.com/users/181772/andrew-cooke)
[安德鲁·库克](https://stackoverflow.com/users/181772/andrew-cooke)
**44.2k**99个金徽章8888枚银质徽章142142个青铜徽章

于 2010 年 2 月 1 日 21:54 回答
[![dc41c338d8a89f8cec603f65f6fd0c89?s=64&d=identicon&r=PG](https://www.gravatar.com/avatar/dc41c338d8a89f8cec603f65f6fd0c89?s=64&d=identicon&r=PG)](https://stackoverflow.com/users/20481/plinth)
[底座](https://stackoverflow.com/users/20481/plinth)
**47.2k**1111个金徽章7777银徽章120120个青铜徽章

		1
	
	+1 又是一个漫长而详细的过程。真正让您欣赏 perl 中 system() 的“向量”/列表形式。唔。也许一个带有 argv 数组的系统函数会很好？！？
	– [机器人](https://stackoverflow.com/users/63369/roboprog)
	[2010 年 2 月 1 日 23:20](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127949_2180347)
	

		...毕竟它是在 17 年前在 glibc 中实现的，并且在您写下答案之前 10 年成为标准功能..
	– [Antti Haapala -- Слава Україні](https://stackoverflow.com/users/918959/antti-haapala-%d0%a1%d0%bb%d0%b0%d0%b2%d0%b0-%d0%a3%d0%ba%d1%80%d0%b0%d1%97%d0%bd%d1%96)
	[2017 年 10 月 19 日 21:27](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment80626035_2180347)
	

[添加评论](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#)

6
[![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.5.svg]]](https://stackoverflow.com/posts/25786694/timeline)

使用普通 POSIX 调用且没有任何循环的复制函数的另一个变体。代码灵感来自 caf 答案的缓冲区复制变体。警告：`mmap`在 32 位系统上使用很容易失败，在 64 位系统上危险的可能性较小。

`#include <fcntl.h> #include <unistd.h> #include <errno.h> #include <sys/mman.h> int cp(const char *to, const char *from) {   int fd_from = open(from, O_RDONLY);   if(fd_from < 0)     return -1;   struct stat Stat;   if(fstat(fd_from, &Stat)<0)     goto out_error;   void *mem = mmap(NULL, Stat.st_size, PROT_READ, MAP_SHARED, fd_from, 0);   if(mem == MAP_FAILED)     goto out_error;   int fd_to = creat(to, 0666);   if(fd_to < 0)     goto out_error;   ssize_t nwritten = write(fd_to, mem, Stat.st_size);   if(nwritten < Stat.st_size)     goto out_error;   if(close(fd_to) < 0) {     fd_to = -1;     goto out_error;   }   close(fd_from);   /* Success! */   return 0; } out_error:;   int saved_errno = errno;   close(fd_from);   if(fd_to >= 0)     close(fd_to);   errno = saved_errno;   return -1; }`

_编辑_：更正了文件创建错误。请参阅[http://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c/21​​80157#2180157](http://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c/2180157#2180157)答案中的评论。

[分享](https://stackoverflow.com/a/25786694)
[改进这个答案](https://stackoverflow.com/posts/25786694/edit)

[于 2017 年 7 月 7 日 9:52编辑](https://stackoverflow.com/posts/25786694/revisions)

于 2014 年 9 月 11 日 11:49 回答
[![5b36bf29056af38269b2147d5c36b94e?s=64&d=identicon&r=PG](https://www.gravatar.com/avatar/5b36bf29056af38269b2147d5c36b94e?s=64&d=identicon&r=PG)](https://stackoverflow.com/users/146377/patrick-schl%c3%bcter)
[帕特里克·施吕特](https://stackoverflow.com/users/146377/patrick-schl%c3%bcter)
**11k**11 个金徽章4040个银质徽章4747枚铜牌

		[与stackoverflow.com/questions/2180079](http://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c/2180157#2180157) /...中相同的错误 。如果目标已经存在并且比源大，那么文件副本只会部分覆盖目标而不截断生成的文件；
	– [帕特里克·施吕特](https://stackoverflow.com/users/146377/patrick-schl%c3%bcter)
	[2017 年 7 月 7 日 9:54](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment76910490_25786694)
	

		（我意识到这是一个老问题，但是......）当被映射的文件的大小与可用内存和交换文件的大小相比非常大时，mmap 会发生什么？会在内存不足/交换情况下挂起系统吗？
	– [本·斯莱德](https://stackoverflow.com/users/469352/ben-slade)
	[2018 年 2 月 13 日 16:28](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment84543762_25786694)
	
		将文件映射到进程的地址范围本身并不占用任何内存。就好像您说您的文件现在是交换空间的一部分。这意味着当您访问映射文件中的地址时，它将首先生成页面错误，因为内存中没有任何内容。然后操作系统从磁盘加载该地址的相应页面并将控制权恢复到进程。如果没有可用内存，那么操作系统将简单地从任何其他进程中释放一些其他映射页面；在优先清洁页面（即不需要写入磁盘）和脏页面中。\=>
	– [帕特里克·施吕特](https://stackoverflow.com/users/146377/patrick-schl%c3%bcter)
	[2018 年 2 月 14 日 13:48](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment84579166_25786694)
	
		当对映射页面的访问模式超过系统中的物理内存量并且它必须一直读取和写入页面时，就会发生交换。mmap 可以看作无非只是增加了系统交换区域。带有选项 MAP\_SHARED 的 mmap 也可以被视为使进程可以访问文件缓存的一种方式。
	– [帕特里克·施吕特](https://stackoverflow.com/users/146377/patrick-schl%c3%bcter)
	[2018 年 2 月 14 日 13:49](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment84579177_25786694)
	
		因此，如果您 mmap 一个大文件，然后访问大量文件，并且您访问的文件量大于您的实际内存，则操作系统将开始分页其他进程。如果这种情况发生太多，操作系统将开始在交换活动上颠簸。我的观点是，对于相对于内存+交换而言较大的文件，您必须考虑正在访问的 mmap 数据的大小，以免造成问题
	– [本·斯莱德](https://stackoverflow.com/users/469352/ben-slade)
	[2018 年 2 月 15 日 21:18](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment84636751_25786694)
	
[显示另外**1**条评论](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#)

4
[![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.6.svg]]](https://stackoverflow.com/posts/2180204/timeline)

有一种方法可以做到这一点，而无需求助于`system`调用，您需要合并一个包装器，如下所示：

`#include <sys/sendfile.h> #include <fcntl.h> #include <unistd.h> /*  ** http://www.unixguide.net/unix/programming/2.5.shtml  ** About locking mechanism... */ int copy_file(const char *source, const char *dest){    int fdSource = open(source, O_RDWR);    /* Caf's comment about race condition... */    if (fdSource > 0){      if (lockf(fdSource, F_LOCK, 0) == -1) return 0; /* FAILURE */    }else return 0; /* FAILURE */    /* Now the fdSource is locked */    int fdDest = open(dest, O_CREAT);    off_t lCount;    struct stat sourceStat;    if (fdSource > 0 && fdDest > 0){       if (!stat(source, &sourceStat)){           int len = sendfile(fdDest, fdSource, &lCount, sourceStat.st_size);           if (len > 0 && len == sourceStat.st_size){                close(fdDest);                close(fdSource);                /* Sanity Check for Lock, if this is locked -1 is returned! */                if (lockf(fdSource, F_TEST, 0) == 0){                    if (lockf(fdSource, F_ULOCK, 0) == -1){                       /* WHOOPS! WTF! FAILURE TO UNLOCK! */                    }else{                       return 1; /* Success */                    }                }else{                    /* WHOOPS! WTF! TEST LOCK IS -1 WTF! */                    return 0; /* FAILURE */                }           }       }    }    return 0; /* Failure */ }`

上面的示例（省略了错误检查！）使用`open`,`close`和`sendfile`.

**编辑：**正如**_caf_**指出的那样，和之间可能会发生_争用情况_`open`，`stat`所以我想我会让它变得更加健壮......请记住，锁定机制因平台而异......在 Linux 下，这种锁定机制`lockf`就足够了。如果您想让它可移植，请使用`#ifdef`宏来区分不同的平台/编译器...感谢 caf 发现这一点...这里有一个指向产生“通用锁定例程”的站点的[链接](http://web.mit.edu/shutkin/MacData_1124b/afs/sipb/project/source-sipb/source/third/mh/support/bboards/mmdfII/bboards/lock.c)。

[分享](https://stackoverflow.com/a/2180204)
[改进这个答案](https://stackoverflow.com/posts/2180204/edit)

[于 2017 年 6 月 30 日 14:10编辑](https://stackoverflow.com/posts/2180204/revisions)
[![2554d837bb158637bd537e8d315836ea?s=64&d=identicon&r=PG](https://www.gravatar.com/avatar/2554d837bb158637bd537e8d315836ea?s=64&d=identicon&r=PG)](https://stackoverflow.com/users/6532156/eduard-itrich)
[爱德华·伊特里希](https://stackoverflow.com/users/6532156/eduard-itrich)
**777**11 个金徽章77个银质徽章1919枚铜牌

于 2010 年 2 月 1 日 21:25 回答
[![121b360a4d7d20be6e220dcdb555649f?s=64&d=identicon&r=PG](https://www.gravatar.com/avatar/121b360a4d7d20be6e220dcdb555649f?s=64&d=identicon&r=PG)](https://stackoverflow.com/users/206367/t0mm13b)
[t0mm13b](https://stackoverflow.com/users/206367/t0mm13b)
**33.5k**88 金徽章7575枚银质徽章107107枚铜牌

		我不是 100% 确定 sendfile 原型，我想我弄错了一个参数...请记住这一点... :)
	– [t0mm13b](https://stackoverflow.com/users/206367/t0mm13b)
	[2010 年 2 月 1 日 21:29](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127187_2180204)
	

		您有一个竞争条件 - 您打开`fdSource`的文件和您拥有的文件`stat()ed`不一定相同。
	– [咖啡馆](https://stackoverflow.com/users/134633/caf)
	[2010 年 2 月 1 日 22:30](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127631_2180204) ![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.7.svg]]
	
		@caf：您能否在我查看时提供更多详细信息以及如何存在竞争条件？我会相应地修改答案..谢谢让我知道...
	– [t0mm13b](https://stackoverflow.com/users/206367/t0mm13b)
	[2010 年 2 月 1 日 23:53](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2128105_2180204)
	
		tommbieb75：很简单——在`open()`调用和`stat()`调用之间，其他人可以重命名文件并在该名称下放置一个不同的文件——所以你将从第一个文件复制数据，但使用第二个文件的长度。
	– [咖啡馆](https://stackoverflow.com/users/134633/caf)
	[2010 年 2 月 2 日 0:35](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2128268_2180204)
	
		@caf：天哪......为什么我没有想到......很好发现......锁应该在源文件上起到作用......很好地发现......竞争条件......好吧，我从来没有......正如“大都灵”中的克林特伊斯特伍德所说的“整个星期五......”
	– [t0mm13b](https://stackoverflow.com/users/206367/t0mm13b)
	[2010 年 2 月 2 日 0:57](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2128374_2180204)
	
[显示**另外 6**条评论](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#)

3
[![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.8.svg]]](https://stackoverflow.com/posts/65753723/timeline)

逐字节复制文件确实有效，但在现代 UNIX 上是缓慢且浪费的。现代 UNIX 在文件系统中内置了“写时复制”支持：系统调用创建一个指向磁盘上现有字节的新目录条目，并且在修改其中一个副本之前不会触及磁盘上的文件内容字节，此时只有更改的块被写入磁盘。这允许不使用额外文件块的近乎即时的文件复制，而不管文件大小。例如，这里有一些关于[它在 xfs 中如何工作的](https://blogs.oracle.com/linux/xfs-data-block-sharing-reflink)细节。

在 linux[上，默认](https://git.savannah.gnu.org/gitweb/?p=coreutils.git;a=commitdiff;h=25725f9d41735d176d73a757430739fb71c7d043)[使用`FICLONE` `ioctl`](https://man7.org/linux/man-pages/man2/ioctl_ficlonerange.2.html) [as coreutils cp](https://git.savannah.gnu.org/gitweb/?p=coreutils.git;a=blob;f=src/copy.c;h=159556d052158a6d3839b5b4ff69887471ff32da;hb=HEAD#l405) now 。<https://git.savannah.gnu.org/gitweb/?p=coreutils.git;a=commitdiff;h=25725f9d41735d176d73a757430739fb71c7d043>

 `#ifdef FICLONE    return ioctl (dest_fd, FICLONE, src_fd);  #else    errno = ENOTSUP;    return -1;  #endif`

在 macOS 上，使用[clonefile(2)](https://keith.github.io/xcode-man-pages/clonefile.2.html)在 APFS 卷上进行即时复制。这就是苹果公司的`cp -c`用途。文档并不完全清楚，但很可能[copyfile(3)`COPYFILE_CLONE`](https://keith.github.io/xcode-man-pages/copyfile.3.html)也使用它。如果您希望我对此进行测试，请发表评论。

如果不支持这些写时复制操作——无论是操作系统太旧、底层文件系统不支持它，还是因为你在不同的文件系统之间复制文件——你确实需要回退到尝试 sendfile，或者作为最后的手段，逐字节复制。但是为了节省大家大量的时间和磁盘空间，请先`FICLONE`尝试`clonefile(2)`一下。

[分享](https://stackoverflow.com/a/65753723)
[改进这个答案](https://stackoverflow.com/posts/65753723/edit)

[于 2021 年 1 月 23 日 22:58编辑](https://stackoverflow.com/posts/65753723/revisions)

于 2021 年 1 月 16 日 19:22 回答
[![701bda63fa3973ece37a74ecc2be5be9?s=64&d=identicon&r=PG](https://www.gravatar.com/avatar/701bda63fa3973ece37a74ecc2be5be9?s=64&d=identicon&r=PG)](https://stackoverflow.com/users/14558/andrewdotn)
[安德鲁多顿](https://stackoverflow.com/users/14558/andrewdotn)
**30.2k**77个金徽章9191银徽章122122个青铜徽章

[添加评论](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#)

2
[![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.9.svg]]](https://stackoverflow.com/posts/52343888/timeline)

`#include <unistd.h> #include <string.h> #include <errno.h> #include <fcntl.h> #include <sys/types.h> #include <sys/stat.h> #include <stdio.h> #define    print_err(format, args...)   printf("[%s:%d][error]" format "n", __func__, __LINE__, ##args) #define    DATA_BUF_SIZE                (64 * 1024)    //limit to read maximum 64 KB data per time int32_t get_file_size(const char *fname){     struct stat sbuf;     if (NULL == fname || strlen(fname) < 1){         return 0;     }     if (stat(fname, &sbuf) < 0){         print_err("%s, %s", fname, strerror(errno));         return 0;     }     return sbuf.st_size; /* off_t shall be signed interge types, used for file size */ } bool copyFile(CHAR *pszPathIn, CHAR *pszPathOut) {     INT32 fdIn, fdOut;     UINT32 ulFileSize_in = 0;     UINT32 ulFileSize_out = 0;     CHAR *szDataBuf;     if (!pszPathIn || !pszPathOut)     {         print_err(" Invalid param!");         return false;     }     if ((1 > strlen(pszPathIn)) || (1 > strlen(pszPathOut)))     {         print_err(" Invalid param!");         return false;     }     if (0 != access(pszPathIn, F_OK))     {         print_err(" %s, %s!", pszPathIn, strerror(errno));         return false;     }     if (0 > (fdIn = open(pszPathIn, O_RDONLY)))     {         print_err("open(%s, ) failed, %s", pszPathIn, strerror(errno));         return false;     }     if (0 > (fdOut = open(pszPathOut, O_CREAT | O_WRONLY | O_TRUNC, 0777)))     {         print_err("open(%s, ) failed, %s", pszPathOut, strerror(errno));         close(fdIn);         return false;     }     szDataBuf = malloc(DATA_BUF_SIZE);     if (NULL == szDataBuf)     {         print_err("malloc() failed!");         return false;     }     while (1)     {         INT32 slSizeRead = read(fdIn, szDataBuf, sizeof(szDataBuf));         INT32 slSizeWrite;         if (slSizeRead <= 0)         {             break;         }         slSizeWrite = write(fdOut, szDataBuf, slSizeRead);         if (slSizeWrite < 0)         {             print_err("write(, , slSizeRead) failed, %s", slSizeRead, strerror(errno));             break;         }         if (slSizeWrite != slSizeRead) /* verify wheter write all byte data successfully */         {             print_err(" write(, , %d) failed!", slSizeRead);             break;         }     }     close(fdIn);     fsync(fdOut); /* causes all modified data and attributes to be moved to a permanent storage device */     close(fdOut);     ulFileSize_in = get_file_size(pszPathIn);     ulFileSize_out = get_file_size(pszPathOut);     if (ulFileSize_in == ulFileSize_out) /* verify again wheter write all byte data successfully */     {         free(szDataBuf);         return true;     }     free(szDataBuf);     return false; }`

[分享](https://stackoverflow.com/a/52343888)
[改进这个答案](https://stackoverflow.com/posts/52343888/edit)

[于 2018 年 9 月 16 日 2:31编辑](https://stackoverflow.com/posts/52343888/revisions)

于 2018 年 9 月 15 日 10:37 回答
[![ml1Pk.jpg?s=64&g=1](https://i.stack.imgur.com/ml1Pk.jpg?s=64&g=1)](https://stackoverflow.com/users/5393174/kgbook)
[公斤书](https://stackoverflow.com/users/5393174/kgbook)
**348**33个银质徽章1515个青铜徽章

[添加评论](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#)

2
[![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.10.svg]]](https://stackoverflow.com/posts/2180115/timeline)

一种选择是您可以使用`system()`来执行`cp`. 这只是重新使用`cp(1)`命令来完成工作。如果您只需要创建另一个文件链接，可以使用`link()`或来完成`symlink()`。

[分享](https://stackoverflow.com/a/2180115)
[改进这个答案](https://stackoverflow.com/posts/2180115/edit)

[于 2020 年 10 月 23 日 16:58编辑](https://stackoverflow.com/posts/2180115/revisions)
[![7bfc7728cf3e556a8b9da88e662f4fd3?s=64&d=identicon&r=PG](https://www.gravatar.com/avatar/7bfc7728cf3e556a8b9da88e662f4fd3?s=64&d=identicon&r=PG)](https://stackoverflow.com/users/2430549/holdoffhunger)
[暂缓饥饿](https://stackoverflow.com/users/2430549/holdoffhunger)
**15.8k**88 金徽章8282枚银质徽章117117个青铜徽章

于 2010 年 2 月 1 日 21:10 回答
[![rpdwr.png?s=64&g=1](https://i.stack.imgur.com/rpdwr.png?s=64&g=1)](https://stackoverflow.com/users/15401/concernedoftunbridgewells)
[关注通布里奇韦尔斯](https://stackoverflow.com/users/15401/concernedoftunbridgewells)
**62.2k**1515个金徽章140140银徽章195195枚铜牌

		4
	
	请注意 system() 是一个安全漏洞。
	– [底座](https://stackoverflow.com/users/20481/plinth)
	[2010 年 2 月 1 日 21:12](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127032_2180115)
	

		1
	
	真的吗？你会在生产代码中使用它吗？我想不出一个很好的理由不这样做，但它并没有让我觉得它是一个_干净_的解决方案。
	– [莫蒂](https://stackoverflow.com/users/3848/motti)
	[2010 年 2 月 1 日 21:14](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127045_2180115)
	
		1
	
	如果你指定 /bin/cp 的路径，你就相对安全了，除非攻击者已经设法破坏系统，以至于他们可以修改 /bin 中的任意系统 shell 实用程序。如果他们对系统造成了一定程度的破坏，那么你就会遇到更大的问题。
	– [关注通布里奇韦尔斯](https://stackoverflow.com/users/15401/concernedoftunbridgewells)
	[2010 年 2 月 1 日 21:14](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127048_2180115)
	
		使用 system 来运行命令在 unix-land 中相当普遍。通过适当的卫生，它可以相当安全和坚固。毕竟，这些命令被设计为以这种方式使用。
	– [关注通布里奇韦尔斯](https://stackoverflow.com/users/15401/concernedoftunbridgewells)
	[2010 年 2 月 1 日 21:16](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127071_2180115)
	
		12
	
	如果用户创建像“somefile;rm /bin/\*”这样的文件名会发生什么？system() 使用 sh -c 执行命令，因此整个字符串的文本由 shell 执行，这意味着在将分号作为命令执行后您会得到任何东西 - 如果您的代码也在运行 setuid，那么会很臭。这与 Bobby Tables ( [xkcd.com/327](http://xkcd.com/327/) ) 没有什么不同。对于完全清理 system() 所带来的麻烦，您可以直接在 /bin/cp 上使用正确的参数执行 fork/exec 对。
	– [底座](https://stackoverflow.com/users/20481/plinth)
	[2010 年 2 月 1 日 21:27](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127173_2180115)
	
[显示另外**3**条评论](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#)

2
[![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.11.svg]]](https://stackoverflow.com/posts/68529953/timeline)

我看到还没有人提到[copy_file_range](https://man7.org/linux/man-pages/man2/copy_file_range.2.html)，至少在 Linux[和 FreeBSD](https://www.freebsd.org/cgi/man.cgi?query=copy_file_range&sektion=2&n=1)上受支持。这个的优点是它明确记录了使用 CoW 技术（如 reflinks）的能力。报价：

> `copy_file_range()`为文件系统提供了实现“复制加速”技术的机会，例如使用 reflinks _（即，两个或多个 inode 共享指向相同的写时复制磁盘块的指针）_或服务器端复制_（在NFS）_。

FWIW，我不确定老年人`sendfile`是否能够做到这一点。我发现的少数提及声称它没有。从这个意义上说，`copy_file_range`优于`sendfile`。

下面是使用调用的示例_（从手册中逐字复制）_。我还检查了在使用此代码`bash`在 BTRFS 文件系统中复制二进制文件后，该副本被重新链接到原始文件_（我通过调用`duperemove`文件并查看`Skipping - extents are already deduped.`消息来做到这一点）_。

`#define _GNU_SOURCE #include <fcntl.h> #include <stdio.h> #include <stdlib.h> #include <sys/stat.h> #include <unistd.h> int main(int argc, char **argv) {     int fd_in, fd_out;     struct stat stat;     off64_t len, ret;     if (argc != 3) {         fprintf(stderr, "Usage: %s <source> <destination>n", argv[0]);         exit(EXIT_FAILURE);     }     fd_in = open(argv[1], O_RDONLY);     if (fd_in == -1) {         perror("open (argv[1])");         exit(EXIT_FAILURE);     }     if (fstat(fd_in, &stat) == -1) {         perror("fstat");         exit(EXIT_FAILURE);     }     len = stat.st_size;     fd_out = open(argv[2], O_CREAT | O_WRONLY | O_TRUNC, 0644);     if (fd_out == -1) {         perror("open (argv[2])");         exit(EXIT_FAILURE);     }     do {         ret = copy_file_range(fd_in, NULL, fd_out, NULL, len, 0);         if (ret == -1) {             perror("copy_file_range");             exit(EXIT_FAILURE);         }         len -= ret;     } while (len > 0 && ret > 0);     close(fd_in);     close(fd_out);     exit(EXIT_SUCCESS); }`

[分享](https://stackoverflow.com/a/68529953)
[改进这个答案](https://stackoverflow.com/posts/68529953/edit)

于 2021 年 7 月 26 日 12:36 回答
[![4QzSp.jpg?s=64&g=1](https://i.stack.imgur.com/4QzSp.jpg?s=64&g=1)](https://stackoverflow.com/users/2388257/hi-angel)
[喜天使](https://stackoverflow.com/users/2388257/hi-angel)
**4,527**88 金徽章6161枚银质徽章8181枚铜牌

[添加评论](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#)

1
[![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.12.svg]]](https://stackoverflow.com/posts/2180124/timeline)

`sprintf( cmd, "/bin/cp -p '%s' '%s'", old, new); system( cmd);`

添加一些错误检查...

否则，打开两者并循环读/写，但可能不是你想要的。

...

更新以解决有效的安全问题：

与其使用“system()”，不如执行 fork/wait，然后在子进程中调用 execv() 或 execl()。

`execl( "/bin/cp", "-p", old, new);`

[分享](https://stackoverflow.com/a/2180124)
[改进这个答案](https://stackoverflow.com/posts/2180124/edit)

[于 2012 年 12 月 29 日 1:55编辑](https://stackoverflow.com/posts/2180124/revisions)

于 2010 年 2 月 1 日 21:11 回答
[![1bc2471f5a97b66082c3522697078ef7?s=64&d=identicon&r=PG](https://www.gravatar.com/avatar/1bc2471f5a97b66082c3522697078ef7?s=64&d=identicon&r=PG)](https://stackoverflow.com/users/63369/roboprog)
[机器人](https://stackoverflow.com/users/63369/roboprog)
**2,964**22个金徽章2626枚银质徽章2727枚铜牌

		这不适用于名称中包含空格（或引号、反斜杠、美元符号等）的文件。我经常在文件名中使用空格。
	– [迪特里希·埃普](https://stackoverflow.com/users/82294/dietrich-epp)
	[2010 年 2 月 1 日 21:44](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127300_2180124)
	

		哎哟。这是正确的。在 sprintf() 中的文件名周围添加反斜杠单引号。
	– [机器人](https://stackoverflow.com/users/63369/roboprog)
	[2010 年 2 月 1 日 21:45](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127306_2180124)
	
		好的，这是瑞士奶酪（请参阅其他地方的评论中的有效安全问题），但如果您有一个相对受控的环境，它可能会有一些用处。
	– [机器人](https://stackoverflow.com/users/63369/roboprog)
	[2010 年 2 月 1 日 21:47](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2127316_2180124)
	
		1
	
	You have a shell code injection vulnerability if you do not properly handle single quote characters in the values of `old` or `new`. A little more effort to use fork and do your own exec can avoid all these problems with quoting.
	– [Chris Johnsen](https://stackoverflow.com/users/193688/chris-johnsen)
	[Feb 2, 2010 at 1:59](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2128638_2180124)
	
		1
	
	Yep, simple obvious and wrong, in many cases. Which is why I up-voted some of the more elaborate examples.
	– [Roboprog](https://stackoverflow.com/users/63369/roboprog)
	[Feb 2, 2010 at 18:19](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment2134214_2180124)
	
[Add a comment](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#)

1
[![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.13.svg]]](https://stackoverflow.com/posts/66521396/timeline)

Very simple :

`#define BUF_SIZE 65536 int cp(const char *from, const char*to){ FILE *src, *dst; size_t in, out; char *buf = (char*) malloc(BUF_SIZE* sizeof(char)); src = fopen(from, "rb"); if (NULL == src) exit(2); dst = fopen(to, "wb"); if (dst < 0) exit(3); while (1) {     in = fread(buf, sizeof(char), BUF_SIZE, src);     if (0 == in) break;     out = fwrite(buf, sizeof(char), in, dst);     if (0 == out) break; } fclose(src); fclose(dst); }`

Works on windows and linux.

[Share](https://stackoverflow.com/a/66521396)
[Improve this answer](https://stackoverflow.com/posts/66521396/edit)

[edited Mar 8, 2021 at 12:55](https://stackoverflow.com/posts/66521396/revisions)

answered Mar 7, 2021 at 21:02
[![fh4e0.jpg?s=64&g=1](https://i.stack.imgur.com/fh4e0.jpg?s=64&g=1)](https://stackoverflow.com/users/15342240/mbaram)
[MBaram](https://stackoverflow.com/users/15342240/mbaram)
**23**55 bronze badges

		`cp()` doesn't return anything, yet it is typed as `int`, which could cause problems and might even be UB.
	– [dfalzone - Reinstate Monica](https://stackoverflow.com/users/10942736/dfalzone-reinstate-monica)
	[Mar 26 at 1:20](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment126586239_66521396) ![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.14.svg]]
	

		Seems like free(buf) is missing?
	– [Wil S](https://stackoverflow.com/users/121421/wil-s)
	[Jun 2 at 18:55](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#comment128039742_66521396)
	

[Add a comment](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#)

0
[![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.15.svg]]](https://stackoverflow.com/posts/68695871/timeline)

Good question. Related to another good question:

[In C on linux how would you implement cp](https://stackoverflow.com/questions/33792754/in-c-on-linux-how-would-you-implement-cp/68716245#68716245)

There are two approaches to the "simplest" implementation of cp. One approach uses a file copying system call function of some kind - the closest thing we get to a C function version of the Unix cp command. The other approach uses a buffer and read/write system call functions, either directly, or using a FILE wrapper.

It's likely the file copying system calls that take place solely in kernel-owned memory are faster than the system calls that take place in both kernel- and user-owned memory, especially in a network filesystem setting (copying between machines). But that would require testing (e.g. with Unix command time) and will be dependent on the hardware where the code is compiled and executed.

It's also likely that someone with an OS that doesn't have the standard Unix library will want to use your code. Then you'd want to use the buffer read/write version, since it only depends on <stdlib.h> and <stdio.h> (and friends)

### <unistd.h>

Here's an example that uses function `copy_file_range` from the unix standard library `<unistd.h>`, to copy a source file to a (possible non-existent) destination file. The copy takes place in kernel space.

`/* copy.c  *  * Defines function copy:  *  * Copy source file to destination file on the same filesystem (possibly NFS).  * If the destination file does not exist, it is created. If the destination  * file does exist, the old data is truncated to zero and replaced by the   * source data. The copy takes place in the kernel space.  *  * Compile with:  *  * gcc copy.c -o copy -Wall -g  */ #define _GNU_SOURCE  #include <fcntl.h> #include <stdio.h> #include <stdlib.h> #include <sys/stat.h> #include <sys/syscall.h> #include <unistd.h> /* On versions of glibc < 2.27, need to use syscall.  *   * To determine glibc version used by gcc, compute an integer representing the  * version. The strides are chosen to allow enough space for two-digit   * minor version and patch level.  *  */ #define GCC_VERSION (__GNUC__*10000 + __GNUC_MINOR__*100 + __gnuc_patchlevel__) #if GCC_VERSION < 22700 static loff_t copy_file_range(int in, loff_t* off_in, int out,    loff_t* off_out, size_t s, unsigned int flags) {   return syscall(__NR_copy_file_range, in, off_in, out, off_out, s,     flags); } #endif /* The copy function.  */ int copy(const char* src, const char* dst){   int in, out;   struct stat stat;   loff_t s, n;   if(0>(in = open(src, O_RDONLY))){     perror("open(src, ...)");     exit(EXIT_FAILURE);   }   if(fstat(in, &stat)){     perror("fstat(in, ...)");     exit(EXIT_FAILURE);   }   s = stat.st_size;    if(0>(out = open(dst, O_CREAT|O_WRONLY|O_TRUNC, 0644))){     perror("open(dst, ...)");     exit(EXIT_FAILURE);   }   do{     if(1>(n = copy_file_range(in, NULL, out, NULL, s, 0))){       perror("copy_file_range(...)");       exit(EXIT_FAILURE);     }     s-=n;   }while(0<s && 0<n);   close(in);   close(out);   return EXIT_SUCCESS; } /* Test it out.  *  * BASH:  *  * gcc copy.c -o copy -Wall -g  * echo 'Hello, world!' > src.txt  * ./copy src.txt dst.txt  * [ -z "$(diff src.txt dst.txt)" ]  *  */ int main(int argc, char* argv[argc]){   if(argc!=3){     printf("Usage: %s <SOURCE> <DESTINATION>", argv[0]);     exit(EXIT_FAILURE);   }   copy(argv[1], argv[2]);   return EXIT_SUCCESS; }`

It's based on the example in my Ubuntu 20.x Linux distribution's man page for copy\_file\_range. Check your man pages for it with:

`> man copy_file_range`

Then hit `j` or `Enter` until you get to the example section. Or search by typing `/example`.

### <stdio.h>/<stdlib.h> only

Here's an example that only uses `stdlib/stdio`. The downside is it uses an intermediate buffer in user-space.

`/* copy.c  *  * Compile with:  *   * gcc copy.c -o copy -Wall -g  *  * Defines function copy:  *  * Copy a source file to a destination file. If the destination file already  * exists, this clobbers it. If the destination file does not exist, it is  * created.   *  * Uses a buffer in user-space, so may not perform as well as   * copy_file_range, which copies in kernel-space.  *  */ #include <stdlib.h> #include <stdio.h> #define BUF_SIZE 65536 //2^16 int copy(const char* in_path, const char* out_path){   size_t n;   FILE* in=NULL, * out=NULL;   char* buf = calloc(BUF_SIZE, 1);   if((in = fopen(in_path, "rb")) && (out = fopen(out_path, "wb")))     while((n = fread(buf, 1, BUF_SIZE, in)) && fwrite(buf, 1, n, out));   free(buf);   if(in) fclose(in);   if(out) fclose(out);   return EXIT_SUCCESS; } /* Test it out.  *  * BASH:  *  * gcc copy.c -o copy -Wall -g  * echo 'Hello, world!' > src.txt  * ./copy src.txt dst.txt  * [ -z "$(diff src.txt dst.txt)" ]  *  */ int main(int argc, char* argv[argc]){   if(argc!=3){     printf("Usage: %s <SOURCE> <DESTINATION>n", argv[0]);     exit(EXIT_FAILURE);   }   return copy(argv[1], argv[2]); }`

[Share](https://stackoverflow.com/a/68695871)
[Improve this answer](https://stackoverflow.com/posts/68695871/edit)

[edited Aug 10, 2021 at 4:09](https://stackoverflow.com/posts/68695871/revisions)

answered Aug 7, 2021 at 20:25
[![8ewzQ.jpg?s=64&g=1](https://i.stack.imgur.com/8ewzQ.jpg?s=64&g=1)](https://stackoverflow.com/users/15347182/angstyloop)
[angstyloop](https://stackoverflow.com/users/15347182/angstyloop)
**59**55 bronze badges

[Add a comment](https://stackoverflow.com/questions/2180079/how-can-i-copy-a-file-on-unix-using-c#)

## 你的答案

### 注册或[登录](https://stackoverflow.com/users/login?ssrc=question_page&returnurl=https%3a%2f%2fstackoverflow.com%2fquestions%2f2180079%2fhow-can-i-copy-a-file-on-unix-using-c%23new-answer)

![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.16.svg]]使用谷歌注册
![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.17.svg]]使用 Facebook 注册
![[./_resources/如何使用_C_在_Unix_上复制文件？_-_堆栈溢出.resources/embedded.18.svg]]使用电子邮件和密码注册

### 以访客身份发帖

姓名

电子邮件

必需，但从未显示

点击“发布您的答案”，即表示您同意我们[的服务条款](https://stackoverflow.com/legal/terms-of-service/public)、[隐私政策](https://stackoverflow.com/legal/privacy-policy)和[cookie 政策](https://stackoverflow.com/legal/cookie-policy)

## 不是您要找的答案？浏览标记的其他问题[C](https://stackoverflow.com/questions/tagged/c) [Unix](https://stackoverflow.com/questions/tagged/unix) [复制](https://stackoverflow.com/questions/tagged/copy) 或者[问你自己的问题](https://stackoverflow.com/questions/ask)。
