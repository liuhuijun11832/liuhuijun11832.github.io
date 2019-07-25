---
title: 深入Tomcat/Jetty-内核知识和高性能之道
categories: 学习笔记
date: 2019-07-18 14:32:24
tags:
- Tomcat
- Jetty
keywords: [Tomcat,Jetty]
description:深入学习Tomcat和Jetty的笔记
---

# Linux-进程和线程
执行程序：程序文件加载到内存，然后CPU读取和执行；
进程：一次程序的执行过程；
关键字：每一个进程都有自己的task_struct;
内核：也是一个程序，启动时加载到内存；
内存分配：假如进程分配有4G虚拟内存，但是只有当访问到的虚拟内存没有分配物理内存时，才会产生缺页中断，此时内存管理单元（MMU）才会将虚拟内存和物理内存的映射关系保存在表中，再次访问虚拟内存就能读找到物理内存页。

<!--more-->

![LinuxMM.png](深入Tomcat-Jetty-内核知识和高性能之道/LinuxMM.png)

高位1G在内核空间，低位3G在用户空间。只有内核才可以直接访问各种资源，如果用户向访问，需要调用内核函数，比如`ssize_t read(int fd,void *buf,size_t nbyte)`就是用Socket的read函数。**栈向低位增长，堆向高位增长**。

其中的mmap映射区可以将文件内容映射到这片区域，由于Linux一切设备皆文件，所以I/O事件可以通过映射到mmap区来实现用户程序直接读取文件内容，而无需再调用系统函数，Java的MappedByteBuffer（DirectByteBuffer的父类）就是通过这种方式实现，同时用户程序调用的系统程序也是通过该区域来实现共享。

task_struct:包含vm_struct-保存了各个区域的内存起始和终止地址；还有类似进程号、打开的文件、创建的Socket以及CPU运行上下文等。

线程是一个CPU调度单元，因此线程有自己的task_struct和运行栈区，但是其他资源都是和进程共享的。



# 阻塞和唤醒

工作方式：内核-进程运行队列，当任务为Task_Running状态进入该队列，由CPU调度（**时间片轮转法**）。

进程运行队列：双向链表，task_struct-task_struct-task_struce

调度：从进程队列中选择一个进程，再从CPU列表中选择一个可用CPU，将进程上下文恢复到CPU中，(用户态

)执行上下文里的下一条指令；

阻塞：从进程队列里移除线程，添加到等待队列，置为Task_Uninterruptible或者Task_Interruptible，重新触发一次调度；

唤醒：添加到等待队列的同时，向内核注册一个回调函数。当数据到达（比如网卡）时，产生硬件中断，内核通过回调函数唤醒等待队列中的进程，添加到运行队列，置为Task_Running。**这个过程中，CPU(内核态)还会将内核空间里的网络数据复制到堆中(Buffer)**。

# Tomcat和Jetty中的对象池

Tomcat:SynchronizedStack类实现，类中的Object[] 实现栈的功能，同步的push和pop方法来归还对象和获得对象；

Jetty:ByteBufferPool，对象池，分配对象时从池里获得对象，原理：用不同的桶(bucket)管理不同bytebuffer，桶的内部用一个ConcurrentLinkDeque来放置ByteBuffer的引用。

注意点：

- 用完要归还；
- 再次使用要重置；
- 已经归还的对象不可以再做操作；
- 对池操作要考虑阻塞、异常、返回Null值的处理。

为什么要用对象池：http请求时间短，但是需要创建大量复杂对象并且创建对象的代价比较高（CPU负载高），或者限制部分资源的使用，比如连接池。

# Tomcat和Jetty的高效之道

1. 减少资源浪费；
2. 当某种资源称为瓶颈时，用其他资源来换取。

处理原则：

1. 连接由专门的Acceptor来做；
2. I/O事件侦测由Selector来做；
3. 协议解析交由线程池（Tomcat），或者交由Selector线程（Jetty）。

## 减少系统调用

channel带有缓冲区，尽量少地调用系统网卡的write，并且采用延迟解析，Tomcat只读取了request header，WEB应用程序才会读取到request body。

## 高效并发

1. 缩小锁范围，例如synchronized关键字，对对象加锁，而不对方法加锁；
2. 用原子变量和CAS取代锁；
3. 合理使用并发容器-CopyOnWriteArrayList、ConcurrenHashMap等；
4. 用volatile修饰当前组件的生命状态。

