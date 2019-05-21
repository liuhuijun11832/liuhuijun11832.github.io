---
title: Linux基础-Shell分支循环和函数
categories: 编程技术
date: 2019-03-11 11:47:59
tags:
- Linux
- Shell
keywords: [Linux,Shell]
description: Shell循环和函数语法
---

## if

## 语法

单分支：

```bash
#第一种
if <条件表达式>
    then
        指令
fi
#第二种
if <条件表达式> then
    指令
fi﻿​
```

<!--more-->

条件表达式可以参考第二部分的内容，第二种的相当于用分号进行了换行。
双分支：

```bash
if <条件表达式>
    then
        指令1
else
        指令2
fi
```

[《Linux基础-Shell规范和执行 》](https://blog.guitar-coder.cn//Linux基础-Shell规范和执行.html
)的`[ -f "$file" ] && echo 1 || echo 0`就等于:

```bash
if [ if "$file" ]
    then
        echo 1
else
        echo 0
fi﻿​
```

多分支：

```bash
if <条件表达式1>
    then 
        指令1
elif <条件表达式2>
    then
        指令2
elif ......
fi﻿​
```

配置邮箱和密码：

```bash
#配置邮箱和用户名密码
[root@VM_0_8_centos MyBlog]# echo -e "set from=liuhuijun_2017@163.com smtp=smtp.163.com \nset smtp-auth-user=liuhuijun_2017@163.com smtp-auth-password=123456 smtp-auth=login" >> /etc/mail.rc
#输入测试字符串
[root@VM_0_8_centos MyBlog]# echo "Hello world" >> test.txt
#读取测试字符串并发送邮件
[root@VM_0_8_centos MyBlog]# mail -s "title" liuhuijun_2017@163.com < test.txt
```

扩展：
在服务器本地监控端口的命令有netstat、ss、lsof；远程监控服务器本地端口的命令有telnet、nmap、nc（可能需要安装）。

```bash
#查看3306端口的进程数
netstat -lnutp|grep 3306|wc -l
#查看mysql进程数
netstat -lnutp|grep mysql|wc -l
#ss类似
ss -lnutp | grep 3306|wc -l
#lsof用法
lsof -i tcp:3306| wc -l
#查看远端3306端口是否开通，过滤open关键字，返回1说明有该关键字，端口可通
nmap 127.0.0.1 -p 3306 | grep open|wc -l
#telnet需要过滤Connected关键字
echo -e "\n" | telnet 127.0.0.1 3306 2>/dev/null | grep Connected | wc -l
```

# 函数

## 语法

```bash
function name(){
    指令
    return n;
}
```

其中function可以省略不写.

* 函数的定义必须在要执行的程序前面定义或加载
* 调用函数时，直接使用函数名即可
* 函数执行时，会和调用它的脚本共用变量
* shell执行系统中各种程序的执行脚本为：系统别名-函数-系统命令-可执行文件
* return n是退出函数，exit n是退出shell并返回一个返回值给执行程序的当前shell
* 如果将函数存放在独立的文件中，被脚本加载时，需要使用source
* 带参数的函数：name 参数1 参数2

# case

```bash
case "变量" in
    值1)
        指令1 ...
        ;;
    值2)
        指令2 ...
        ;;
    *)
        指令3 ...
esac
```

注意缩进。

# while

```bash
while <条件表达式>
do
    指令...
done
```

拓展：让shell脚本在后台运行的方法：`sh 1.sh &`，ctrl+c：停止执行脚本或任务，ctrl+z：暂停执行脚本或任务，bg：将当前脚本或任务放到后台执行，bg可以理解为background，fg：将当前脚本放到前台执行，如果有多个任务，可以使用fg 加数字表示调出脚本任务，jobs：查看当前执行的脚本或任务，kill：关闭执行的脚本任务，即以kill % 任务编号的形式关闭脚本，这个任务编号可以通过jobs来获得。

使用场景：
    执行守护进程，以及实现我们希望循环执行不退出的应用，或者频率小于1分钟的循环处理。

# for和select

```bash
for 变量名  in  变量取值列表
do
    指令
done﻿​
```

如果"in 变量取值列表"省略，那么缺省为$@。

```bash
for((exp1: exp2; exp3))
do
    指令
done
```

使用场景：
    有限次的循环处理

```bash
select  变量名  [in 菜单取值列表]
do
    指令
done
```

如果"in 菜单取值列表"省却，那么缺省为$@。

# 循环控制

* break：如果后面有正整数，表示跳出循环的层数，如果没有则跳出整个循环
* continue：如果后面有正整数，表示跳出到第n层继续循环，如果没有则开始下一次循环
* exit：后面跟正整数表示退出shell的的返回值，在下一个shell里可以通过"$?"来获取返回值
* return：用于在函数中作为退出函数的返回值，在下一个shell里可以通过"$?"来获取返回值   

# 数组

## 语法

```bash
array=(1 2 3)##推荐使用
array=([1]=one [2]=two [3]=three)
array[1]=a;array[2]=b;array[3]=c
array=($(命令))或者array=(`命令`)
```

## 数组的输出

```bash
##打印第二个元素（下标从0开始）
echo ${array[1]}
##打印所有元素
echo ${array[*]}
echo ${array[@]}
##打印数组元素的个数
echo ${#array[@]}
echo ${#array[*]}
```

## 数组赋值

```bash
##添加或修改数组元素
array[3]=four
##删除数组中某个元素
unset ${array[1]}
##删除整个数组
unset array
```
