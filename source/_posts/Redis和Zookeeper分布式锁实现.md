---
title: Redis和Zookeeper分布式锁实现
categories: 编程技术
date: 2019-04-20 20:30:19
tags: 分布式锁
keywords: [Redis,Zookeeper,分布式锁]
description: redis和zookeeper的Java实现
---

# 简述

锁是一种同步机制，保证了多线程的有序竞争和运行，而在分布式的场景下，两个应用程序同样会有对于公共变量的访问和操作行为，对于分布式锁，常用的有三种方案：

1. 数据库方式，使用select * from table where column = para for update加排他锁;

2. 中间件缓存，例如redis的setnx+lua脚本或者set key value ps milliseconds nx;

3. zookeeper临时节点。

分布式锁要满足以下几个条件：

- 互斥；

- 不死锁；

- 容错；

- 唯一解锁。

<!--more-->

# Redis

本机环境：Windows 版Redis

IDE：IDEA 2019.1.1

## Jedis

引入pom:

```xml
<dependency>
	<groupId>redis.clients</groupId>
	<artifactId>jedis</artifactId>
	<version>2.9.0</version>
</dependency>
```

要注意的是，尽量保证加锁和释放锁时的原子操作，以及value的唯一性和value与会话的匹配：

```java
public class RedisPool {

    private static final String LOCK_SUCCESS = "OK";
    private static final String SET_IF_NOT_EXISTS = "NX";
    private static final String SET_WITH_EXPIRE_TIME = "PX";
    private static final Long RELEASE_SUCCESS = 1L;

    /**
     * @description 获取分布式锁
     * @param jedis
     * @param lockKey
     * @param requestId
     * @param expireTime
     * @return
     */
    public static boolean tryGetDistributedLock(Jedis jedis,String lockKey,String requestId,int expireTime){
        /**
         * lockKey作为key,requestId作为value用于区分加锁的请求，可以使用不重复的字符串例如UUID或者GUID
         * NX表示该key不存在时才会进行set操作
         * PX表示设置过期时间，具体值由最后一个int值决定
         * jedis.setnx()没有提供直接设置超时的操作，如果锁没有释放会导致死锁
         * 这里尽量使用一行操作来set，如果多个操作无法保证原子性
         */
        String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXISTS, SET_WITH_EXPIRE_TIME, expireTime);
        return LOCK_SUCCESS.equals(result);
    }

    /**
     * @description 释放锁
     * @param jedis
     * @param lockKey
     * @param requestId
     * @return
     */
    public static boolean releaseDistributedLock(Jedis jedis,String lockKey,String requestId){
        //将所有的释放和获取操作交由一行Lua脚本操作完成，保证原子操作
        //eval命令执行Lua代码的时候，Lua代码将被当成一个命令去执行，并且直到eval命令执行完成，Redis才会执行其他命令
        //如果使用先get lockKey的值，然后比对requestId的方式判断是否同一请求，可能导致删除的是其他requestID
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));
        return RELEASE_SUCCESS.equals(result);
    }

}
```

## Redisson

使用Jedis只能满足单机redis的场景，对于redis集群，如果出现类似于主备切换等场景，可能会导致锁丢失。Redis的作者提出了Redlock的实现：

- 获取当前unix时间，单位为millisecond；

- 假如有5个redis节点，使用相同的key和具有唯一性的value获取锁；

- 客户端使用当前时间减去第一步里的时间就是获取锁的时间，只有当N/2+1的节点都获取到锁并且使用时间小于失效时间时表示获取成功；

- 如果获取锁超时或者没有获取到锁，应该在所有的节点进行解锁操作。

Redlock类似于Reetrantlock，Redisson封装了Redlock算法，使用eval执行lua脚本。

Redisson提供了几种集群模式：单机SingleServer，ClusterServer，Maste/SlaveServer，SentinelServer：

引入pom:

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.10.6</version>
</dependency>
```

代码：

```java
 Config config = new Config();
 //本机只有一个redis所以使用单机模式

 config.useSingleServer().setAddress("redis://127.0.0.1:6379");

 Redisson redisson = (Redisson) Redisson.create(config);
 RLock lock = redisson().getLock("test");
 lock.lock(60000L, TimeUnit.SECONDS);
 lock.unlock();
```

# Zookeeper

本机环境：Zookeeper 3.4.10

IDE：IDEA 2019.1.1

首先启动zk，然后启动zkCli，创建一个父节点：

```bash
[zk: localhost:2181(CONNECTED) 9] create /LOCKS 00
Created /LOCKS
[zk: localhost:2181(CONNECTED) 10] ls /LOCKS
[]
```

## Zookeeper

引入zk原生jar包，还有辅助的lombok包：

```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.10</version>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.8.16</version>
    <optional>true</optional>
