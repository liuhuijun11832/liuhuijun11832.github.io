---
title: 深入Tomcat/Jetty-两种容器的基本架构
date: 2019-07-18 14:10:37
categories: 学习笔记
tags:
- Jetty
- Tomcat
keywords: [Jetty,Tomcat]
description: 深入学习Jetty和Tomcat
---

# Tomcat系统架构
1. 连接器，将网络数据包装成Request以及将servlet返回的数据包装成Response；
2. Servlet容器：加载、调用Servlet处理Request。

连接器：ProtocolHandler组件+Adapter组件；

ProtocolHandler组件：
1. EndPoint，通信端点，包含Accept监听Socket连接请求，SocketProcess（实现runnable接口）处理Socket连接请求（Executor线程池）；
2. Processor，读取网络数据字节包装成Request和Response，定义了请求的处理，即：EndPoint生成一个SocketProcessor，提交到线程池，SocketProcessor的Run方法调用Processor组件解析应用层协议，Processor生成Request对象后，会调用Adapter的Service方法。

Adapter组件：

适配器模式，将传进来的Request转成ServletRequest，然后执行Service方法。

<!-- more-->

## Servlet容器结构
1. Engine：一个Tomcat最多只有一个；
2. Host：每个站点对应一个；
3. Context：每一个WEB程序都有一个；
4. Wrapper：每一个Servlet有一个。

对应的server.xml文件格式如下：
```xml
<Server>
    <Service>
        <Connector>
        </Connector>
        <Engine>
            <Host>
                <Context>
                </Context>
            </Host>
        </Engine>
    </Service>
</Server>
```
树形结构，所以Tomcat采用组合模式来管理这些子容器，它们都共同实现了Container的接口，该接口还扩展了LifeCycle接口来进行生命周期的管理。

通过Mapper组件来实现组件与访问路径的映射关系。

调用过程通过Pipeline-Value责任链来实现，其中Value是节点的接口，Pipeline是组装链的接口。每一个子容器都有一个PipleLine对象，只要Adapter触发第一个Value，整个链都会执行，而最后一个Value又会调用下层子容器的Pipleline中的第一个Value，并由Value来invoke。Wrapper的最后Value会创建一个FilterChain，并执行doFilter方法。

## 生命周期
LifeCycle接口定义了init，start，stop，destroy等方法，通过上层组件的状态变化触发子组件的状态变化，同时状态变化作为一种事件，通过观察者模式来监听到事件变化。

LefeCycleBase抽象基类定义了一些通用逻辑，利用模板设计模式在基类中实现了生命状态的转变和维护，以及监听器的添加和删除等。Tomcat自定义了一些监听器在创建子组件的时候添加，同时用户还可以在server.xml中手动配置监听器。

各个组件的职能（从上往下依此）：
1. start.sh脚本，启动一个JVM进程来运行BootStrap；
2. BootStrap，初始化Tomcat类加载器，并创建Catalina；
3. Catalina，启动类，解析server.xml，创建相应组件，调用Server的start方法；
4. Server，管理Service组件，调用Service的start方法；
5. Service，管理连接器和顶层容器Engine。

# Jetty系统架构
1. Connector:多个连接器，功能也是接收Socket收到的请求；
2. Handler：多个处理器，处理请求；
3. ThreadPool：分配线程资源给以上两个组件。

和Tomcat的区别：
1. 没有Service；
2. 连接器是被所有Handler共享的，Tomcat是每一个Service都有一个Connector；
3. ThreadPool资源是全局的，Tomcat中每一个连接器有自己的连接池。


Accept:
-  功能：接收请求，对应NIO中的channel；
-  类型：ServerConnector内部类，实现了runnable；
-  描述：通过阻塞的方式来接收连接，接收连接成功以后会调用accepted()函数。

SelecetorManager：
- 功能：选择具体的Selector来处理Channel；
- 类型：具体类，内部保存了一个ManagedSelector数组；
- 描述：调用Selector的register方法把Channel注册到Selector上，拿到一个SelectionKey，然后创建一个EndPoint和Connection，并跟这个SelectionKey绑定在一起，通过调用EndPoint获得一个具体执行任务的线程，交给线程池执行。

Connection：
- 功能：处理Channel；
- 类型：EndPoint内部类，类似于Tomcat的Processor，解析协议包装成Request，并调用Handler；
- 描述：HttpConnection是具体实现类，会向EndPoint注册一堆回调方法，模拟了异步IO的模型。


Jetty总结：

**Accept接收请求，一个连接对应一个Channel，并交由SelecetorManager将Channel注册到Selector上，同时创建EndPoint和Connection绑定注册时产生的SelectionKey，然后获得线程执行，并调用Connection注册到EndPoint里的回调方法读取数据，Connection生成请求对象调用具体的Handler组件，我们可以通过自定义Handler实现功能的。**

## Jetty核心设计-Handler
Handler是一个接口，有一个AbstractHandler，抽象类有两种子类，HandlerWrapper包含一个其他Handler的引用，HandlerCollection中有一个Handler数组的引用。

Handler有三种类型：
- 协调Handler：负责将请求路由到一组Handler中去，比如路由到HandlerCollection中；
- 过滤Handler：自己处理请求，然后转发到下一个Handler，例如HandlerWrapper；
- 内容Handler：真正调用Servlet处理请求，比如ServletHandler处理动态资源或者ResourceHandler处理静态资源请求。

## Jetty对Servlet规范的实现
- ServletHandler：实现了规范中的Servlet，Filter，Listener的功能，依赖FilterHolder、ServletHolder、ServletMapping、FilterMapping；
- SessionHandler：管理Session，同时包含SecurityHandler和GzipHandler；
- WebAppContext：本身是一个Context，管理ServletHandler和SessionHandler。



## 总结
1. 组件化和可配置：利用面向接口编程的思想，实现组件的扩展；并设计成链，在一起链式调用；
2. 启动过程中动态创建组件对象实例（反射）；
3. 父组件负责子组件的创建、启停、销毁；并且将容器的生命周期状态转变定义成一个事件，组件的状态变化触发另一个组件的启动或停止（这些称之为触发点或者扩展点，Spring框架就射这么设计的）。

spring生命周期：
1. 反射实例化Bean和注入属性值；
2. 如果有实现了Aware接口，执行相应的Aware注入：BeanNameAware，BeanFactoryAware，ApplicationContextAware等；
3. 调用BeanPostProcessor的postProcessBeforeInitialization()方法；
4. 调用InitializingBean的实现类中的afterPropertiesSet()方法；
5. 执行init-method定义的方法或者@PostContruct注解的方法；
6. 调用BeanPostProcessor的postProcessBeforeInitialization()方法；
7. 销毁bean时执行，如果该bean实现了DisposableBean接口，则执行destroy()方法；
8. 执行destroy-method定义的方法或者@PreDestroy注解的方法。