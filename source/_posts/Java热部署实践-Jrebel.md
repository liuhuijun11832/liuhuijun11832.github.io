---
title: Java热部署实践-Jrebel
categories: 编程技术
date: 2019-02-22 10:40:49
tags:	Java
keywords:	[JRebel,Java,热部署]
description: Jrebel在IDEA 2018.1的安装破解，以及spring boot dev tools的介绍
---
# 描述
热部署的作用是在不重启项目的情况下，使用类加载器重新加载修改过的.class文件到内存，避免花费时间在重启上。目前比较常用的有两种热部署，分别为spring-boot-devtools和JRebel。spring-boot-devtools官网：[https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-running-with-the-maven-plugin](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-running-with-the-maven-plugin)；使用方式是直接引入Maven插件或者Gradle插件即可：
<!--more-->

```xml
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-devtools</artifactId>
		<optional>true</optional>
	</dependency>
</dependencies>
```
```gradle
configurations {
	developmentOnly
	runtimeClasspath {
		extendsFrom developmentOnly
	}
}
dependencies {
	developmentOnly("org.springframework.boot:spring-boot-devtools")
}
```

同时官网说明如下：

> Restart vs Reload
The restart technology provided by Spring Boot works by using two classloaders. Classes that do not change (for example, those from third-party jars) are loaded into a base classloader. Classes that you are actively developing are loaded into a restart classloader. When the application is restarted, the restart classloader is thrown away and a new one is created. This approach means that application restarts are typically much faster than “cold starts”, since the base classloader is already available and populated.
If you find that restarts are not quick enough for your applications or you encounter classloading issues, you could consider reloading technologies such as JRebelfrom ZeroTurnaround. These work by rewriting classes as they are loaded to make them more amenable to reloading.
 
解释过来就是：spring boot提供了两个类加载器来进行热部署，一个叫基础类加载器，用于加载不会变化的一些系统jar包和第三方jar包；另一个是重启类加载器，当项目发生更改时，重启类加载器会抛弃旧的类加载器，并重新创建一个重启类加载器（个人觉得spring boot这里用到了OSGI的模块化思想），这意味者这种热重启通常意义下是比冷重启是快很多的，因为基础类加载器一直是可用并且就绪。
如果你发现重启速度不够快或者发现了一些重启导致的问题，你可以使用ZeroTurnaround﻿的产品JRebel，它通过重写class文件来使得class文件适合被重新加载。

在一般情况下，我们使用spring boot的热部署即可，然而我们的项目是需要通过RPC方式进行调用和通信的，这势必会导致一个问题，我们在远程发布服务并暴露通知到本地jvm，启动项目用的是v1 restart classLoader，当我们改了A.java某一行代码，原来v1 restart classLoader被抛弃，现在A.java是由v2 restart classLoader加载的。当你本地调用远程服务时，它会发现本地jvm里的A.java与远程A.java类信息不一致，于是会报 "A cannot cast to A"的错误。所以这里我们采用JRebel来进行重启。
JRebel并不是开源免费的，作为一个商业产品，它并不算便宜，有条件的请支持正版。

# 安装集成
环境描述：
    
1. Intellij IDEA 2018.1
1. JRebel 2018.1

JRebel集成到IDEA只需要到IDEA的plugin里，点击Browse repository，输入JRebel，安装JRebel for IntelliJ，完成以后点击重启IDEA。再次进入到IDEA设置界面，此时已经多了JRebel的选项:
![](Java热部署实践-Jrebel/1540305050860_10.png)

