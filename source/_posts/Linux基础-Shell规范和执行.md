---
title: Linux基础-Shell规范和执行
categories: 编程技术
date: 2019-03-11 11:14:16
tags:
- Linux
- Shell
keywords: [Linux,Shell]
description: Shell编程规范和执行
---
# Shell简述
shell是一个命令解释器，通过shell解释用户命令，并调用内核（kernel），由内核调用计算机资源（硬件）对命令响应。

    1.Bourne Shell
    包含Bourne shell（sh），Korn shell（kash），Bourne Again Shell（bash）三种类型。其中bash是各种发行版默认配置的shell。
    2.C shell
    包含csh，tcsh两种类型。tcsh是FreeBSD、Mac OSX等系统上的默认shell。

<!--more-->
centos下可以通过 `cat /etc/shells`查看系统支持的shell，通过`echo $SHELL`  查看默认的shell，
`cat /etc/redhat-release` 查看发行版本，`bash --version` 查看bash版本。

# shell规范和执行
## 基本规范
脚本第一行一般为：`#! /bin/bash` 或者 `#! /bin/sh` ，内核会根据#！幻数后的解释器来确定该用哪个程序解释这个脚本中的内容，如果不加默认就是bash。

    对于流程控制、分支等语句，最好先写好结构，再写内容。
    执行顺序：查找环境指令ENV，该变量指定了环境文件（加载顺序通常是/etc/profile、~/.bash_profile、~/.bashrc、/etc/bashrc等），在加载了上述环境变量文件后，shell开始从上至下，从左至右地执行每一行地命令及语句，如果遇到了子脚本，就会先执行子脚本的内容，然后再返回父脚本继续执行父脚本内后续的命令及语句。
    
设置crond任务时，最好能在定时任务脚本中重新定义系统环境变量，否则，一些系统环境变量可能不能被加载。

## 执行方式
* bash script.sh 或者 sh script.sh，启动新进程执行脚本；推荐使用。
* path/script.sh或者. /script.sh，启动新进程执行脚本；需要可执行权限。
* source script.sh或者. script.sh，在当前父进程中执行脚本，能够获取该脚本的变量值和函数返回值。 
* sh<script.sh或者cat script.sh script1.sh | sh，不常用。

## 定义环境变量
环境变量可以定义在/etc/profile、~/.bash_profile、~/.bashrc、/etc/bashrc四个文件中，加载顺序也是从前到后，类似于Java等的环境变量推荐写在/etc/profile里。

规范：


* 变量名通常大写
* 变量可以在自身shell以及子shell中使用
* 常用export来定义环境变量
* 使用env默认可以显示所有的环境变量名称以及对应的值
* 输出时使用"$变量名"，取消时使用"unset 变量名"
* 书写crond定时任务时要注意，脚本要用到的环境变量最好先再所执行的shell脚本中重新定义
* 如果希望环境变量永久生效，则可以将其放在用户环境变量文件或全局环境变量文件里


方式：

1. export variable=value
1. variable=value;export variable
1. declare -x variable=value
echo或者printf命令可以打印环境变量：

`echo $HOME` 或者 `printf "$HOME\n"`
env或者set命令可以显示当前的环境变量。

## 定义本地变量

定义在当前shell生存期的脚本中使用。

方式：

1. variable=value
1. variable="value"
1. variable='value'
1. variable=$(ls)
1. variable=\`ls\`
1. variable=${variable}

举个栗子:

当前处在root用户家目录下：/root:

* key=1 $(pwd) ==>key=1 /root
* key="1 $(pwd)"==>key=1 /root
* key='1 $(pwd)'==>key=1 $(pwd)
* key=1 `pwd`==>key=1 /root


总结：

* 若变量为连续的数字或字符串，推荐用第一或第二种
* 若变量内容包含空格、其他变量，推荐用第二种方式和第六种，若取其他变量推荐第六种
* 若希望原样输出字符串，推荐使用第三种
* 若将一个命令赋值给一个变量，可以使用第四、第五种，推荐使用第四种


但是注意在用awk的时候，以上规则是不能通用的， 对于awk命令，推荐使用`awk 'BEGIN {print "'$VARIABLE'"}'`，即在要取的变量外面加上单引号再加双引号的方式，这样无论变量是以上哪种方式都能保证结果的正确性，或者使用`echo str | awk '{print $0}`'的方式，因为echo打印规则同上面总结的是一样的。

