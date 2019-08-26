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

![Jemter线程组配置.png](/Users/liuhuijun/Desktop/blog/source/_posts/深入Tomcat-Jetty-性能优化/Jemter线程组配置.png)

15分钟以后，使用GCviewer打开gc日志，结果如下：

![GCViewer-result.png](/Users/liuhuijun/Desktop/blog/source/_posts/深入Tomcat-Jetty-性能优化/GCViewer-result.png)

这里只勾选了上面view工具栏中的蓝色、黑色、绿色这三条线，蓝色代表使用的堆，黑色代表进行了full gc，密集绿色代表了总的GC。可以得出结论：

* 堆的使用率提升导致了新生代频繁GC（绿色线）；
* full gc以后堆内存降低，不是内存泄漏。

所以判断出，这是堆内存不够，因为绿色线和黑色线都比较密集。

调大内存，再试一试15分钟Jmeter：

`java -Xmx1024m -Xss256k -verbose:gc -Xloggc:gc-pid%p-%t.log -XX:+PrintGCDetails -jar target/tomcat-jetty-test-0.0.1-SNAPSHOT.jar`

![GCViewer-1024M-result.png](/Users/liuhuijun/Desktop/blog/source/_posts/深入Tomcat-Jetty-性能优化/GCViewer-1024M-result.png)

可以看到没有了Full GC，并且新生代GC没有上一幅图那么频繁密集，而GC停顿时间也只有2.6秒，比上图15秒好太多了。