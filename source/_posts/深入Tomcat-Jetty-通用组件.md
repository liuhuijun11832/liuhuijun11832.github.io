---
title: 深入Tomcat/Jetty-通用组件
categories: 学习笔记
date: 2019-07-25 14:37:41
tags:
- Tomcat
- Jetty
keywords: [Tomcat,Jetty]
description: 深入学习Tomcat/Jetty的笔记
---

# 日志框架

slf4j：日志接口和门面，记录日志；

JCL：commons logging，和slf4j类似的门面日志；

log4j：第三方日志输出的具体实现；

JUL：Java原生的日志输出；

logback：第三方日志输出，性能由于log4j，也是Spring Boot默认日志框架；

log4j2：第三方日志输出的具体实现，号称目前Java平台性能最好。

<!--more-->

slf4j结构如下：

![Slf4j-Structure.png](深入Tomcat-Jetty-通用组件/Slf4j-Structure.png)

## Java 日志框架

路径：`java.util.logging`；

* Logger：记录日志；
* Handler：日志输出格式；
* Level：日志的不同等级；
* Formatter：日志信息格式化；

使用示例：

```java
public static void main(String[] args) {
  Logger logger = Logger.getLogger("com.mycompany.myapp");
  logger.setLevel(Level.FINE);
  logger.setUseParentHandlers(false);
  Handler hd = new ConsoleHandler();
  hd.setLevel(Level.FINE);
  logger.addHandler(hd);
  logger.info("start log"); 
}
```

## Tomcat 日志框架

JULI：基于JCL和JUL的处理框架。JVM原生的日志框架是每个JVM用同一份日志配置，但是Tomcat中每个Web应用可能有自己的日志框架；

DirectJDKLog：基于JUL中的Logger类，修改了默认输出格式；

LogFactory：单例，默认使用DirectJDKLog，通过ServiceLoader为Log提供自定义的实现版本；

Handler：

* FileHandler：读写锁实现，在某个特定位置往文件里输出日志；
* AsyncFileHandler：继承自FileHandler，实现了异步写操作，缓存存储通过LinkedBlockingQueue来实现，通过publish方法写入相应文件内。

Formatter：通过format方法将日志记录LogRecord转化成格式化的字符串，JULI提供了三个新的Formatter：

* OnlineFormatter：基本类似于JDK的SimpleFormatter，只不过把所有内容写到了同一行。
* VerbatimFormatter：只记录日志信息，没有任何额外信息；
* JdkLoggerFormatter：格式化了一个轻量级的日志信息。

配置文件：conf/logging.properties；以`1catalina.org.apache.juli.AsyncFileHandler`为例：数字是为了区分同一个类的不同实例；catalina、localhost、manager和host-manager是Tomcat用来区分不同系统日志类的标志，后面的字符串表示了handler具体类型，如果需要添加Tomcat服务器的自定义Handler，需要在字符串里添加。接下来是日志等级，目录和文件前缀等。

```properties
1catalina.org.apache.juli.AsyncFileHandler.level = FINE
1catalina.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
1catalina.org.apache.juli.AsyncFileHandler.prefix = catalina.
1catalina.org.apache.juli.AsyncFileHandler.maxDays = 90
1catalina.org.apache.juli.AsyncFileHandler.encoding = UTF-8
```

## Tomcat+Slf4j+Logback

* 去下该地址下载适合自己版本的Tomcat：https://github.com/tomcat-slf4j-logback/tomcat-slf4j-logback/releases/；
* 解压以后分别用bin、conf、lib下的内容替换或者复制到自己原有的Tomcat对应目录里；
* 启动Tomcat，可以看到日志格式已经变了：

```verilog
18:06:17.595 INFO  {main} [o.a.c.h.Http11NioProtocol] : Initializing ProtocolHandler ["http-nio-8080"]
18:06:17.996 INFO  {main} [o.a.t.u.n.NioSelectorPool] : Using a shared selector for servlet write/read
18:06:18.010 INFO  {main} [o.a.c.a.AjpNioProtocol] : Initializing ProtocolHandler ["ajp-nio-8009"]
18:06:18.018 INFO  {main} [o.a.t.u.n.NioSelectorPool] : Using a shared selector for servlet write/read
18:06:32.216 INFO  {main} [o.a.j.s.TldScanner] : At least one JAR was scanned for TLDs yet contained no TLDs. Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs were found in them. Skipping unneeded JARs during scanning can improve startup time and JSP compilation time.
18:06:33.843 INFO  {main} [o.a.c.h.Http11NioProtocol] : Starting ProtocolHandler ["http-nio-8080"]
18:06:33.860 INFO  {main} [o.a.c.a.AjpNioProtocol] : Starting ProtocolHandler ["ajp-nio-8009"]
```

# Session管理

## Session的创建

