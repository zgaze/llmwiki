---
source: http://www.cnblogs.com/wangkangluo1/archive/2012/06/06/2538149.html
---
# GDB 常用命令参考手册

[原文链接](http://wangcong.org/articles/learning-gdb.cn.html)

set scheduler-locking on 只调试当前线程

* * *

**GDB 命令行参数**
启动 GDB：

* gdb _executable_
* gdb -e _executable_ -c _core-file_
* gdb _executable_ -pid _process-id_ 
	（使用命令 'ps -auxw' 可以查看进程的 pid）
	

|     |     |
| --- | --- |
| 选项  | 含义  |
| \--help<br>\-h | 列出命令行参数。 |
| \--exec=_file_<br>\-e _file_ | 指定可执行文件。 |
| \--core=_core-file_<br>\-c _core-file_ | 指明 core 文件。 |
| \--command=_file_<br>\-x _file_ | 从指定文件中读取 gdb 命令。 |
| \--directory=_directory_<br>\-d _directory_ | 把指定目录加入到源文件搜索路径中。 |
| \--cd=_directory_ | 以指定目录作为当前路径来运行 gdb 。 |
| \--nx<br>\-n | 不要执行 .gdbinit 文件中的命令。默认情况下，这个文件中的命令会在所有命令行参数处理完后被执行。 |
| \--batch | 在非交互模式下运行 gdb 。从文件中读取命令，所以需要 -x 选项。 |
| \--symbols=_file_<br>\-s _file_ | 从指定文件中读取符号表。 |
| \-write | 允许对可执行文件和 core 文件进行写操作。 |
| \--quiet<br>\-q | 不要打印介绍和版权信息。 |
| \--tty=_device_ | 指定 _device_ 为运行程序的标准输入输出。 |
| \--pid=_process-id_<br>\-p _process-id_ | 指定要附属的进程 ID 。 |

* * *

**GDB 命令**
GDB 中使用的命令：

|     |     |
| --- | --- |
| 命令  | 描述  |
| help | 列出 gdb 帮助信息。 |
| help _topic_ | 列出相关话题中的 gdb 命令。 |
| help _command_ | 列出命令描述信息。 |
| apropos _search-word_ | 搜索相关的话题。 |
| info args<br>i args | 列出运行程序的命令行参数。 |
| info breakpoints | 列出断点。 |
| info break | 列出断点号。 |
| info break _breakpoint-number_ | 列出指定断点的信息。 |
| info watchpoints | 列出观察点。 |
| info registers | 列出使用的寄存器。 |
| info threads | 列出当前的线程。 |
| info set | 列出可以设置的选项。 |
| Break and Watch |     |
| break _funtion_<br>break _line-number_ | 在指定的函数，或者行号处设置断点。 |
| break +_offset_<br>break -_offset_ | 在当前停留的地方前面或后面的几行处设置断点。 |
| break _file:func_ | 在指定的_file_文件中的_func_处设置断点。 |
| break _file:nth_ | 在指定的_file_文件中的第_nth_行设置断点。 |
| break \*_address_ | 在指定的地址处设置断点。一般在没有源代码时使用。 |
| break _line-number_ if_condition_ | 如果条件满足，在指定位置设置断点。 |
| break _line_ thread_thread-number_ | 在指定的线程中中断。使用info threads可以显示线程号。 |
| tbreak | 设置临时的断点。中断一次后断点会被删除。 |
| watch _condition_ | 当条件满足时设置观察点。 |
| clear<br>clear _func_<br>clear _nth_ | 清除函数_func_处的断点。<br>清除第_nth_行处的断点。 |
| delete<br>d | 删除所有的断点或观察点。 |
| delete _breakpoint-number_<br>delete _range_ | 删除指定的断点，观察点。 |
| disable _breakpoint-number-or-range_<br>enable _breakpoint-number-or-range_ | 不删除断点，仅仅把它设置为无效，或有效。<br>例子：<br>显示断点： info break<br>设置无效： disable 2-9 |
| enable once _breakpoint-number_ | 设置指定断点有效，当到达断点时置为无效。 |
| enable del _breakpoint-number_ | 设置指定断点有效，当到达断点时删除它。 |
| finish | 继续执行到函数结束。 |
| Line Execution |     |
| step<br>s<br>step _number-of-steps-to-perform_ | 进入下一行代码的执行，会进入函数内部。 |
| next<br>n<br>next _number_ | 执行下一行代码。但不会进入函数内部。 |
| until<br>until _line-number_ | 继续运行直到到达指定行号，或者函数，地址等。 |
| return<br>return _expression_ | 弹出选中的栈帧（stack frame）。如果后面指定参数，则返回表达式的值。 |
| stepi<br>si<br>nexti<br>ni | 执行下一条汇编/CPU指令。 |
| info signals<br>info handle<br>handle _SIGNAL-NAME__option_ | 当收到信号时执行下列动作：nostop（不要停止程序），stop（停止程序执行），print（显示信号），noprint（不显示），pass/noignore（允许程序处理信号），nopass/ignore（不让程序接受信号） |
| where | 显示当前的行号和所处的函数。 |
| Program Stack |     |
| backtrace<br>bt<br>bt _inner-function-nesting-depth_<br>bt -_outer-function-nesting-depth_ | 显示当前堆栈的追踪，当前所在的函数。 |
| backtrace full | 打印所有局部变量的值。 |
| frame _number_<br>f _number_ | 选择指定的栈帧。 |
| up _number_<br>down _number_ | 向上或向下移动指定个数的栈帧。 |
| info frame _addr_ | 描述选中的栈帧。 |
| info args<br>info all-reg<br>info locals<br>info catch | 显示选中栈帧的参数，局部变量，异常处理函数。_all-reg_也会列出浮点寄存器。 |
| Source Code |     |
| list<br>l<br>list _line-number_<br>list _function_<br>list -<br>list _start#,end#_<br>list _filename:function_ | 列出相应的源代码。 |
| set listsize _count_<br>show listsize | 设置_list_命令打印源代码时的行数。 |
| directory _directory-name_<br>dir _directory-name_<br>show directories | 在源代码路径前添加指定的目录。 |
| directory | 当后面没有参数时，清除源代码目录。 |
| Examine Variables |     |
| print _variable_<br>p _variable_<br>p _file::variable_<br>p '_file_'::_variable_ | 打印指定变量的值。 |
| p \*_array-var_@_length_ | 打印_arrary-var_中的前_length_项。 |
| p/x _var_ | 以十六进制打印整数变量_var_。 |
| p/d _var_ | 把变量_var_当作有符号整数打印。 |
| p/u _var_ | 把变量_var_作为无符号整数打印。 |
| p/o _var_ | 把变量_var_作为八进制数打印。 |
| p/t _var_<br>x/b _address_<br>x/b &_variable_ | 以整数二进制的形式打印_var_变量的值。 |
| p/c _variable_ | 当字符打印。 |
| p/f _variable_ | 以浮点数格式打印变量_var_。 |
| p/a _variable_ | 打印十六进制形式的地址。 |
| x/w _address_<br>x/4b &_variable_ | 打印指定的地址，以四字节一组的方式。 |
| call _expression_ | 类似于_print_，但不打印 void 。 |
| disassem _addr_ | 对指定地址中的指令进行反汇编。 |
| Controlling GDB |     |
| set _gdb-option_ _value_ | 设置 GDB 的选项。 |
| set print array on<br>set print array off<br>show print array | 以可读形式打印数组。默认是 off 。 |
| set print array-indexes on<br>set print array-indexes off<br>show print array-indexes | 打印数组元素的下标。默认是 off 。 |
| set print pretty on<br>set print pretty off<br>show print pretty | 格式化打印 C 结构体的输出。 |
| set print union on<br>set print union off<br>show print union | 打印 C 中的联合体。默认是 on 。 |
| set print demangle on<br>set print demangle off<br>show print demangle | 控制 C++ 中名字的打印。默认是 on 。 |
| Working Files |     |
| info files<br>info share | 列出当前的文件，共享库。 |
| file _file_ | 把_file_当作调试的程序。如果没指定参数，丢弃。 |
| core _file_ | 把_file_当作 core 文件。如果没指定参数，则丢弃。 |
| exec _file_ | 把_file_当作执行程序。如果没指定参数，则丢弃。 |
| symbol _file_ | 从_file_中读取符号表。如果没指定参数，则丢弃。 |
| load _file_ | 动态链入_file_文件，并读取它的符号表。 |
| path _directory_ | 把目录_directory_加入到搜索可执行文件和符号文件的路径中。 |
| Start and Stop |     |
| run<br>r<br>run _command-line-arguments_<br>run < _infile_ > _outfile_ | 从头开始执行程序，也允许进行重定向。 |
| continue<br>c | 继续执行直到下一个断点或观察点。 |
| continue _number_ | 继续执行，但会忽略当前的断点_number_次。当断点在循环中时非常有用。 |
| kill | 停止程序执行。 |
| quit<br>q | 退出 GDB 调试器。 |

* * *

**GDB 操作提示**

* 在编译可执行文件时需要给 gcc 加上 "-g" 选项，这样它才会为生成的可执行文件加入额外的调试信息。
* 不要使用编译器的优化选项，比如： "-O"，"-O2"。因为编译器会为了优化而改变程序流程，那样不利于调试。
* 在 GDB 中执行 shell 命令可以使用：shell _command_
* GDB 命令可以使用 TAB 键来补全。按两次 TAB 键可以看到所有可能的匹配。
* GDB 命令缩写：例如 info bre 中的 bre 相当于 breakpoints。
* GDB 在 Emacs 中的操作：

|     |     |
| --- | --- |
| emacs 按键 | 动作  |
| M-x gdb | 切换到 gdb 模式。 |
| C-h m | 显示 gdb 模式介绍信息。 |
| M-s | 等同于gdb 中的 step 命令。 |
| M-n | 等同于gdb 中的 next 命令。 |
| M-i | 等同于gdb 中的 stepi 命令。 |
| C-c C-f | 等同于gdb 中的 finish 命令。 |
| M-c | 等同于gdb 中的 continue 命令。 |
| M-u | 等同于gdb 中的 up 命令。 |
| M-d | 等同于gdb 中的 down 命令。 |

* * *

**GDB 相关手册：**

* [gdb](http://node1.yo-linux.com/cgi-bin/man2html?cgi_command=gdb) - GNU debugger
* [ld](http://node1.yo-linux.com/cgi-bin/man2html?cgi_command=ld) - The GNU linker
* [gcc/g++](http://node1.yo-linux.com/cgi-bin/man2html?cgi_command=gcc) - GNU project C and C++ compiler
