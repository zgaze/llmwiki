---
---
~~[Edit](http://maxiang.info/#/?provider=evernote&guid=e164679d-229b-4460-9e50-6890ffea3721&notebook=C%2B%2BWarning)~~

# C++ Warning 常规篇

## 类

* \= 号很多时候执行的是拷贝初始化而不是赋值

* switch 语句 case标签必须是整形常量表达式，case标签匹配成功，会向下执行所有的语句，所以需要在适当的地方break；`尽量在case 和 break之间使用{}`

## 基础语法

### 字符串

    char *p="hello“； 
    以这种方法初始化的字符串是常量字符串，所以不能修改。
    
    char p[]="hello“;
    这是变量，可以修改！
    char *p=new char[6];
    p="hello“； 
    这也是变量，可以修改！
    

### 源文件

* `头文件不应包含using声明，因为头文件的内容会拷贝到所有引用它的文件中去。`

## templete

* 模板类的实现妄图和定义分开，死活编译不过。

%23%20C++%20Warning%20%u5E38%u89C4%u7BC7%0A@%28C++Warning%29%0A%0A%0A%0A%23%23%20%u7C7B%0A-%20%3D%20%u53F7%u5F88%u591A%u65F6%u5019%u6267%u884C%u7684%u662F%u62F7%u8D1D%u521D%u59CB%u5316%u800C%u4E0D%u662F%u8D4B%u503C%0A-%20switch%20%u8BED%u53E5%20case%u6807%u7B7E%u5FC5%u987B%u662F%u6574%u5F62%u5E38%u91CF%u8868%u8FBE%u5F0F%uFF0Ccase%u6807%u7B7E%u5339%u914D%u6210%u529F%uFF0C%u4F1A%u5411%u4E0B%u6267%u884C%u6240%u6709%u7684%u8BED%u53E5%uFF0C%u6240%u4EE5%u9700%u8981%u5728%u9002%u5F53%u7684%u5730%u65B9break%uFF1B%60%u5C3D%u91CF%u5728case%20%u548C%20break%u4E4B%u95F4%u4F7F%u7528%7B%7D%60%0A%23%23%20%u57FA%u7840%u8BED%u6CD5%0A%23%23%23%20%u5B57%u7B26%u4E32%0A%60%60%60bash%0Achar%20\*p%3D%22hello%u201C%uFF1B%20%0A%u4EE5%u8FD9%u79CD%u65B9%u6CD5%u521D%u59CB%u5316%u7684%u5B57%u7B26%u4E32%u662F%u5E38%u91CF%u5B57%u7B26%u4E32%uFF0C%u6240%u4EE5%u4E0D%u80FD%u4FEE%u6539%u3002%0A%0Achar%20p%5B%5D%3D%22hello%u201C%3B%0A%u8FD9%u662F%u53D8%u91CF%uFF0C%u53EF%u4EE5%u4FEE%u6539%uFF01%0Achar%20\*p%3Dnew%20char%5B6%5D%3B%0Ap%3D%22hello%u201C%uFF1B%20%0A%u8FD9%u4E5F%u662F%u53D8%u91CF%uFF0C%u53EF%u4EE5%u4FEE%u6539%uFF01%0A%60%60%60%0A%23%23%23%20%u6E90%u6587%u4EF6%0A-%20%60%u5934%u6587%u4EF6%u4E0D%u5E94%u5305%u542Busing%u58F0%u660E%uFF0C%u56E0%u4E3A%u5934%u6587%u4EF6%u7684%u5185%u5BB9%u4F1A%u62F7%u8D1D%u5230%u6240%u6709%u5F15%u7528%u5B83%u7684%u6587%u4EF6%u4E2D%u53BB%u3002%60%0A%0A%23%23%20templete%0A-%20%u6A21%u677F%u7C7B%u7684%u5B9E%u73B0%u5984%u56FE%u548C%u5B9A%u4E49%u5206%u5F00%uFF0C%u6B7B%u6D3B%u7F16%u8BD1%u4E0D%u8FC7%u3002
