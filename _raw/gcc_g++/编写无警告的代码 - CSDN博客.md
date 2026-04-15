---
source: https://blog.csdn.net/e5max/article/details/8753294
---
原

###### 编写无警告的代码

2013年04月02日 22:19:32
阅读数：5711

     今天把项目的Qt版本从Qt4.6.3升级到Qt4.8.4，重新编译项目代码的时候，特别关注了一下编译器的警告。于是找到《C++编程规范——101条规则、准则与最佳实践》翻了翻，重温了一下第1条 在高级别警告干净利落地进行编译。
     如果编译器对某个构造发出警告，一般表明代码中存有潜在的问题。警告就好比代码的“肿瘤”，可能是良性的也可能是恶性的——作为代码医生的我们不能对其视而不见。必须“把它弄清楚”，然后通过“改写代码以排除警告”！
  典型的编译器警告示例：
1、"Unused function parameter"（未使用的函数参数）:
编译器会对没有使用到的函数参数报告警告。如果确实不需要，那直接删除函数参数名就行了。

2、"Variable defined but never used"（定义了从未使用过的变量）:
如果确实不需要，可以直接将之删除。 如果该变量的实例化是为了启动某项服务，则经常可以通过插入一个变量本身的求值表达式，使编译器不再报警。（这种求值不会影响运行时的速度）

3、"Variable may be used without being initialized"（变量使用前可能未经初始化）:
这种警告通常意味着你的代码存在问题，请慎重处理之。

4、 "Missing return"（遗漏了return语句）。:
这可能是一种好现象。即使你认为控制永远都不会运行到结尾或某个分支，则应该加上执行assert(false ) 的语句。
像下面这样（兼有注释作用的断言）：
 default:  
     assert( !"should never get here!" );  // !"string" 的求值结果为false  

5、"signed/unsigned mismatch"（有符号数/无符号数不匹配）:
  通常需要插入一个显式的强制转换。

6、 "warning: type qualifiers ignored on function return type"（忽略了函数返回类型的类型限定符）:
通常是因为在函数前面的返回值面前添加了多余的 const 限定符。

例如，const int getAge() const { return m\_age; }  ，最前面的const 是多余的，编译器会报告警告。

7、 "warning: deprecated conversion from string constant to 'char\*'"（不推荐的转化用法）:
 函数原型：void setName(char \*name);

 函数调用：setName("personName");  //  编译器报告警告
释义：这是因为 char \* 是一个指针，其背后的含义是：给我一个字符串，我可以修改它。

而如果我们传给函数一个“字符串常量”，这应该是没法被修改的。所以说，比较合理的办法是把参数类型修改为const char \* ，而这个类型背后的含义是：给我一个字符串，我只要读取它（const意味着只读）。
8、 "warning: "MAX\_PATH" redefined" (重复定义) :
    #ifndef MAX\_PATH
    #define MAX\_PATH(260)
    #endif

9、第三方头文件。

10、"auto-importing has been activated without --enable-auto-import specified on the command line"

       当然，对纯属噪声的警告，如果一时无法提供消除的方法。应该单独禁用这个警告，并且编写一个清晰的注释，说明为什么要禁用。

文章标签： [读书](http://so.csdn.net/so/search/s.do?q=%E8%AF%BB%E4%B9%A6&t=blog)
个人分类： [C/C++](https://blog.csdn.net/e5max/article/category/1299375)
