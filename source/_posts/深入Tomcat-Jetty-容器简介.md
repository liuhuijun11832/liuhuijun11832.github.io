---
title: 深入Tomcat/Jetty-servlet容器简介
categories: 学习笔记
date: 2019-07-18 14:00:00
tags:
- Jetty
- Tomcat
keywords: [Jetty,Tpmcat]
description: 深入学习Tomcat/Jetty等容器
---

# 学习路径
## 概念和基础
Tomcat或者Jetty："HTTP服务器+Servlet容器"，又称为WEB容器，本文基于Tomcat 9。

操作系统：《UNIX环境高级编程》

Java基础：面向对象设计概念，集合，I/O体系，多线程并发，网络模型（NIO，BIO，AIO），注解和反射

WEB基础：Spring框架开发经验。

<!-- more -->

## Http协议基础
HTTP本质：浏览器和服务器之间通信的内容格式，属于应用层协议;

http请求的组成：
1. 请求行
2. 请求头
3. 请求体

数据在网络中传输都是通过字节流，即8位二进制，tomcat能够把字节流转成Request对象，该对象封装了HTTP所有的请求信息，经过程序处理后又得到一个Response对象。

http响应的组成：
1. 状态行
2. 响应头
3. 响应体

cookie和session技术：
为了弥补HTTP协议无状态的特性而产生的技术，每次浏览器和服务器建立连接时都会产生一个session，并在Cookie中填充了sessionID用来标请求的身份。Tomcat后台会有线程定时轮询session，删除过期session。

在HTTP 1.1中引入了长连接，会在响应头中加入Connection:keep-alive，这样会使得每一次建立的TCP连接可以被HTTP请求复用。

## servlet规范和servlet容器
servlet实质上是一个Java中的接口。servlet规范=servlet接口+servlet容器。

servlet接口：

```java
public interface Servlet {
    //加载某个servlet的时候会调用该servlet实现类的init方法
    public void init(ServletConfig config) throws ServletException;
    //web.xml中配置的参数
    public ServletConfig getServletConfig();
    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException;
    public String getServletInfo();
    public void destroy();
}
```
service用于调用业务，其中的ServletRequest和ServletResponse就是对通信协议的封装。

ServletContext：每一个WEB应用都会有一个ServletContext。

WEB应用标准结构：

    | - MyWebApp
        | - WEB-INF/web.xml    --配置ServletConfig信息
        | - WEB-INF/classes    --servlet编译输出目录
        | - WEB-INF/lib        --web依赖的jar包存放
        | - META-INF/          --工程清单信息

Filter：web应用部署完成以后，会实例化所有Filter并组装成一个FilterChain；

Listener：当Servlet容器内发生某些指定事件时，Servlet容器会调用指定方法，比如Servlet容器启动时会触发Spring的ServletContext的监听器，使得Spring容器被创建并初始化。

> Spring MVC中，第一个请求进入到DispatcherServlet时，如果该类还未实例化，就会调用该类的init方法，创建Spring MVC容器，创建过程中还可以通过ServletContext拿到Spring的Ioc容器，并将Spring 容器设置为自己的父容器，Spring MVC能够访问Spring容器里的Bean，但是反过来是不行的。

## Tomcat日志
1. catalina.***.log：启动过程信息，包括JVM参数以及操作系统信息；
2. catalina.out：标准输出和标准错误；
3. localhost.**.log：初始化过程中遇到的未处理异常；
4. localhsot_access_log.**.txt：请求日志，包含IP地址、路径、时间、协议以及状态码等信息；
5. manager.`***`.log/host-manager.`***`.log：自带的manager项目的日志信息。