## 特殊变量
* $0：获取当前执行的shell脚本的文件名，如果执行脚本包含了路径，那么也会包括路径
* $n：n为正整数，如果n是大于9的数，那么需要用{}包起来，代表执行脚本时传入的第n个参数
* $#：获取当前shell脚本的所有参数数量
* $*：获取所有参数，如果加上双引号，例如"$*"="$1 $2 $3……"
* $@：获取所有参数，不加双引号同上，如果加上双引号时，例如"$@"="$1" "$2" "$3"……，它会将每一个参数视为独立参数，并且保留参数之间的空格
* $?：获取上一个指令的执行状态返回值，0成功，非零失败
* $$：获取当前执行shell脚本的进程号，不常用
* $!：获取上一个在后台工作的进程的进程号，不常用
* $_：获取在此之前执行的命令或者脚本的最后一个参数，不常用

> 详情参考：man bash，然后按/，输入Special Parameters即可。

**关于$?：可能是上一次命令执行结果；可能是脚本exit 后面返回的结果；可能是函数return 后返回的结果。**

## 内置变量命令
### echo

`echo args`
功能：打印，args可以是字符串变量和变量的组合。

选项：

* -n：不换行输出内容
* -e：解析转义字符如\n（换行）、\r（回车） 、\t（制表）、\b（退格）、\v（纵向制表符）


### eval

`eval args`功能：当Shell程序执行到eval语句时，shell读入参数args，并将他们组合成一个新的命令，然后执行。
例如：eval.sh里的内容为

```bash
[root@localhost ~]# cat noeval.sh 
echo \$$#﻿​
```

执行下面代码，结果为$2：

```bash
[root@localhost ~]# sh noeval.sh 1 2
$2﻿
```

\\$会直接输出$，而$#取的是执行脚本传入的参数数量。所以最终结果为$2。
如果使用了eval命令，则会有不同结果：

```bash
[root@localhost ~]# cat eval.sh 
eval "echo \$$#"﻿​
```

此时结果为：

```bash
[root@localhost ~]# sh eval.sh 1 2 
2﻿​
```

在上面的基础上，eval会重新解析echo $2，因此打印了第二个参数。  

### exec

`exec 其他命令`
功能：在不创建新的子进程的前提下，转取执行指定命令，指定命令执行完毕后，该进程就终止了

示例：

```bash
[root@localhost ~]# seq 5 > tmp.log
[root@localhost ~]# cat exec.sh 
#读取tmp.logs
exec < tmp.log
while read line
do
 echo $line
done
echo ok﻿​
```

执行：

```bash
[root@localhost ~]# sh exec.sh 
1
2
3
4
5
ok﻿​
```

### read

`read 变量名表`
功能：从标准输入读取字符串等信息，传跟shell程序内部定义的变量

shift

`shift-Shift positional parameters`
功能：左移参数，即$2会变成$1，$3会变成$2，同时$#也会减1，而$1会消失。

示例：

```bash
[root@localhost ~]# cat n.sh 
 echo $1 $2
if [ $# -eq 2 ];then
 shift
 echo $1
fi﻿​
```
 
执行：

```bash
[root@localhost ~]# sh n.sh 1 2
1 2
2﻿​
```

### exit

`exit-Exit the shell`功能：退出shell程序。在exit之后可以有选择地指定一个数作为返回状态。

### 特殊扩展变量
`${parameter:-word}`：如果parameter的变量值为空或未赋值，则会返回word字符串并替代变量的值，但是parameter依然是未赋值。用途：如果变量未定义，则返回缺省的值。

```bash
[root@localhost ~]# echo $test
[root@localhost ~]# result=${test:-unset}
[root@localhost ~]# echo $result
unset﻿​
```
`${parameter:=word}`：如果parameter的变量值为空或者未赋值，并返回其值。用途：基本同上一个，但是还会给parameter赋值。

```bash
[root@localhost ~]# echo $test
[root@localhost ~]# result=${test:=unset}
[root@localhost ~]# echo $result 
unset
[root@localhost ~]# echo $test 
unset﻿​
```

`${parameter:?word}`：如果parameter的变量值为空或者未赋值，那么word字符串将被作为标准错误输出，否则输出变量的值。用途：用于捕捉
由于变量未定义而导致的错误，并退出程序。

```bash
[root@localhost ~]# echo $test
[root@localhost ~]# result=${test:?unset}
-bash: test: unset﻿​
```

${parameter:+word}：如果parameter的变量值为空或者未赋值，则什么都不做，否则word字符串将替代变量的值。

