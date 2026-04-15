---
source: http://www.cnblogs.com/ggjucheng/archive/2011/12/14/2287738.html
---
#### 1简介

GCC 的意思也只是 GNU C Compiler 而已。经过了这么多年的发展，GCC 已经不仅仅能支持 C 语言；它现在还支持 Ada 语言、C++ 语言、Java 语言、Objective C 语言、Pascal 语言、COBOL语言，以及支持函数式编程和逻辑编程的 Mercury 语言，等等。而 GCC 也不再单只是 GNU C 语言编译器的意思了，而是变成了 GNU Compiler Collection 也即是 GNU 编译器家族的意思了。另一方面，说到 GCC 对于操作系统平台及硬件平台支持，概括起来就是一句话：无所不在。

#### 2简单编译

示例程序如下：

//test.c
#include <stdio.h>
int main(void)
{
    printf("Hello World!\\n");
    return 0;
}

这个程序，一步到位的编译指令是:

gcc test.c -o test

实质上，上述编译过程是分为四个阶段进行的，即预处理(也称预编译，Preprocessing)、编译(Compilation)、汇编 (Assembly)和连接(Linking)。

##### 

 

##### 

##### 

##### 

 

#### 

#### 

 

####