Context中的`interface Manager`，默认实现类是`StandardManager`。

主要API：

* load：持久化；
* unload：从磁盘加载；
* getSession：获取该次请求的session，如果参数为ture，不存在时会新建；
* `class Request`：Tomcat实现了HttpServletRequest的类；
* `class RequestFacade`：具体拿到的类，为了保证安全在Request上做的包装；

Request持有context，context持有manager，manager创建session。

Tomcat中Session的具体实现类是StandardSession，`StandardSession implements javax.servlet.http.HttpSession,org.apache.catalina.Session`，所有创建的session都在一个名为sessions的ConcurrentHashMap中。

## Session的清理

StandardContext调用StandardManager的后台backgroundProcess完成session的清理。*每隔10s启动一次，backgroundProcess调用6（对6进行了取模）次才会执行一次session清理。*

## Session事件通知

servlet：Session生命周期过程中，要将事件通知监听者：`interface HttpSessionListener extends EventListener`。

通过StandardContext中的集合取出HttpSessionListener类型的监听器，依次调用它们的sessionCreated方法或者sessionDestroyed方法。

# 集群通信

需要将server.xml中集群的一行注释打开：

```xml
 <!--
      <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>
      -->
```

该配置等同于以下：

```xml
<!-- 
  SimpleTcpCluster 是用来复制 Session 的组件。复制 Session 有同步和异步两种方式：
  同步模式下，向浏览器的发送响应数据前，需要先将 Session 拷贝到其他节点完；
  异步模式下，无需等待 Session 拷贝完成就可响应。异步模式更高效，但是同步模式
  可靠性更高。
  同步异步模式由 channelSendOptions 参数控制，默认值是 8，为异步模式；4 是同步模式。
  在异步模式下，可以通过加上 " 拷贝确认 "（Acknowledge）来提高可靠性，此时
  channelSendOptions 设为 10
-->
<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"
                 channelSendOptions="8">
   <!--
    Manager 决定如何管理集群的 Session 信息。
    Tomcat 提供了两种 Manager：BackupManager 和 DeltaManager。
    BackupManager－集群下的某一节点的 Session，将复制到一个备份节点。
    DeltaManager－ 集群下某一节点的 Session，将复制到所有其他节点。
    DeltaManager 是 Tomcat 默认的集群 Manager。
    
    expireSessionsOnShutdown－设置为 true 时，一个节点关闭时，
    将导致集群下的所有 Session 失效
    notifyListenersOnReplication－集群下节点间的 Session 复制、
    删除操作，是否通知 session listeners
    
    maxInactiveInterval－集群下 Session 的有效时间 (单位:s)。
    maxInactiveInterval 内未活动的 Session，将被 Tomcat 回收。
    默认值为 1800(30min)
  -->
  <Manager className="org.apache.catalina.ha.session.DeltaManager"
                   expireSessionsOnShutdown="false"
                   notifyListenersOnReplication="true"/>

   <!--
    Channel 是 Tomcat 节点之间进行通讯的工具。
    Channel 包括 5 个组件：Membership、Receiver、Sender、
    Transport、Interceptor
   -->
  <Channel className="org.apache.catalina.tribes.group.GroupChannel">
     <!--
      Membership 维护集群的可用节点列表。它可以检查到新增的节点，
      也可以检查没有心跳的节点
      className－指定 Membership 使用的类
      address－组播地址
      port－组播端口
      frequency－发送心跳 (向组播地址发送 UDP 数据包) 的时间间隔 (单位:ms)。
      dropTime－Membership 在 dropTime(单位:ms) 内未收到某一节点的心跳，
      则将该节点从可用节点列表删除。默认值为 3000。
     -->
     <Membership  className="org.apache.catalina.tribes.membership.
         McastService"
         address="228.0.0.4"
         port="45564"
         frequency="500"
         dropTime="3000"/>
     
     <!--
       Receiver 用于各个节点接收其他节点发送的数据。
       接收器分为两种：BioReceiver(阻塞式)、NioReceiver(非阻塞式)

       className－指定 Receiver 使用的类
       address－接收消息的地址
       port－接收消息的端口
       autoBind－端口的变化区间，如果 port 为 4000，autoBind 为 100，
                 接收器将在 4000-4099 间取一个端口进行监听。
       selectorTimeout－NioReceiver 内 Selector 轮询的超时时间
       maxThreads－线程池的最大线程数
     -->
     <Receiver className="org.apache.catalina.tribes.transport.nio.
         NioReceiver"
         address="auto"
         port="4000"
         autoBind="100"
         selectorTimeout="5000"
         maxThreads="6"/>

      <!--
         Sender 用于向其他节点发送数据，Sender 内嵌了 Transport 组件，
         Transport 真正负责发送消息。
      -->
      <Sender className="org.apache.catalina.tribes.transport.
          ReplicationTransmitter">
          <!--
            Transport 分为两种：bio.PooledMultiSender(阻塞式)
            和 nio.PooledParallelSender(非阻塞式)，PooledParallelSender
            是从 tcp 连接池中获取连接，可以实现并行发送，即集群中的节点可以
            同时向其他所有节点发送数据而互不影响。
           -->
          <Transport className="org.apache.catalina.tribes.
          transport.nio.PooledParallelSender"/>     
       </Sender>
       
       <!--
         Interceptor : Cluster 的拦截器
         TcpFailureDetector－TcpFailureDetector 可以拦截到某个节点关闭
         的信息，并尝试通过 TCP 连接到此节点，以确保此节点真正关闭，从而更新集
         群可用节点列表                 
        -->
       <Interceptor className="org.apache.catalina.tribes.group.
       interceptors.TcpFailureDetector"/>
       
       <!--
         MessageDispatchInterceptor－查看 Cluster 组件发送消息的
         方式是否设置为 Channel.SEND_OPTIONS_ASYNCHRONOUS，如果是，
         MessageDispatchInterceptor 先将等待发送的消息进行排队，
         然后将排好队的消息转给 Sender。
        -->
       <Interceptor className="org.apache.catalina.tribes.group.
       interceptors.MessageDispatchInterceptor"/>
  </Channel>

  <!--
    Valve : Tomcat 的拦截器，
    ReplicationValve－在处理请求前后打日志；过滤不涉及 Session 变化的请求。                 
    -->
  <Valve className="org.apache.catalina.ha.tcp.ReplicationValve"
    filter=""/>
  <Valve className="org.apache.catalina.ha.session.
  JvmRouteBinderValve"/>
  <valve className="org.apache.catalina.ha.tcp.ReplicationValve"
         filter=".*\.gif|.*\.js|.*\.jpeg|.*\.jpg|.*\.png|.*\.htm|.*\.html|.*\.css|.*\.txt"
 
  <!--
    Deployer 用于集群的 farm 功能，监控应用中文件的更新，以保证集群中所有节点
    应用的一致性，如某个用户上传文件到集群中某个节点的应用程序目录下，Deployer
    会监测到这一操作并把文件拷贝到集群中其他节点相同应用的对应目录下以保持
    所有应用的一致，这是一个相当强大的功能。
  -->
  <Deployer className="org.apache.catalina.ha.deploy.FarmWarDeployer"
     tempDir="/tmp/war-temp/"
     deployDir="/tmp/war-deploy/"
     watchDir="/tmp/war-listen/"
     watchEnabled="false"/>

  <!--
    ClusterListener : 监听器，监听 Cluster 组件接收的消息
    使用 DeltaManager 时，Cluster 接收的信息通过 ClusterSessionListener
    传递给 DeltaManager，从而更新自己的 Session 列表。
    -->
  <ClusterListener className="org.apache.catalina.ha.session.
  ClusterSessionListener"/>
  
</Cluster>
```

