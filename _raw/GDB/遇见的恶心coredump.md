---
---
**使用空悬指针**

1、背景：每次coredump到了map的find和另一个函数，map的内存地址被修改（\_m\_left...)。开始排查
2、怀疑：被析构的对象又做了修改
3、突破口：debug那一块内存，发现跟我们常用的时间戳和类似。
4、发现问题：

* 一个请求会下发多个isearch\_root，有一个total\_num，process\_num，timeout\_num。
* 回调执行时，对应的num++。
* 然后调用common\_call，isskip判断是否所有请求都完成；如果不是return，如果是继续执行并delete handler。

问题出在：1、process\_num++，之后其他线程的判断就可以满足isskip，这个继续下去的线程就会delete handler，如果这时候，有线程还在操作handler，就可能core
