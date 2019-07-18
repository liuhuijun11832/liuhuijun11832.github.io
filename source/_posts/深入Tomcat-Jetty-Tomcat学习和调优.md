---
title: 深入Tomcat/Jetty-Tomcat学习和调优
categories: 学习笔记
date: 2019-07-18 14:15:22
tags:
- Tomcat
- Jetty
keywords:[Tocmat,Jetty]
description:深入学习Tomcat和Jetty
---

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
5. 禁止TomcatTLD扫描，如果项目中没有用到jsp，或者不需要TLD，可以配置为禁止。At least on jar was scanned for TLDs yet contained no TLDs.Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs这种提示也表明tomcat推荐禁止TLD扫描。
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