激活需要反向代理工具，github链接：
[https://github.com/ilanyu](https://github.com/ilanyu)
是不是一个眼熟的名字……是的，同时也是IDEA破解一系列的作者，在校大学生。
选择ReverseProxy，再进入到Release页面：

![](Java热部署实践-Jrebel/1540305050882_11.png)

选择对应版本下载，darwin代表Mac os的UNIX-like系统。
 
选择立刻激活，再选择I have license：
![](Java热部署实践-Jrebel/1540305050913_12.png)
运行刚刚下载的反向代理：
![](Java热部署实践-Jrebel/1540305050076_7.png)

代表监听本机的8888端口请求发往了作者提供的一个激活服务器，有条件的可以自己搭建一个类似的激活服务器避免lanyu的服务器被封。
输入上图中类似的内容，除了[http://127.0.0.1:8888]()不能改变以外，后面的内容可以随便填写，但是不能直接写明文，需要转换为GUID的形式。例如[http://127.0.0.1:8888/liuhuijun]()，搜索GUID生成工具，输入liuhuijun生成，则上图中正确的内容为[http://127.0.0.1:8888/e250f540-41e3-450b-aabb-0f376f83c241]()，下面的邮箱可以随意填写，只要格式正确即可，然后可以看到激活成功，可以使用180天，这个时候就可以再JRebel的界面点击work offline，开始脱机工作，如果180天后还想继续使用，可以点击Renew ofline seat，重新获取180天（Renew可以不借助反向代理）。

以上激活步骤，目前适用于JRebel 2018.1等较新版本，其他版本未测试。有条件请支持正版！
接下来，就可以享受热部署的顺滑了……，左边启动，右边debug。还可以点击左下角的JRebel唤出panel勾选项目，它会自动在对应的项目下面的resource生成一个rebel.xml文件，可以根据官方来进行一些自定义配置。
![](Java热部署实践-Jrebel/1540305050033_6.png)
![](Java热部署实践-Jrebel/1540305050466_8.png)

# 安装扩展Mybatis
以上的标准步骤所激活的JRebel是不支持Mybatis里的xml更新热部署的，如果你想更新了sql也能够热部署， 请继续往下看：
支持sql热部署的JRebel叫JRebel-nightly，官网：
[https://zeroturnaround.com/software/jrebel/download/nightly-build/#!/intellij ](https://zeroturnaround.com/software/jrebel/download/nightly-build/#!/intellij)
![](Java热部署实践-Jrebel/1540305054391_17.png)

方式一：下载第一个红框里地址的压缩包，解压开，里面有一个JRebel.jar，记住其位置，然后在IDEA里面指定使用的代理jar包：
![](Java热部署实践-Jrebel/1540305053000_15.png)
![](Java热部署实践-Jrebel/1540305052789_13.png)
使用刚刚解压的那个作为JRebel的代理类。

方式二：.就是不使用IDEA自带的的plugin repository的插件，直接在JRebel-nightly的那个页面选择下载第二个红框里的包，然后卸载原来的JRebel，从磁盘安装你新下载的这个插件压缩包（无需解压）,然后重复上面的po 解步骤即可。

# 一些配置
为方便开发，可以通过设置IDEA来实现完全的实时更新部署。
![](Java热部署实践-Jrebel/1540305053442_16.png)

勾选上图中的自动编译：
然后按Ctrl + Shift + a，弹出万能搜索框，输入registry

![](Java热部署实践-Jrebel/1540305052976_14.png)
再输入running，勾选运行时自动构建：

![](Java热部署实践-Jrebel/1540305050831_9.png)
这样当你更新代码后，即会触发JRebel的热部署，但是同时也会增加IDEA的开销，如果代码报错还会导致报错，所以推荐还是用手动编译吧，快捷键：Ctrl + F9 编译整个项目 Ctrl + Shift + F9，编译刚刚修改的类。
# 补充
（2018-12-20更新）
对于spring boot的devtools热部署出现的"A cannot cast to A"的问题，可以通过配置热部署配置文件来让某个类文件使用同一个类加载器加载。
例如：Mybatis的通用Mapper在使用devtools的时候，通用Mapper会使用base类加载器，而项目中的实体类会使用restat类加载器，会导致上述的错，只要保证它们使用同一个类加载器就可以解决，可以在 src/main/resources 中创建 META-INF 目录，在此目录下添加 spring-devtools.properties 配置，内容如下：

	restart.include.mapper=/mapper-[\\w-\\.]+jar
	restart.include.pagehelper=/pagehelper-[\\w-\\.]+jar
	
dubbo里也可能会出现这种情况，所以也可以将dubbo的类加载器配置为使用restart类加载器。

