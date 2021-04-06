---
title:  "java程序员 '不务正业' 的假期"
date:  2020-05-27 14:16:01 -0600
categories: shell
layout: post
---
 > 假期在家主要复习了一下设计模式、文件编码的知识。另外重点学习了一下bash脚本的知识。



*其实bash脚本的知识也不是一点也不会，工作中也写过一些shell。但是没有系统的学习过。每次遇到什么问题，就上网搜一下，面向谷歌编程而已，缺乏系统的认识，比如变量应该怎么定义，怎么访问，单引号和双引号的区别。清明节假期借这个机会系统学了一下。说是不务正业，其实只是个玩笑的说法，作为java程序员，或者说作为后端程序员，基本的bash命令，和shell脚本应该是掌握的。* 



**主要参考资料为阮一峰的bash教程和鸟哥的私房菜基础版**



##### 脚本分类

两个名词：bash和shell。先说shell，shell是外壳的意思，有外自然有内。与外壳对应的是“内核”，即我们学习操作系统的时候常听到的一个名词：linux内核，由于内核是受保护的，因此出现了外壳shell，意思是内核外面的外壳。

 bash：shell分类之一

shell 有很多种，我们常常执行命令的时候会输入 sh test.sh 。 这个sh 就是shell的一种。叫做Bourne  shell 。 Bourne   shell是Unix的，而bash 是GNU中的shell，叫做 Bourne Again  shell .



##### 注释

一般情况下脚本以

~~~shell
#!/bin/bash
~~~

开头。

‘   #  ’ 代表注释

#! 称作 Shebang 意思是告诉操作系统用哪种解释器来执行这个shell脚本。



##### 父子进程

* 每次打开一个shell就是一个进程

* 每次执行一个命令就是一个进程。

* 父进程定义的变量，在子进程中无法访问到。

因此如果有下面的脚本：

~~~shell
vi test.sh
~~~

内容如下：

~~~shell
#!/bin/bash
cd /opt/test
ls 
~~~

 ~~~shell
./test.sh
 ~~~

如果执行上面的脚本: ./test.sh  会发现目录并没有切换，原因是我们当前在一个终端中，即在一个shell 中，在这里执行 ./test.sh 的时候，其实打开了另外一个shell。因此这些命令是在另外的shell中执行的，当前是父进程，不会受到影响。 如果想要在当前shell中生效，则执行 source 命令即可。

~~~shell
source test.sh
~~~

或者

~~~shell
. test.sh
~~~

. 相当于 source命令的简写。

因此我们改了环境变量的时候， 都是执行 source  profile ，其实是让改动在当前shell中生效，而不需要退出再登录。



##### 单引号和双引号

* 被单引号包裹的内容，将原样输出。

* 被双引号包裹的内容，在解析时候会尝试先解析里面的变量，若其中存在变量，则先将变量解析为实际值。



##### 变量定义和取值

变量的定义跟java中不太一样，shell中不需要指定变量的类型： 

java :

~~~java
String uname = "tom";
~~~

shell:

~~~shell
uname='tom'
~~~

注意： **等号两边不能有空格**。我最初在写shell的时候，习惯了java的格式化，老是带空格。 另外双引号和单引号见上面的解释。 变量名可以是 大小写、字母、下划线。不能以字母开头。

读取一个字符串： 通过 $加变量名

~~~shell
echo $uname
~~~

这样就可以读取到上面这个变量。

还有另外一种方法：

~~~shell
echo ${uname}
~~~

第二种多用于变量名后面直接跟着其他字符的场景，比如定义一个变量作为文件夹的前缀:

~~~shell
fold_pre=tom
full_path=/home/${fold_pre}_video
~~~

如果不用花括号的话， 上面的 fold_pre_video 会被当做一个变量名来解析。



##### 读取键盘输入

使用 read 命令

常有一些需要交互的脚本，比如显示出符合条件的进程id，让用户确认是否停止这些pid:

~~~shell
echo '是否停止上述进程 y/n ?'
read stop_flag
echo $stop_flag
~~~

直接输入 read 变量 ，则当前shell 将阻塞等待输入数据。



##### 判断

if 配合中括号

~~~shell
#!/bin/bash
test_val='1'
if [ $test_val == '1' ]
        then
        echo 'succee'
fi
~~~

if 的格式：

 if 

  then 

fi

**中括号里面的内容，每一项前后都要有空格。**



##### 连续执行多个命令

方法有三种：

* 使用分号

~~~sh
cd /opt/test; ls
~~~

前后两个命令没关系，俩命令都会执行，无论第一个是否执行成功

* 使用 &&

~~~
cd /opt/test && ls
~~~

只有前一个执行成功时，后一个才会执行。 类似java中的 &&  且操作

* 使用 || 

~~~
cd /opt/test || ls 
~~~

只有前一个执行失败时，才会执行后一个。类似java中的 || 或操作





###### 参考资料

[阮一峰:bash脚本教程](https://wangdoc.com/bash/index.html) 