```bash
[root@localhost ~]# echo $test
test
[root@localhost ~]# result=${test:+unset}
[root@localhost ~]# echo $test 
test
[root@localhost ~]# echo $result 
unset
```

# 变量运算
运算符：


|算术运算符|	意义|
|---|---|
|+,-	|加法、减法|
|*,/,%|乘、除、取余（取模）|
|**|	幂运算|
|++,--|	自增、自减|
|<code>!,&&,||</code> |逻辑非（取反）、逻辑与（and）、逻辑或（or）|
|<,<=,>,>=	|比较符号|
|==,!=,=	|比较符号|
|<<,>>|	移位运算|
|<code>~,|,&,^</code>	|按位取反、按位或、按位与、按位异或|
|=,+=,-=,*=,/=,%=|	赋值运算 a+=1相当于a=a+1|

运算命令：


|运算操作符与运算命令|	意义|
|---|---|
|(())|	用于整数运算的常用运算符，效率很高|
|let	|用于整数运算，类似于(())|
|expr	|可用于整数运算，但还有很多其他额外功能|
|bc|	Linux下的一个计算器程序，适合整数以及小数运算|
|$[]|	用于整数运算|
|awk|	可以用于整数运算和小数运算|
|declare	|定义变量值和属性，-i参数可以用于定义整型变量，做运算|

示例：

```bash
##(())双括号运算操作
[root@localhost ~]# echo $((1+1))
2
[root@localhost ~]# echo $((6-43))
-37
[root@localhost ~]# ((i=5))
[root@localhost ~]# ((i=i*2))
[root@localhost ~]# echo $i
10
[root@localhost ~]# ((a=1+2**3-4%3))
[root@localhost ~]# echo $a
8
[root@localhost ~]# b=$((a=1+2**3-4%3))
[root@localhost ~]# echo $b
8
[root@localhost ~]# echo $((1+2**3-4%3))
8
[root@localhost ~]# echo $((100*(1+100)/2))
5050
##比较操作1为真0为假，只适用整数
[root@localhost ~]# echo $((3<8))
1
[root@localhost ~]# echo $((3>8))
0
[root@localhost ~]# a=8
[root@localhost ~]# echo $((a++))
8
[root@localhost ~]# echo $((++a))
10
##let运算操作基本和(())相同：
[root@localhost ~]# i=2
[root@localhost ~]# i=i+8
[root@localhost ~]# echo $i
i+8
[root@localhost ~]# unset i
[root@localhost ~]# i=2
[root@localhost ~]# let i=i+8
[root@localhost ~]# echo $i
10﻿
##expr操作
[root@localhost ~]# expr 2+2
2+2
[root@localhost ~]# expr 2 + 2
4
[root@localhost ~]# expr 2 - 2
0
[root@localhost ~]# expr 2 \* 2
4
[root@localhost ~]# expr 2 / 2 
1
[root@localhost ~]# i=5
[root@localhost ~]# i=`expr $i + 6` 
[root@localhost ~]# echo $i
11
## 判断运算一个变量是否为整数，结果为0则是，不为0则不是
[root@localhost ~]# i=5
[root@localhost ~]# expr $i + 6 &> /dev/null
[root@localhost ~]# echo $?
0
[root@localhost ~]# i=aaa
[root@localhost ~]# expr $ + 6 &>/dev/null
[root@localhost ~]# echo $?
2
##判断文件类型是否为指定类型
[root@localhost ~]# vim expr1.sh
[root@localhost ~]# cat expr1.sh 
#！ /bin/sh
if expr "$1" : ".*\.pub" &> /dev/null
        then
                echo "you are using $1"
        else
                echo "pls use *.pub file"
fi
[root@localhost ~]# sh expr1.sh 1.pub
you are using 1.pub
[root@localhost ~]# sh expr1.sh 1.pu
pls use *.pub file
[root@localhost ~]# char="i am oldboy"
##统计字符串长度
[root@localhost ~]# expr length "$char"
11
[root@localhost ~]# echo ${char}|wc -L
11
[root@localhost ~]# echo ${char}|awk '{print length($0)}'
11
##打印I am oldboy linux welcome to our traning中字符数不大于6的单词
[root@localhost ~]# vim word_length.sh
[root@localhost ~]# cat word_length.sh 
for n in I am oldboy linux welcome to our training
do 
 if [ `expr length $n` -le 6 ] 
 then
 echo $n
 fi
done﻿​
[root@localhost ~]# sh word_length.sh 
I
am
oldboy
linux
to
our

##awk运算操作
[root@localhost ~]# echo "7.7 3.8" | awk '{print $1-$2}'
3.9
[root@localhost ~]# echo "358 113" | awk '{print ($1-3)/$2}'
3.14159
[root@localhost ~]# echo "3.9" | awk '{print ($1+3)*$2}'
0
[root@localhost ~]# echo "3 9" | awk '{print ($1+3)*$2}'  
54
##declare（同typeset）运算操作
## -i可以将变量定义为整型
[root@localhost ~]# declare -i A=30 B=7
[root@localhost ~]# A=A+B
[root@localhost ~]# echo $A
37
[root@localhost ~]# 
##可以用read命令从标准输入中获得变量值。
##read [参数] [变量名]
##常用参数：-p：设置提示信息；-t：设置输入等待时间，单位默认为秒
[root@localhost ~]# read -t 10 -p "please input one num:" num
please input one num:17
[root@localhost ~]# read -t 10 -p "please input two num:" a1 a2
please input two num:1 2
[root@localhost ~]# echo $a1
1
[root@localhost ~]# echo $a2
2
[root@localhost ~]# cat read.sh 
echo -n "please input two number:"
read a1 a2
[root@localhost ~]# sh read.sh 
please input two number:1 2
##下面那句作用与上面的脚本相等
[root@localhost ~]# read -t 5 -p "please input two number:" a1 a2
please input two number:1 2﻿​
```
# 条件测试
## 综述

