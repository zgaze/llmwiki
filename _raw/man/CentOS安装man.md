---
source: http://blog.csdn.net/gatieme/article/details/51656707
---
<http://blog.csdn.net/gatieme/article/details/51656707>

CentOS安装man
man手册

yum install man

man中文安装包

yum install manpages-zh

如果查不到manpages-zh中文包,则可以使用如下命令搜索

yum list |grep man.\*zh

man-pages-zh-CN.noarch 1.5.2-4.el7 @base

由此可以找到以上安装包，如果找不到，执行 yum -y update 更新安装包。

执行安装命令

sudo yum install man-pages-zh-CN.noarch

centos 7 安装man

yum install man-pages
