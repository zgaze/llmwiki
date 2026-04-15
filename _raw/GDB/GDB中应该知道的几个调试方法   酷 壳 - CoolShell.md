---
source: https://coolshell.cn/articles/3643.html
---
# GDB中应该知道的几个调试方法

七、八年前写过一篇《[用GDB调试程序](http://blog.csdn.net/haoel/archive/2003/07/02/2879.aspx)》，于是，从那以后，很多朋友在MSN上以及给我发邮件询问我关于GDB的问题，一直到今天，还有人在问GDB的相关问题。这么多年来，有一些问题是大家反复在问的，一方面，我觉得我以前的文章可能没有说清楚，另一方面，我觉得大家常问的问题正是最有用的，所以，在这里罗列出来。希望大家补充。

#### 一、多线程调试

多线程调试可能是问得最多的。其实，重要就是下面几个命令：

* info thread 查看当前进程的线程。
* thread <ID> 切换调试的线程为指定ID的线程。
* break file.c:100 thread all  在file.c文件第100行处为所有经过这里的线程设置断点。
* set scheduler-locking off|on|step，这个是问得最多的。在使用step或者continue命令调试当前被调试线程的时候，其他线程也是同时执行的，怎么只让被调试程序执行呢？通过这个命令就可以实现这个需求。
	* off 不锁定任何线程，也就是所有线程都执行，这是默认值。
	* on 只有当前被调试程序会执行。
	* step 在单步的时候，除了next过一个函数的情况(熟悉情况的人可能知道，这其实是一个设置断点然后continue的行为)以外，只有当前线程会执行。

#### 二、调试宏

这个问题超多。在GDB下，我们无法print宏定义，因为宏是预编译的。但是我们还是有办法来调试宏，这个需要GCC的配合。

在GCC编译程序的时候，加上**\-ggdb3**参数，这样，你就可以调试宏了。

另外，你可以使用下述的GDB的宏调试命令 来查看相关的宏。

* info macro – 你可以查看这个宏在哪些文件里被引用了，以及宏定义是什么样的。
* macro – 你可以查看宏展开的样子。

#### 三、源文件

这个问题问的也是很多的，太多的朋友都说找不到源文件。在这里我想提醒大家做下面的检查：

1. 编译程序员是否加上了-g参数以包含debug信息。
2. 路径是否设置正确了。使用GDB的directory命令来设置源文件的目录。

下面给一个调试/bin/ls的示例（ubuntu下）

|     |     |
| --- | --- |
| 1<br>2<br>3<br>4<br>5<br>6<br>7<br>8<br>9<br>10<br>11<br>12<br>13<br>14<br>15<br>16<br>17<br>18<br>19<br>20 | `$ apt-get` `source` `coreutils`<br>`---
source: https://coolshell.cn/articles/3643.html
---
 `sudo` `apt-get` `install` `coreutils-dbgsym`<br>`---
source: https://coolshell.cn/articles/3643.html
---
 `gdb` `/bin/ls`<br>`GNU` `gdb` `(GDB) 7.1-ubuntu`<br>`(``gdb``) list main`<br>`1192`    `ls``.c: No such` `file` `or directory.`<br>`in` `ls``.c`<br>`(``gdb``) directory ~``/src/coreutils-7``.4``/src/`<br>`Source directories searched:` `/home/hchen/src/coreutils-7``.4:$cdir:$cwd`<br>`(``gdb``) list main`<br>`1192        }`<br>`1193    }`<br>`1194`<br>`1195    int`<br>`1196    main (int argc, char **argv)`<br>`1197    {`<br>`1198      int i;`<br>`1199      struct pending *thispend;`<br>`1200      int n_files;`<br>`1201` |

#### 四、条件断点

条件断点是语法是：break  \[where\] if \[condition\]，这种断点真是非常管用。尤其是在一个循环或递归中，或是要监视某个变量。注意，这个设置是在GDB中的，只不过每经过那个断点时GDB会帮你检查一下条件是否满足。

#### 五、命令行参数

有时候，我们需要调试的程序需要有命令行参数，很多朋友都不知道怎么设置调试的程序的命令行参数。其实，有两种方法：

1. gdb命令行的 –args 参数
2. gdb环境中 set args命令。

#### 六、gdb的变量

有时候，在调试程序时，我们不单单只是查看运行时的变量，我们还可以直接设置程序中的变量，以模拟一些很难在测试中出现的情况，比较一些出错，或是switch的分支语句。使用set命令可以修改程序中的变量。

另外，你知道gdb中也可以有变量吗？就像shell一样，gdb中的变量以$开头，比如你想打印一个数组中的个个元素，你可以这样：

|     |     |
| --- | --- |
| 1<br>2<br>3<br>4<br>5 | `(``gdb``)` `set` `$i = 0`<br>`(``gdb``) p a[$i++]`<br>`...`  `#然后就一路回车下去了` |

当然，这里只是给一个示例，表示程序的变量和gdb的变量是可以交互的。

#### 七、x命令

也许，你很喜欢用p命令。所以，当你不知道变量名的时候，你可能会手足无措，因为p命令总是需要一个变量名的。x命令是用来查看内存的，在gdb中 “help x” 你可以查看其帮助。

* x/x 以十六进制输出
* x/d 以十进制输出
* x/c 以单字符输出
* x/i  反汇编 – 通常，我们会使用 `x/10i $ip-20 来查看当前的汇编（$ip是指令寄存器）`
* x/s 以字符串输出

#### 八、command命令

有一些朋友问我如何自动化调试。这里向大家介绍command命令，简单的理解一下，其就是把一组gdb的命令打包，有点像字处理软件的“宏”。下面是一个示例：

|     |     |
| --- | --- |
| 1<br>2<br>3<br>4<br>5<br>6<br>7<br>8<br>9<br>10 | `(``gdb``)` `break` `func`<br>`Breakpoint 1 at 0x3475678:` `file` `test``.c, line 12.`<br>`(``gdb``)` `command` `1`<br>`Type commands` `for` `when breakpoint 1 is hit, one per line.`<br>`End with a line saying just` `"end"``.`<br>`>print arg1`<br>`>print arg2`<br>`>print arg3`<br>`>end`<br>`(``gdb``)` |

当我们的断点到达时，自动执行command中的三个命令，把func的三个参数值打出来。

（全文完）

![[./_resources/GDB中应该知道的几个调试方法___酷_壳_-_CoolShell.resources/unknown_filename.jpeg]]
关注CoolShell微信公众账号可以在手机端搜索文章

**（转载本站文章请注明作者和出处 [酷 壳 – CoolShell](https://coolshell.cn/) ，请勿用于任何商业用途）**

——=== **访问 [酷壳404页面](http://coolshell.cn/404/) 寻找遗失儿童。** ===——
![[./_resources/GDB中应该知道的几个调试方法___酷_壳_-_CoolShell.resources/rating_on.gif]]![[./_resources/GDB中应该知道的几个调试方法___酷_壳_-_CoolShell.resources/rating_on.gif]]![[./_resources/GDB中应该知道的几个调试方法___酷_壳_-_CoolShell.resources/rating_on.gif]]![[./_resources/GDB中应该知道的几个调试方法___酷_壳_-_CoolShell.resources/rating_on.gif]]![[./_resources/GDB中应该知道的几个调试方法___酷_壳_-_CoolShell.resources/rating_half.gif]] (**15** 人打了分，平均分： **4.80** )