|条件测试语法|	说明|
|---|---|
|语法1：test <测试表达式>|	test和后面的<>之间有一个空格|
|语法2：[ <测试表达式> ]	|和test类似，[]和<测试表达式>之间至少有一个空格，推荐|
|语法3：[[ <测试表达式> ]]	|和[]类似，与<测试表达式>之间至少有一个空格|
|语法4：((<测试表达式>))	|括号和内容之间无需空格，一般用于if语句里|
举例：

```bash
#测试nginx压缩包是否存在并且是文件，如果是打印true，不是打印false
[root@VM_0_8_centos ~]# test -f nginx-1.14.1.tar.gz && echo true || echo false
true
[root@VM_0_8_centos ~]# test -d nginx-1.14.1.tar.gz && echo true || echo false 
false
[root@VM_0_8_centos ~]# test -d nginx-1.14.1 && echo true || echo false       
true
```

详情可以查看`man test`获得帮助。

```bash
#命令类似于test 为true时执&&后的命令，为false时执行||后的命令
[root@VM_0_8_centos ~]# [ -f nginx-1.14.1.tar.gz ] && echo 1
1
#因为前面判断为true。所以|| 后面的指令得不到执行，所以没有输出任何结果
[root@VM_0_8_centos ~]# [ -f nginx-1.14.1.tar.gz ] || echo 0  
[root@VM_0_8_centos ~]# [ -f nginx-1.14.1 ] || echo 0       
0
```

测试变量：

```bash
[root@VM_0_8_centos ~]# file=nginx-1.14.1.tar.gz 
[root@VM_0_8_centos ~]# [ -f "$file" ] && echo 1
1
[root@VM_0_8_centos ~]# [ -f "$file" ] || echo 0
```

例如生产环境下：系统NFS启动脚本测试：
`[ -f /etc/sysconfig/network ] && . /etc/sysconfig/network`

```bash
[ 条件1 ] && {
    命令1
    命令2
    命令3
}
#这两个等价
if [ 条件1 ]
    then
        命令1
        命令2
        命令3
fi
```

## 字符串测试操作符
两个字符串是否相同、测试字符串长度是否等于0、字符串是否为null等。

|常用操作符	|说明|
|---|---|
|-n "字符串"|	长度不为0则true，成立|
|-z "字符串"	|长度为0，成立|
|"串1" = "串2"|	等价于"串1"=="串2"，比较是否相等|
|"串1"！="串2"	|等价于"串1"！=="串2"，比较是否不想等|


**字符串测试一定要把字符串加双引号之后比
并且比较符号两端必须有空格**

## 整数二元比较


|test和[]中的符号	|在(())和[[]]中使用的符号|	说明|
|---|---|---|
|-eq	|==或=	|=也可以在test和[]中用|
|-ne	|!=	|!=也可以在test和[]中用|
|-gt	|>	|如果>在test或者[]中用需要转义\
|-ge	|>=| 用法大体同gt	|
|-lt	|<	|用法大体同gt|
|-le	|<=| 用法大体同gt|
**比较符号两端要有空格并且尽量不要混用，例如[]中使用<或[[]]中使用-ge，虽然结果可能是正确的，但是不建议**
	