</dependency>
```

zookeeper分布式锁的原理是：**客户端在父节点下创建临时子节点，然后获取所有子节点，判断当前创建的临时节点是否是最小节点，如果是最小节点即表示获取锁，如果不是最小节点则监听当前节点的前一个节点，如果监听到前一节点删除则当前客户端获取到锁***。使用临时节点可以避免死锁，这里使用countDownLatch限制当前只有一个客户端连接zk：

```java
public class ZookeeperClient {

    private static int sessionTimeout = 5000;

    public static ZooKeeper getInstance() throws IOException, InterruptedException {
        final CountDownLatch countDownLatch = new CountDownLatch(1);//countDpwnlatch表示需要等待的线程数，直到该数值变为0才会真正执行任务
        ZooKeeper zooKeeper = new ZooKeeper("127.0.0.1:2181", sessionTimeout , new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {
                if(watchedEvent.getState() == Event.KeeperState.SyncConnected){
                    countDownLatch.countDown();
                }
            }
        });
        countDownLatch.await();
        return zooKeeper;
    }

    public static int getSessionTimeout(){
        return sessionTimeout;
    }
}
```

然后需要一个监听器监听前一节点是否删除：

```java
public class LockWatcher implements Watcher {
    private CountDownLatch countDownLatch;

    public LockWatcher(CountDownLatch countDownLatch) {
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void process(WatchedEvent watchedEvent) {
        if (watchedEvent.getType() == Event.EventType.NodeDeleted) {
            countDownLatch.countDown();
        }
    }
}
```

获取锁的代码：

```java
@Slf4j
public class DistributedLock {

    /**
     * zookeeper分布式锁原理：
     * 节点有序性：节点可以设置为有序的，例如node-1,node-2等
     * 临时节点：超时以后自动删除避免死锁
     * 事件监听：节点变化时客户端可以收到
     */

    private static final String ROOT_LOCK = "/LOCKS";
    private ZooKeeper zooKeeper;
    private int sessionTimeout;
    private String lockId;
    private final static byte[] data = {1, 2};
    private CountDownLatch countDownLatch = new CountDownLatch(1);

    public DistributedLock() throws IOException, InterruptedException {
        this.zooKeeper = ZookeeperClient.getInstance();
        this.sessionTimeout = ZookeeperClient.getSessionTimeout();
    }

    public boolean tryGetDistributedLock() {
        try {
            //这里的四个参数分别是：路径，保存内容，权限，临时有序节点
            lockId = zooKeeper.create(ROOT_LOCK + "/", data, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
            log.info("当前线程:{} 创建节点,id={}", Thread.currentThread().getName(), lockId);
            List<String> childrenList = zooKeeper.getChildren(ROOT_LOCK, true);
            childrenList.sort(String::compareTo);
            for (int i = 0; i < childrenList.size(); i++) {
                childrenList.set(i, ROOT_LOCK + "/" + childrenList.get(i));
            }
            String first = childrenList.get(0);
            if (lockId.equals(first)) {
                log.info("当前线程:{} 获取锁成功,节点为:{}", Thread.currentThread().getName(), lockId);
                return true;
            }
            //获取当前节点前的节点集合

            List<String> lessThanLockIDList = childrenList.subList(0, childrenList.indexOf(lockId));
            if (!lessThanLockIDList.isEmpty()) {
                String preLockID = lessThanLockIDList.get(lessThanLockIDList.size() - 1);
                //监听上一节点变化,如果删除在监听器里会将countDownLatch减1，这样就能执行挂起的客户端

                zooKeeper.exists(preLockID, new LockWatcher(countDownLatch));
                //使用countDownLatch闭锁来挂起当前线程直到lockWatcher监听到上一节点的变化countDown了或者超时sessionTimeout以后
                countDownLatch.await(sessionTimeout, TimeUnit.MILLISECONDS);
                log.info("当前线程:{} 获取锁成功,节点为:{}", Thread.currentThread().getName(), lockId);
            }
            return true;
        } catch (Exception e) {
            log.error("获取锁异常",e);
        }
        return false;
    }

    public boolean releaseDistributedLock(){
        log.info("当前线程:{} 将要释放锁:{}", Thread.currentThread().getName(), lockId);
        try {
            zooKeeper.delete(lockId, -1);
            log.info("当前线程:{} 释放锁:{} 成功", Thread.currentThread().getName(), lockId);
            return true;
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
        return false;
    }

}
```

测试类：

```java
    public static void main(String[] args) {
        final CountDownLatch countDownLatch = new CountDownLatch(10);
        Random random = new Random();
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                DistributedLock distributedLock = null;
                try {
                    distributedLock = new DistributedLock();
                    countDownLatch.countDown();
                    countDownLatch.await();
                    distributedLock.tryGetDistributedLock();
                    Thread.sleep(random.nextInt(500));
                } catch (IOException e) {
                    e.printStackTrace();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    if (distributedLock != null) {
                        distributedLock.releaseDistributedLock();
                    }
                }
            }).start();
        }
    }
