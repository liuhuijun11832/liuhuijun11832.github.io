---
title: 深入Tomcat/Jetty-双亲委派
categories: 学习笔记
date: 2019-07-22 11:31:56
tags:
- Tomcat
- Jetty
keywords: [Tomcat,Jetty]
description: 深入学习Tomcat和Jetty
---

# 类加载

概念：将.class文件加载到JVM**方法区**，并在**堆区**创建一个java.lang.Class的对象实例 ，作为具体业务类对象实例。

时期：需要用到时才加载。

Java实现：`abstract class：ClassLoader`，包含`private final ClassLoader parent`、`public loadClass(String name)`，`protected findClass(String name)`。

步骤：

如果父加载器不为空，委托给父加载器加载（递归），否则查找bootstrap加载器是否加载过，最后才会调用自己的`findClass`方法。

findClass：从本地文件系统或者网络中寻找.class文件读入到内存；将读入内存的字节数组转成Class对象。

JDK类加载器:

![深入Tomcat/Jetty-双亲委派/JDK-ClassLoader.png](深入Tomcat-Jetty-双亲委派/JDK-ClassLoader.png)

- BootstrapClassLoader:启动类加载器，由C语言实现，加载rt.jar，resources.jar等；
- ExtClassLoader:扩展类加载器，加载jre\lib\ext目录下的jar；
- AppClassLoader:系统类加载器，加载classpath下的类，也是默认的应用程序类加载器。
- 自定义：用于加载自定义自定义路径下的类。

这种指向不是继承关系，而是parent指向了另一个ClassLoader，所以如果想打破双亲委派的话，只需要新写一个ClassLoader继承ClassLoader类再重写loadClass和findClass。

# Tomcat类加载器

WebAppClassLoader：