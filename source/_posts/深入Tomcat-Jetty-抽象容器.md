---
title: 深入Tomcat/Jetty-抽象容器
categories: 编程技术
date: 2019-07-19 14:51:31
tags:
keywords:
description:
---

# Tomcat-Host容器

热加载：后台线程定期检测类的变化，如果有变化，就重新加载类，不会清空session；

热部署：后台线程定时检测应用的变化，如果有变化，就会清空session；

线程池：ScheduledThreadPoolExecutor，周期性执行

方法：`exec.scheduleWithFixedDelay(
              new ContainerBackgroundProcessor(),// 要执行的 Runnable
              backgroundProcessorDelay, // 第一次执行延迟多久
              backgroundProcessorDelay, // 之后每次执行间隔多久
              TimeUnit.SECONDS);        // 时间单位`

ContainerBackgroundProcessor，ContainerBase内部类，实现Runnable接口；

执行当前容器里的backgroundProcess方法，并且递归调用子容器的backgroundProcess方法，如果子容器的backgroundProcessorDelay大于0，表明子容器有自己的线程，所以此时就不用父类来调用；

这个方法是接口中的默认方法，所以顶层engine启动后台线程以后，它会启动顶层engine以及子engine的周期性任务。

由于ContainerBase是所有组件基类，所以其他组件都可以由自己的周期性任务。

## 热加载

通过context的backgroundProcess实现：

- webapploader检查WEB-INF/classes和WEB-INF/lib下目录的类文件有没有变化；
- sessionManager检查是否有过期session；
- WebResources检查静态资源是否有变化。

webapploader的工作：

- 停止和销毁context以及wrapper；
- 停止和销毁关联的filter和listener；
- 停止和销毁关联的pipline-value；
- 停止和销毁类加载器，和相关的类文件资源；
- 启动context，并重新生成前四步销毁的资源。

**默认情况下，Tomcat的热加载功能是关闭的，需要配置`<Context reloadable="true"/>`来启用**。

## 热部署

通过Host容器实现，Host通过监听HostConfig实现，HostConfig就是前文”周期性事件“的监听器。

- 检查web应用目录是否被删除，如果被删除了就把相应的Context容器销毁；
- 如果有新的War包，就部署相应的应用。

宏观webapps目录级别，而不是web应用目录下文件的变化。