集群同通信：组播，即Tomcat启动和运行的时候会周期性（默认500ms）向一组服务器发送组播心跳包，同一个集群的Tomcat都在相同的地址和端口监听这些信息，一定时间（默认3s）内不发送组播报文的节点被认为删除，就将其从本地维护的集群列表中移除。

默认情况下，使用DeltaManager进行集群通信，采用的是All-to-All的工作方式，Session会拷贝到集群内所有服务，集群内数量比较多的时候同步时间较长。

也可以使用BackupManager进行通信，Session只会拷贝到备份节点。

集群时，推荐所有的Tomcat使用相同的配置。

工作过程（Tomcat A和Tomcat B构成集群）：

1. 当在server.xml中配置了cluster组件时，Tomcat A在启动Host容器时，会关联Cluster组件。如果web应用在web.xml中配置了Distributable时，Tomcat会为此上下文创建一个DeltaManager，Cluster的默认实现SimpleTcpCluster启动Membership和Replication服务；
2. Tomcat B启动，前面也是启动Host容器，关联Cluster，然后启动一个由A和B组成的Membership。接着Tomcat B会向A请求Session数据，如果A没用响应，则60s后time out，session完成以前不回响应浏览器请求；
3. 用户请求，如果是ReplicationValve filter中配置的请求，就不拦截，否则就会更新session，并利用Replication服务通过TCP连接发送到Tomcat B进行拷贝；拷贝时会拷贝Session中所有可序列化的数据；
4. A崩溃，Tomcat B将A从MemberShip中移除，同时负载均衡器会将所有的请求发到Tomcat B；
5. B正常提供服务；
6. A启动，重复1，2步；
7. A接收用户注销请求，同时向B发送session过期的消息；
8. B创建session，同步到A；
9. A上的session超时过期，B同样过期（只要系统时间保持一致）。

Tomcat原生的集群通信适用于小规模集群。