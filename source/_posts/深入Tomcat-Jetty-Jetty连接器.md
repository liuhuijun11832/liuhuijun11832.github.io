---
title: 深入Tomcat/Jetty-Jetty连接器
categories: 编程技术
date: 2019-07-18 14:29:18
tags:
- Tomcat
- Jetty
keywords:[Tomcat,Jetty]
description:深入学习Tomcat和Jetty的笔记
---

# Jetty-Selector
## ManagedSelector
```java
public class ManagedSelector extends ContainerLifeCycle implements Dumpable
{
    // 原子变量，表明当前的 ManagedSelector 是否已经启动
    private final AtomicBoolean _started = new AtomicBoolean(false);
    
    // 表明是否阻塞在 select 调用上
    private boolean _selecting = false;
    
    // 管理器的引用，SelectorManager 管理若干 ManagedSelector 的生命周期
    private final SelectorManager _selectorManager;
    
    //ManagedSelector 不止一个，为它们每人分配一个 id
    private final int _id;
    
    // 关键的执行策略，生产者和消费者是否在同一个线程处理由它决定
    private final ExecutionStrategy _strategy;
    
    //Java 原生的 Selector
    private Selector _selector;
    
    //"Selector 更新任务 " 队列
    private Deque<SelectorUpdate> _updates = new ArrayDeque<>();
    private Deque<SelectorUpdate> _updateable = new ArrayDeque<>();
    
    ...
}
```
## SelectorUpdate接口
Jetty将Channel注册到Selector的事件抽象为SelectorUpdate接口，如果操作ManageSelector中的Selector，需要提交一个任务类，这个类需要实现接口的update方法，在方法里定义想要的操作。

例如Connector中Endpoint组件对读就绪事件感兴趣，于是就向ManagedSelector提交了一个内部任务类ManagedSelector.SelectorUpdate，并在update方法里调用updateKey方法，这些update方法的调用者就是ManagedSelector自己，它在一个死循环里拉取这些SelectorUpdate任务类逐个执行。

## Selectable接口
I/O事件到达时，通过这个接口返回一个Runnable，这个Runnable就是I/O事件就绪时的处理逻辑。
```java
public interface Selectable
{
    // 当某一个 Channel 的 I/O 事件就绪后，ManagedSelector 会调用的回调函数
    Runnable onSelected();

    // 当所有事件处理完了之后 ManagedSelector 会调的回调函数，我们先忽略。
    void updateKey();
}
```
当Channel被选中时，ManagedSelector调用这个Channel所绑定的附件类的onSelected方法来拿到一个Runnable。例如，Endpoint组件在向ManagedSelector注册读就绪事件时，同时也要告诉ManagedSelector在事件就绪时执行什么任务，具体而言就是传入一个附件类，这个附件类需要实现Selectable接口。

## ExecutionStrategy
```java
public interface ExecutionStrategy
{
    // 只在 HTTP2 中用到，简单起见，我们先忽略这个方法。
    public void dispatch();

    // 实现具体执行策略，任务生产出来后可能由当前线程执行，也可能由新线程来执行
    public void produce();
    
    // 任务的生产委托给 Producer 内部接口，
    public interface Producer
    {
        // 生产一个 Runnable(任务)
        Runnable produce();
    }
}
```
具体的策略实现类有四种：
- ProduceConsume：生产者自己依次生产和执行任务，也就是用一个线程来侦测和处理一个ManagedSelector上所有的I/O事件；
- ProduceExecuteConsume：生产者开启新线程来运行任务；
- ExecuteProduceConsume：生产者自己运行自己生产的任务，但是该策略可能会新建一个新线程继续生产和执行任务，能利用CPU缓存；
- EatWhatYouKill：对ExecuteProduceConsume策略的改良，如果线程不够或者系统繁忙，就会切换成ExecuteProduceConsume的策略，因为它使用的线程来源于Jetty全局线程池，一旦被阻塞的多了，会连I/O侦测都没有线程可用。

Jetty的实现：
SelectorProducer是ManagedSelector的内部类，ExecutionStrategy中的Producer接口中的produce方法，需要向ExecutionStrategy返回一个Runnable。这个方法里SelectorProducer主要干了三件事：
1. 如果Channel集合中有I/O事件就绪，就通过Selectable接口获取Runnable，直接返回给ExecutionStrategy去处理；
2. 如果没有，就看看有没有提交SelectorUpdate等事件注册；
3. 继续侦测select方法，侦测I/O事件。