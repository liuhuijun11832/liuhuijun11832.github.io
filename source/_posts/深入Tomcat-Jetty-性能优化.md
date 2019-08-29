---
title: 深入Tomcat/Jetty-性能优化
categories: 编程技术
date: 2019-08-24 11:06:40
tags: 
- Tomcat
- Jetty
keywords: [Tomcat,Jetty]
description: 深入学习Tomcat/Jetty的笔记
---

# CMS 和 G1

CMS将整个堆分成了新生代、老年代两大部分，新生代分为Eden和Survivor，新的对象通常情况下都在Eden区域创建，收集一次以后如果对象还在则会转移到Survivor区域，收集多次达到一定次数对象还在的话就会转移到老年代。**分代收集是为了更好地管理和回收对象**，因为每个代对对象管理的方法不一样。



G1使用非连续空间，所以它能够管理更大的堆，同时能够并发完成大部分GC的工作，这期间不会发生STW；G1管理的每个区域都称之为region，它的固定大小通常为2m。



# 调优原则

CMS：合理设置新生代和老年代的大小；

G1：设置堆总的大小和GC最大停顿时间：`-XX:MaxGCPauseMillis`;

# 实战

## 工具

Apache Jmeter：5.1.1，[https://jmeter.apache.org/download_jmeter.cgi](https://jmeter.apache.org/download_jmeter.cgi)；

GCViewer：JDK 8使用 1.3.5的Release版本即可，如果是JDK 8以上，需要下载1.3.5以上的版本，没有的话可能需要自己下载当前最新代码进行构建，[https://github.com/chewiebug/GCViewer](https://github.com/chewiebug/GCViewer)。

## 代码

创建一个Spring Boot程序（Spring Boot 2.1），只需要mvc依赖即可，测试代码如下：

```java
@RestController
public class GcTestController {
		//公有的全局变量，队列里数量到达200000即进行回收，用来模仿老年代对象
    private Queue<Greeting> objCache = new ConcurrentLinkedQueue<>();

    @GetMapping("/greeting")
    public Greeting greeting(){
        Greeting greeting = new Greeting("Hello World");
        if (objCache.size() > 200000) objCache.clear();
        else objCache.add(greeting);
        return greeting;
    }
}

@Data
@AllArgsConstructor
class Greeting{
    private String message;
}
```

JDK8 下，启动参数如下：

`java -Xmx32m -Xss256k -verbose:gc -Xloggc:gc-pid%p-%t.log -XX:+PrintGCDetails -jar target/tomcat-jetty-test-0.0.1-SNAPSHOT.jar`。

JDK 9以及以上，启动参数如下：

`java -Xmx32m -Xss256k -verbosegc -Xlog:gc*,gc+ref=debug,gc+heap=debug,gc+age=trace:file=gc-%p-%t.log:tags,uptime,time,level:filecount=2,filesize=100m`

32M方便看到Full GC，`-verbose:gc`打印GC日志，`-xloggc:gc`用于生成gc日志文件，后面是文件名格式，`PrintGCDetails`打印日志详情包括停顿时间等。使用Jmeter工具创建一个线程组，并创建一个Http请求，持续时间为15分钟。

![Jemter线程组配置.png](深入Tomcat-Jetty-性能优化/Jemter线程组配置.png)

15分钟以后，使用GCviewer打开gc日志，结果如下：

![GCViewer-result.png](深入Tomcat-Jetty-性能优化/GCViewer-result.png)

这里只勾选了上面view工具栏中的蓝色、黑色、绿色这三条线，蓝色代表使用的堆，黑色代表进行了full gc，密集绿色代表了总的GC。可以得出结论：

* 堆的使用率提升导致了新生代频繁GC（绿色线）；
* full gc以后堆内存降低，不是内存泄漏。

所以判断出，这是堆内存不够，因为绿色线和黑色线都比较密集。

调大内存，再试一试15分钟Jmeter：

`java -Xmx1024m -Xss256k -verbose:gc -Xloggc:gc-pid%p-%t.log -XX:+PrintGCDetails -jar target/tomcat-jetty-test-0.0.1-SNAPSHOT.jar`

![GCViewer-1024M-result.png](深入Tomcat-Jetty-性能优化/GCViewer-1024M-result.png)

可以看到没有了Full GC，并且新生代GC没有上一幅图那么频繁密集，而GC停顿时间也只有2.6秒，比上图15秒好太多了。

# 监控

为了模拟服务端Tomcat的监控，改造一下上节的spring boot项目，主要是pom文件里的两个依赖：

```xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
			<exclusions>
				<exclusion>
					<artifactId>spring-boot-starter-tomcat</artifactId>
					<groupId>org.springframework.boot</groupId>
				</exclusion>
			</exclusions>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-tomcat</artifactId>
			<scope>provided</scope>
		</dependency>
```

然后修改启动类中的代码：

```java
@SpringBootApplication
public class Application extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(Application.class);
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

如果提示缺少javax的jar包，在pom文件中引入并将scope填写为provided即可。

这样就可以放到外置的Tomcat容器中运行了，在启动外置Tomcat之前，还需要编辑一个脚本文件setenv.sh，写入以下脚本：

```sh
export JAVA_OPTS="${JAVA_OPTS} -Dcom.sun.management.jmxremote"
export JAVA_OPTS="${JAVA_OPTS} -Dcom.sun.management.jmxremote.port=9001"
export JAVA_OPTS="${JAVA_OPTS} -Djava.rmi.server.hostname=x.x.x.x"
export JAVA_OPTS="${JAVA_OPTS} -Dcom.sun.management.jmxremote.ssl=false"
export JAVA_OPTS="${JAVA_OPTS} -Dcom.sun.management.jmxremote.authenticate=false"
```

放到tomcat的bin目录下，这样即可设置好环境变量，但是使用bin目录下shutdown.sh关闭时可能会报`address already in use`的错，猜测是在执行该脚本的时候又执行了setenv.sh文件，所以报了地址被占用的错。推荐在bin目录下再写一个脚本：

```sh
#! /bin/sh
source /etc/profile
ps -ef | grep tomcat | grep -v grep | awk '{print $2}'| xargs kill
sh /usr/local/tomcat9/bin/startup.sh 
```

用于重启tomcat。

输入`jconsole x.x.x.x:9001`打开jconsole界面，这里主要记录**吞吐量、响应时间、错误数、线程池、CPU、JVM情况**等。

![Jconsole-Window.png](深入Tomcat-Jetty-性能优化/Jconsole-Window.png)

`maxTime:`最长响应时间；

`processTime:`平均响应时间；

`requestCount:`吞吐量；

`errorCount:`错误数。

线程标签可以看到线程的堆栈和等待状态，以及堆栈；内存标签能够看到各个区的空间使用量，VM概要能够看到JVM的基本信息。

命令行可以使用以下方式：

```sh
ps -ef | grep tomcat
cat /proc/30086/status 
```

其中30086就是tomcat的pid，然后使用`top -p 30086`可以看到该进程所占用的系统资源。

## 工具

和上一节一样。

## 实战

在controller添加代码：

```java
		@GetMapping("/greeting/latency/{seconds}")
    public Greeting delayGreeting(@PathVariable Long seconds){
        try {
            TimeUnit.SECONDS.sleep(seconds);
        } catch (InterruptedException e) {
            log.error("睡眠异常", e);
        }
        Greeting greeting = new Greeting("Hello Second");
        return greeting;
    }
```

通过url传入参数来达到睡眠的效果。

使用Jmeter测试，分为三个阶段，即睡眠2s、睡眠4s和睡眠6s来测试，同时设置线程组响应超时为1000ms（这样Jemeter就不会等到请求返回），100个线程。得到如下结果：

![Jconsole-lab-result.png](深入Tomcat-Jetty-性能优化/Jconsole-lab-result.png)

图中有几条不和谐的线是当时由于连接数太多导致Jconsole丢失连接，所以线程数降为0。

首先看线程，一开始压测未开始的时候只有30条线程左右，等到2s睡眠的压测开始以后，由于睡眠2s才会返回数据，所以同时有200条线程在运行，加上一些负责网络通线和后台任务，线程一共差不多250条左右；第一阶段停止后，又下降到40（不是0，为0是因为第二阶段压测开始，丢失连接导致无法正确获取数据），第二阶段需要450条线程，第三阶段需要650条左右线程。

再看内存，基本上随着线程的增加，创建线程导致内存的消耗也会提高。

最后看CPU，基本上CPU的峰值稳定，只要吞吐量一定，CPU的占用率基本保持一定。