```

执行方法可以看到结果：

```textile
17:55:03.680 [Thread-0] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-0 创建节点,id=/LOCKS/0000000157
17:55:03.680 [Thread-7] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-7 创建节点,id=/LOCKS/0000000163
17:55:03.680 [Thread-5] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-5 创建节点,id=/LOCKS/0000000156
17:55:03.680 [Thread-3] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-3 创建节点,id=/LOCKS/0000000154
17:55:03.680 [Thread-2] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-2 创建节点,id=/LOCKS/0000000161
17:55:03.680 [Thread-8] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-8 创建节点,id=/LOCKS/0000000155
17:55:03.680 [Thread-1] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-1 创建节点,id=/LOCKS/0000000162
17:55:03.680 [Thread-6] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-6 创建节点,id=/LOCKS/0000000159
17:55:03.680 [Thread-4] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-4 创建节点,id=/LOCKS/0000000160
17:55:03.680 [Thread-9] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-9 创建节点,id=/LOCKS/0000000158
17:55:03.691 [Thread-3] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-3 获取锁成功,节点为:/LOCKS/0000000154
17:55:03.929 [Thread-3] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-3 将要释放锁:/LOCKS/0000000154
17:55:03.935 [Thread-8] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-8 获取锁成功,节点为:/LOCKS/0000000155
17:55:03.936 [Thread-3] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-3 释放锁:/LOCKS/0000000154 成功
17:55:04.106 [Thread-8] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-8 将要释放锁:/LOCKS/0000000155
17:55:04.108 [Thread-8] INFO com.joy.lock.zookeeper.DistributedLock - 当前线程:Thread-8 释放锁:/LOCKS/0000000155 成功
//...省略后面日志
```

## Curator

当然apache已经封装好了分布式锁的实现，需要引入Curator的jar包：

```
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>2.12.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>2.12.0</version>
        </dependency>
```

Java代码比较简单：

```java
@Slf4j
public class CuratorDistributedLock {

    private static final String ZK_ADDRESS = "127.0.0.1:2181";

    private static final String ROOT_LOCK = "/LOCKS";

    static CuratorFramework client = CuratorFrameworkFactory.newClient(ZK_ADDRESS, new RetryNTimes(10, 500));
    static InterProcessMutex lock = new InterProcessMutex(client, ROOT_LOCK);

    public static void tryGetDistributedLock() {
        try {
            if (lock.acquire(10 * 10000, TimeUnit.MILLISECONDS)) {
                log.info("当前线程:{}获取锁",Thread.currentThread().getName());
                Thread.sleep(5000L);
                CuratorDistributedLock.releaseDistributedLock();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void releaseDistributedLock() throws Exception {
        log.info("当前线程:{}释放锁",Thread.currentThread().getName());
        lock.release();
    }

}
```

测试类如下：

```java
public static void main(String[] args) {
        final CountDownLatch countDownLatch = new CountDownLatch(3);
        client.start();
        for (int i = 0; i < 3; i++) {
            new Thread(() -> {
                countDownLatch.countDown();
                try {
                    countDownLatch.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                CuratorDistributedLock.tryGetDistributedLock();
            }).start();
        }
    }
```

日志如下：

```textile
18:04:55.467 [Thread-2] INFO com.joy.lock.zookeeper.CuratorDistributedLock - 当前线程:Thread-2获取锁
18:05:00.468 [Thread-2] INFO com.joy.lock.zookeeper.CuratorDistributedLock - 当前线程:Thread-2释放锁
18:05:00.478 [Thread-1] INFO com.joy.lock.zookeeper.CuratorDistributedLock - 当前线程:Thread-1获取锁
18:05:05.479 [Thread-1] INFO com.joy.lock.zookeeper.CuratorDistributedLock - 当前线程:Thread-1释放锁
18:05:05.484 [Thread-3] INFO com.joy.lock.zookeeper.CuratorDistributedLock - 当前线程:Thread-3获取锁
18:05:10.485 [Thread-3] INFO com.joy.lock.zookeeper.CuratorDistributedLock - 当前线程:Thread-3释放锁
```

> 在程序运行中，可以在zkcli中执行`ls /LOCKS`随时看临时子节点的存在。

# 参考

https://blog.csdn.net/qq_26857649/article/details/82383853

阿飞的博客：

https://mp.weixin.qq.com/s/XoXcqpehhXSQlRxgCBtDcw

https://mp.weixin.qq.com/s/PnlPgqfVXqJmN26vvGp5MA


