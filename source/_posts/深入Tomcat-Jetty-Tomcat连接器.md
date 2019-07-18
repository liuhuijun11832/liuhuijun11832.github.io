---
title: 深入Tomcat/Jetty-Tomcat连接器
categories: 学习笔记
date: 2019-07-18 14:26:15
tags:
- Tomcat
- Jetty
keywords:[Tomcat,Jetty]
description:Tomcat和Jetty的深入学习笔记
---

# Tomcat-NioEndpoint
Java I/O模型：**一个进程的地址空间分为用户空间和内核空间**，用户线程不能直接访问内核空间。

同步阻塞I/O：用户线程发起read调用，然后阻塞让出CPU，接着内核等待网卡数据到来，把数据从网卡拷贝到内核空间，最后把数据拷贝到用户空间，把用户线程唤醒；

同步非阻塞I/O：用户线程不断发起read调用，如果没有数据，内核就返回失败，直到数据到了内核空间。在等待数据从内核空间拷贝到用户空间这个时间里，线程还是阻塞的，等数据到了用户空间再把线程叫醒。

I/O多路复用：首先：线程发起select调用，询问内核是否准备好，等准备好，用户线程再发起read调用；多路复用是指select可以向内核查多个数据通道（Channel）的状态。

异步I/O：用户线程发起read调用的同时注册一个回调函数，内核将数据准备好后，再调用指定的回调函数完成处理。在这个过程中，用户线程一直阻没有塞。

NioEndpoint实现了**多路复用**。

<!-- more -->

包含组件：
- LimitLatch：限制连接数；
- Acceptor：监听连接请求，单独线程，死循环调用accept，返回Channel，交给Poller处理；
- Poller：监测Channel的I/O事件，创建SocketProcess任务类扔给线程池处理；
- Executor：Http11Processor处理请求，通过NioSocketWrapper读写数据。

设计特点：
- LimitLatch：内部类Sync，Sync扩展了AQS，AQS内部维护了一个状态和线程队列，用来控制线程什么时候挂起，什么时候唤醒。sync类重写了AQS的tryAcquireShared()方法，如果连接数count小于limit，线程就可以获取锁，否则返回-1，同时还重写了releaseShared()方法。
- Accept：实现了Rnnable接口，可以跑在单独线程里。由于一个端口号对应一个ServerSocketChannel，所以这个ServerSocketChannel实在多个Accept线程之间共享的，它是Endpoint的属性，由Endpint完成初始化和端口绑定。
```java
serverSock = ServerSocketChannel.open();
serverSock.socket().bind(addr, getAcceptCount());
serverSocket.configureBlock(true);
```
bind()的第二个参数表示操作系统的等待队列，默认100；ServerSocketChannel设置称阻塞模式，通过accept()接受新的连接，并返回获得的SocketChannel对象，然后封装在一个PollerEvent对象中，并将该对象压入Poller的Queue中。
- Poller，本质是一个Selector，内部维护了一个Queue`private final SynchornizedQueue<PollerEvent> = new SynchornizedQueue<>()`；poller线程可以有多个同时运行，不断通过内部的Selector对象向内核查询Channel状态，一旦可读就生成任务类SocketProcessor交给Executor处理，并且还需要循环遍历自己所管理的SocketChannel是否已经超时，如果超时就关闭。
- SocketProcessor：任务类，交由线程池Executor处理；
- Excutor：执行SocketProcessor的run方法。

# Tomcat-Nio2EndPoint

Nio2EndPoint实现了**异步I/O**。

Java NIO.2 API创建服务端：
```java
public class Nio2Server{
    void listen(){
        //1.ExecutorService
        ExecutorService es = Executors.newCachedThreadPool();
        //2.ChannelGroup
        AsynchronousChannelGroup tg = AsychronousChannelGroup.with
        //3.Open Channel
        AsynchronousServerSocketChannel assc = AsynchronousServerSocketChannel.open(tg);
        //4.bind
        assc.bind(new InetSocketAddress("127.0.0.1", 8080));
        //5.accept
        assc.accept(this,new AcceptHandler());
    }
}
```
向内核注册回调函数，同时给内核提供一个线程池，内核只需要将工作交给线程池。

