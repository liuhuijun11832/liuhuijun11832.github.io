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

![LinuxMM.png](深入Tomcat-Jetty-内核知识和高性能之道/LinuxMM.png)

高位1G在内核空间，低位3G在用户空间。只有内核才可以直接访问各种资源，如果用户向访问，需要调用内核函数，比如`ssize_t read(int fd,void *buf,size_t nbyte)`就是用Socket的read函数。**栈向低位增长，堆向高位增长**。

其中的mmap映射区可以将文件内容映射到这片区域，由于Linux一切设备皆文件，所以I/O事件可以通过映射到mmap区来实现用户程序直接读取，而无需再调用系统函数，Java的MappedByteBuffer（DirectByteBuffer的父类）就是通过这种方式实现，同时用户程序调用的系统程序也是通过该区域来实现共享。