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

# 源码学习方法

经典开源框架和中间件：

1. 服务接入层：Nginx、Node.js；

2. 业务逻辑层：Tomcat、Jetty、Spring家族、Mybatis；

3. 数据缓存层：Redis、Kafka；

4. 数据存储层：Mysql、MondoDB、文件存储HDFS、搜索分析引擎ES。

   <!--more-->

通过问题找到解决办法。

网络通信要考虑的问题：

- IO模型，同步还是异步，阻塞还是非阻塞；
- 通信协议，二进制（gRPC）还是文本（HTTP）；
- 序列化方式，JSON还是Protocol Buffer。

服务端处理网络连接的过程：
accept-select-read-decode-process-encode-send

对应角色和职责：

- Acceptor-accept
- Selector-select
- Processor-read，decode，process，encode，send

组件：

- netty: 通过EventLoop将Selector和Processor跑在同一个线程，充分利用CPU缓存来侦测I/O事件和读写数据，同时可以设置业务处理和I/O处理的事件比率，超过这个比率就会将任务扔到专门的线程池来执行，Jetty的EatWhatYouKill线程也与之类似；
- kafka：Selector和Processor跑在不同的线程里，因为kafka的业务逻辑大多涉及与磁盘读写，处理时间不确定，这一点Tomcat与之类似；

学习Tomcat源码的过程：

1. 弄清楚中间件的核心功能是什么，比如Tomcat，核心功能是Http服务器和Servlet容器，怎么连接，怎么读取，怎么解析，怎么调用servlet，怎么处理业务等；
2. 核心架构，核心骨架类：
   - Server；
   - Service；
   - 连接器：流程Acceptor->SocketProcessor->Executor->Processor->Adapter，其中顶层为Connector，包含ProtocolHandler和Adapter，ProtocolHandler又包括EndPoint和Executor、Processor，而EndPoint包含了Acceptor和SocketProcessor；
   - 容器： Engine->Host->Context->Wrapper(Servlet)。

# 实战优化Tomcat启动速度

1. 删除webapps文件夹下不需要的工程；

2. server.xml文件尽量简洁；

3. 清理不需要的jar文件，对于servlet-api等的jar包，通常tomcat都已经提供，如果项目里需要，引入该jar包时将其作用域调整为`provided`；

4. 清理logs文件夹下不需要的日志文件，还有work下的Catalina，这个文件夹是Tomcat把jsp转换为class文件的工作目录；

5. 禁止TomcatTLD扫描，如果项目中没有用到jsp，或者不需要TLD，可以配置为禁止。`At least on jar was scanned for TLDs yet contained no TLDs.Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs`这种提示也表明tomcat推荐禁止TLD扫描。

   - 对于没有使用jsp的项目，可以直接禁止TLD扫描：在tomcat的conf/context.xml文件中，加如下配置：

   ```xml
   <Context>
       <JarScanner>
           <JarScanFilter defaultIdScan="false"/>
       </JarScanner>
   </Context>
   ```

   - 如果使用了Jsp，那么TLD扫描无法避免，但是可以在tomcat目录下的conf/catalina.properties里配置tomcat指定扫描的Jar：

   ```properties
   tomcat.util.scan.StandardJarScanFilter.jarsToSkip=xx.jar
   ```

6. 不需要websocket时关闭websocket支持，或者直接删除tomcat lib下的websocket-api.jar和tomcat-websocket.jar；

```xml
<Context containerSciFilter="org.apache.tomcat.websocket.server.WsSci">
</Context>
```

7. 不需要jsp时，关闭jsp功能；

```xml
<Context containerSciFilter="org.apache.jasper.servlet.JasperInitializer">
</Context>
```

8. 如果没有用到注解式servlet，可以在web应用的web.xml文件中，设置`<web-app metadata-complete="true">`告诉tomcat web.xml中配置的servlet时完整的，不需要再去jar包中扫描servlet；
9. 关闭web-fragment扫描；servlet 3.0引入了web-fragment.xml，这是一个部署描述文件，可以完成web.xml的配置功能，这个文件必须放置在jar的META-INF目录下，所以tomcat需要扫描所有jar包来扫描该文件，可以通过在项目里的web.xml加入`<web-app> ...<absolute-ordering /> ... </web-app>`让它不扫描;
10. 随机数熵源优化，tomcat 7以上的版本依赖Java的SecureRandom类来生成随机数，比如Session ID，而Jvm默认使用阻塞式熵源（/dev/random），容易导致启动速度变慢，可以设置Jvm参数`-Djava.security.egd=file:/dev/urandom`;
11. 允许并行启动多个web应用，因为默认情况下tomcat是一个一个启动程序的，可以配置conf/server.xml并行启动：

```xml
<Engine startStopThreads="0">
    <Host startStopThreads="0">
    ...
    </Host>
</Engine>
```