NIO2和NIO的明显区别是：NIO2没有Poller组件，因为Selector的工作交给内核来做了。

包含组件：
- LimitLatch：限制连接数；
- Acceptor：自己实现了`CompletionHandler`接口，自己就是回调类，所以自己调用自己`serverSock.accept(null,this);`
- Executor：Http11Processor处理请求，通过Nio2SocketWrapper读写数据。Nio2SocketWrapper主要作用是封装Cahnnel，并提供接口给Http11Processor读写数据。为了实现异步非阻塞I/O，Http11Processor通过两次read调用来完成数据读取操作，第一次read调用发生于连接刚刚建立，acceptor创建了SocketProcessor任务类交给线程池处理，http11Processor在处理过程中会调用Nio2SocketWrapper的read方法发出第一次请求，并且注册回调类readCompletionHandler，由于数据没读到，因此将Nio2SocketWrapper标记为不完整，**然后SocketProcessor线程被回收，Http11Processor并没有阻塞等待数据，但是Http11Processor维护了一个Nio2SocketWrapper列表**，也就是维护了连接状态；第二次read调用发生于内核把数据拷贝到Http11Processor指定的buffer里，回调类readCompletionHandler被调用，重新创建一个SocketProcessor来继续处理这个连接，这个新的sp持有原来的Nio2SocketWrapper。

# Tomcat-AprEndpoint

APR是Apache Protable Runtime Libraries，基于C语言实现的可移植运行库，目的是向上层应用提供一个跨平台的操作系统接口库。ArpEndpoit通过JNI调用APR实现非阻塞I/O。

包含的组件类似于NioEndpoint，但是采用DirectByteBuffer和sendfile来进行优化。

- Accept：监听连接，通过JNI调用APR里的socket、bind、listen、accept；
- Poller：通过JNI调用APR里的poll方法，APR调用操作系统的epoll API；

Tomcat的endpoint组件在接收网络数据时需要预先分配好一块buffer，也就是数组byte[];在进行JNI调用时，将这个buffer地址传递给C代码，C代码把数据填充到这块buffer。

HeapByteBuffer：`ByteBuffer buf = ByteBuffer.allocate(1024)`,该类型buffer本质上还是开辟在JVM的堆内存中，内核将数据复制到本地临时内存，在从本地临时内存复制到JVM堆内存，如果直接从内核复制到JVM堆内存，JVM GC的时候由于对象移动而导致buffer失效，如果通过中转的方式，本地内存写入到JVM堆内存时，由于没有safepoint因此不会发生GC；

DirectByteBuffer：`ByteBuffer buf = ByteBuffer.allocateDirect(1024)`该类型buffer是开辟在本地内存，JVM中只是持有该本地内存的虚引用，DirectByteBuffer在JVM的对象中有个long类型字段address，记录本地内存地址，这样内核将数据拷贝到本地内存地址后，JVM可以直接读取，省掉了一次拷贝过程。

sendfile（系统API）：文件读取到内核缓冲区->数据位置+长度 的描述符添加到Socket缓冲区，内核将对应数据传递给网卡；

传统传输文件的方式：文件读取到内核缓冲区->内核缓冲区放到本地临时内存->本地临时内存->JVM堆->本地临时内存->内核缓冲区->网卡

# Tomcat-Executor
## Java线程池
```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)

```
corePoolSize:核心线程数，如果提交任务时核心线程没满，则创建新线程来执行；

workQueue:如果核心线程数满了，新增任务就放到workQueue里，线程池调用poll方法从workQueue里获取任务；

maximumPoolSize:如果workQueue满了，则创建临时线程，当临时线程达到maximumPoolSize时，执行拒绝策略handler，比如抛出异常或者由调用者线程执行任务；

**临时线程**使用poll(keepAliveTime,unit)方法从工作队列中拉取任务，如果达到超时时间依然没有获取到任务，则该线程会被回收。

- FixedThreadPool:固定长度(nThreads)的线程数组，它的workQueue是一个LinkedBlockingQueue无界队列；
- CachedThreadPool:maximumPoolSize大小为Integer.MAX_VALUE的线程池，无限创建临时线程，闲下来再回收，它的任务队列是SynchronousQueue，长度为0；

## Tomcat线程池
关键点：

- 线程个数；
- 队列长度；
- 重写execute方法；

基本步骤和Tomcat的步骤类似，区别在于：

- 到达maximumPoolSize后，继续尝试把任务添加到任务队列中去；
- 如果缓冲队列也满了，则执行拒绝策略。

其原理就是调用Java的Execute方法，然后捕捉其拒绝策略抛出的异常，再尝试将该任务放到任务队列，如果任务队列也满了，再执行拒绝策略。

```java
public class ThreadPoolExecutor extends java.util.concurrent.ThreadPoolExecutor {
  
  ...
  
  public void execute(Runnable command, long timeout, TimeUnit unit) {
      submittedCount.incrementAndGet();
      try {
          // 调用 Java 原生线程池的 execute 去执行任务
          super.execute(command);
      } catch (RejectedExecutionException rx) {
         // 如果总线程数达到 maximumPoolSize，Java 原生线程池执行拒绝策略
          if (super.getQueue() instanceof TaskQueue) {
              final TaskQueue queue = (TaskQueue)super.getQueue();
              try {
                  // 继续尝试把任务放到任务队列中去
                  if (!queue.force(command, timeout, unit)) {
                      submittedCount.decrementAndGet();
                      // 如果缓冲队列也满了，插入失败，执行拒绝策略。
                      throw new RejectedExecutionException("...");
                  }
              } 
          }
      }
}
```
上面代码中，用submittedCount来维护已经提交到了线程池，但是还没有执行完的任务个数。

原因：Tomcat定制版的任务队列TaskQueue扩展了LinkedBlockingQueue，默认情况下长度是长度无限的，除非给它一个capacity。这个参数一般情况下是maxQueueSize参数设置，但问题是默认情况下maxQueueSize值是Integer.MAX_VALUE，当前线程数达到核心线程数，再来任务直接就被添加到任务队列了，这样永远不会有机会创建新线程。所以定制版的TaskQueue重写了LinkedBlockingQueue的offer方法，在合适的时候返回false，表示添加失败，这样就可以返回新的线程。

判断逻辑：

- 线程数达到maxQueueSize了，直接添加进任务队列；
- 已提交任务数小于当前线程，表示有空闲线程，无需创建新的；
- 已提交任务数大于当前线程，线程不够用，新任务放进TaskQueue；

所以，上面线程池代码中，执行时首先会对任务数首先加一，然后捕捉到异常时又减一。

## WebSocket
请求头：

    Connection: Upgrade
    Upgrader: websocket

Tomcat实现方式：WebSocket Endpoint，每一个WebSocket连接创建一个Endpoint实例，对应一个session。

编程方式：

- 继承javax.websocket.Endpoint，并实现onOpen、onClose和onError方法，每一个session会被作为endpoint各个生命周期事件的参数，通过在session中添加了MessageHandler消息处理器接收消息。
- 注解`@ServerEndpoint(value="/websocket/chat")`在类上，然后类中方法加上@OnOpen,@OnClose,@OnMessage等注解。

Tomcat工作方式：

1. Web加载，通过SCI(ServletContainerInitializer，Servlet 3.0规范中定义)机制接收web 应用启动事件的接口。在实现了SCI接口的类上增加HandlerTypes({ServerEndpoint.class,ServerApplicaitonConfig.class,Endpoint.class})注解，这样Tomcat在启动阶段就会扫描出类出来，作为SCI的onStartup方法参数，并调用onStartup方法。所有扫描到的Endpoint子类和添加了@ServerEndpoint的类都会被放到WebSocketContainer容器中，并且维护了url和endpoint的映射关系。
2. 处理请求，使用UpgradeProcessor处理WebSocket请求。在WebSocket握手请求到来时，HttpProtocolHandler首先接收到这个请求，通过一个特殊的Filter判断是否具有`Upgrader: websocket`信息，如果有，就用UpgradeProtocolHandler替换原来的HttpProtocolHandler，并把当前Socket的Processor替换成UpgradeProcessor，由该Processor调用最终的Endpoint实例处理请求。