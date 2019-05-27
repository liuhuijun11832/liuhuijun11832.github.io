---
title: Spring整合RabbitMQ
categories: 编程技术
date: 2019-05-21 15:01:12
tags: RabbitMQ
keywords: [Spring,RabbitMQ]
description: Spring整合RabbitMQ的常见用法。
---

# 简述

RabbitMQ应该是业界内最大名鼎鼎的消息队列之一了，尤其随着分布式系统架构的盛行，消息队列的作用也越来越大，它的应用场景有解耦，削峰，数据冗余，广播，缓冲，顺序保证等等。

RabbitMQ使用Erlang语言编写，实现AMQP，同时支持MQTT，STOMP等多种消息协议，具有高可靠性、高扩展性等特点，现在由Pivotal公司维护。

<!-- more-->

# 环境

服务器环境：

    CentOS Linux 7.2
    
    RabbitMQ Server 3.6.10
    
    Erlang OTP 20.0

客户端环境：

    JDK 11
    
    IDEA 2019.1

# RabbitMQ分析

首先看一下RabbitMQ的模型：

![Spring整合RabbitMQ\rabbit-model](Spring整合RabbitMQ\rabbit-model.png)

这里的每一台broker即代表一台服务器，一个生产者-消费者模型，producer发送的消息里会包含routingKey，经过exchange时，交换器会根据bindingKey发送到对应的queue，routingKey和bindingKey大部分情况下是相等的，除了topic的exchange以外，bindingKey是绑定queue和exchange的routingKey。

下面看一下它的工作流程：

![Spring整合RabbitMQ\work-order](Spring整合RabbitMQ\work-order.png)

这里使用NIO模型，每一个连接线程打开了一个channel，多个channel复用同一个TCP连接。

RabbitMQ中常用的交换机Exchange有四种类型：

1. fanout，发送到该类型交换机的消息会发送到与之绑定的所有队列上，即不会管RoutingKey和BindingKey的关系；

2. direct，会根据消息的RoutingKey发送到对应BindingKey=RoutingKey的队列中；

3. topic，与direct类似，但是BindingKey是特殊格式，用以模糊匹配RoutingKey；

4. headers，绑定交换机和队列时需要指定键值对KV1，同时发送消息时会在消息内容中带上headers属性，该属性也为一个键值对KV2，只有当KV1=KV2时才发送到对应队列，效率和实用性都很低。

# RabbitMQ安装

RabbitMQ依赖于Erlang环境，所以我们需要下载Erlang的包[https://www.erlang.org/downloads/20.0](https://www.erlang.org/downloads/20.0)。下载tar.gz包到服务器，解压开，在解压开的目录下分别执行

```bash
./configure --prefix=/opt/erlang
make
make install
```

过程不再赘述，缺对应的包即安装缺少的包（baidu即可）。最后编辑环境变量，由于prefix已经指定了目录，剩下的配置和JDK的配置类似，最后可以输入erl命令看是否安装成功，如果安装成功，则会打印：

    Erlang/OTP 20 [erts-9.1] [source] [64-bit] [smp:48:48] [ds:48:48:10] [async-threads:10] [hipe] [kernel-poll:false]
    
    Eshell V9.1  (abort with ^G)

然后下载RabbitMQ的tar.gz包，使用tar xzvf 命令解压开，进入到解压后的目录下的bin目录中，输入`rabbitmq-server -detached`即可启动，并保持后台运行。

如果需要启用控制台，还需要执行`rabbitmq-plugins enable rabbitmq_management`，使用guest/guest可以登录。不过，默认情况下这个账户只允许本地网络访问，所以我们需要添加一个新用户，添加权限和角色：

```bash
[root@load_balance_2 ~]rabbitmqctl add_user root root123
[root@load_balance_2 ~]rabbitmqctl set_permissions -p / root ".*" ".*" ".*"
[root@load_balance_2 ~]rabbitmqctl set_user_tags root administrator
```

访问rabbitmq所在服务器的15672端口，即可访问控制台。

当然，docker安装更为简单，无需上面那么多步骤，直接下载rabbitmq的镜像，然后一步搞定：

`docker run -d  --name rabbitmq -p 25672:25672 -p 5672:5672 -p 15672:15672 rabbitmq:latest`

# 代码整合

这里采用父子模块的方案来构建整体骨架。点击New，新建一个Project，在弹出来的框中选择Maven选项，并勾选Create from archetype，选择maven-archetype-quickstart，新建项目后，删除src等源代码目录。在该项目里再New一个Module，同样是Maven项目，但是archetype选择maven-archetype-webapp。

父pom文件主要内容如下：

```xml
   ...
     <modules>
        <module>rabbit-provider</module>
        <module>rabbit-consumer</module>
        <module>rabbit-common</module>
    </modules>
    <packaging>pom</packaging>
    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.amqp</groupId>
            <artifactId>spring-amqp</artifactId>
            <version>1.7.6.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.amqp</groupId>
            <artifactId>spring-rabbit</artifactId>
            <version>1.7.6.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <version>4.3.14.RELEASE</version>
        </dependency>
    </dependencies>
    ...
```

消费者和生产者pom文件主要内容：

```xml
    ...
    <parent>
        <artifactId>simple-rabbit-demo</artifactId>
        <groupId>com.joy</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <packaging>war</packaging>
    <dependencies>
        <dependency>
            <groupId>com.joy</groupId>
            <artifactId>rabbit-common</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>
    ...
```

消费者和生产者的mq配置文件resources/rabbit.properties：

```properties
rabbit.hostname=192.168.15.118
rabbit.port=5672
rabbit.username=root
rabbit.password=root123
default-queue=default-queue
```

spring配置resources/applicationContext.xml如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:property-placeholder location="classpath:rabbit.properties"></context:property-placeholder>

    <!-- 自动扫描 (需要修改为自己项目的路径)-->
    <context:component-scan base-package="com.joy">
    </context:component-scan>

</beans>
```

消费者和生产者WEB-INF/web.xml配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
 <web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd" version="2.5">
 <display-name>Archetype Created Web Application</display-name>
     <listener>
         <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
     </listener>
     <context-param>
         <param-name>contextConfigLocation</param-name>
         <param-value>classpath:applicationContext.xml</param-value>
     </context-param>
     <!-- 防止spring内存溢出的监听器-->
     <listener>
         <listener-class>org.springframework.web.util.IntrospectorCleanupListener</listener-class>
     </listener>
 </web-app>
```

common模块中只有一个实体类User：

```java
public class User implements Serializable {

    private final static long serialVersionUID = 1L;

    private String name;
    private Integer age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

生产者默认配置如下，统一使用个人钟爱的注解式配置：

```java
/**
 * @Description: 生产者配置

 * @Author: Joy
 * @Date: 2019-05-20 11:28
 */
@Configuration
public class RabbitConfiguration {

    @Value("${rabbit.hostname}")
    private String rabbitHost;
    @Value("${rabbit.port}")
    private int rabbitPort;
    @Value("${rabbit.username}")
    private String rabbitUName;
    @Value("${rabbit.password}")
    private String rabbitPassword;

    @Bean
    public ConnectionFactory connectionFactory(){
        CachingConnectionFactory cachingConnectionFactory = new CachingConnectionFactory();
        cachingConnectionFactory.setHost(rabbitHost);
        cachingConnectionFactory.setPort(rabbitPort);
        cachingConnectionFactory.setUsername(rabbitUName);
        cachingConnectionFactory.setPassword(rabbitPassword);
        return cachingConnectionFactory;
    }

    @Bean
    public AmqpAdmin amqpAdmin(){
        return new RabbitAdmin((connectionFactory()));
    }

    @Bean
    public AmqpTemplate rabbitTemplate(){
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory());
        rabbitTemplate.setMessageConverter(messageConverter());
        return rabbitTemplate;
    }
}
```

生产者测试类如下：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:applicationContext.xml")
public class AmqpTest {

    @Autowired
    private RabbitTemplate rabbitTemplate;

}
```

消费者配置默认如下：

```java
@EnableRabbit
@Configuration
public class RabbitConfiguration {

    @Value("${rabbit.hostname}")
    private String rabbitHost;
    @Value("${rabbit.port}")
    private int rabbitPort;
    @Value("${rabbit.username}")
    private String rabbitUName;
    @Value("${rabbit.password}")
    private String rabbitPassword;

    @Bean
    public ConnectionFactory connectionFactory(){
        CachingConnectionFactory cachingConnectionFactory = new CachingConnectionFactory();
        cachingConnectionFactory.setHost(rabbitHost);
        cachingConnectionFactory.setPort(rabbitPort);
        cachingConnectionFactory.setUsername(rabbitUName);
        cachingConnectionFactory.setPassword(rabbitPassword);
        return cachingConnectionFactory;
    }

    /**
     * 使用注解式驱动方法监听

     * @return
     */
    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory() {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory());
        factory.setConcurrentConsumers(3);
        factory.setMaxConcurrentConsumers(10);

        return factory;
    }

    @Bean
    public AmqpAdmin amqpAdmin(){
        return new RabbitAdmin((connectionFactory()));
    }
}
```

## 默认交换机

如果不指定交换机，只是指定了一个队列，那么该默认绑定到RabbitMQ上的一个默认交换机，类型为direct，并且其routingKey就是队列名，配置如下(无论配置在生产者还是消费者都可以，或者两者都配置也可以，如果不配置的话消费者可能会报没有找到队列的错误)：

```java
    @Bean
    public Queue defultQueue(){
        //默认durable为true，exclusive为false，auto-delete为false

        return new Queue("default-queue");
    }
```

消费者监听传统会使用这种方式：

```java
@Component
public class DefaultListener implements MessageListener {
    @Override
    public void onMessage(Message message) {
        System.out.println(new String(message.getBody()));
    }
}
```

但是我这里使用注解驱动的方式来监听，这样就可以使用方法级别来监听队列了，避免建立大量的类：

```java
@Component
public class Myservice {
    /**
     * 监听默认队列
     * @param data
     */
    @RabbitListener(queues = "${default-queue}")
    public void processDefault(String data){
        System.out.println("default-queue"+"===>"+data);
    }
}
```

并且这里的@RabbitListener里的queues属性还可以使用spring $取值的形式或者spEL表达式来取值。**注意：使用注解驱动的方式需要在任一配置类中加上@EnableRabbit注解，上文中消费者配置类中已经加上**。

在生产者中测试类中，新建一个单元测试：

```java
    @Test
    public void sendToDefaultQueue(){
        //使用默认交换机
        rabbitTemplate.convertAndSend("default-queue","默认队列");
    }
```

为消费者添加一个tomcat或者jetty，启动，此时进入控制台，可以看到Queues页面表格多了一个default-queue队列：![Spring整合RabbitMQ\default-queue](Spring整合RabbitMQ\default-queue.png)

其中features一栏中，D代表开启了持久化。同时在Exchanges页面表格中，可以看到多了一个(AMQP default)名字的交换机。运行单元测试，消费者打印出："default-queue===>默认队列"

## Fanout交换机

该类型交换机指定routingKey会无效，所以消息会发送到所有与该交换机绑定的队列上。在消费者或者生产者增加配置：

```java
    //声明一个可以持久化的fanout交换机
    @Bean
    public Exchange testFanoutExchange(){
        return ExchangeBuilder.fanoutExchange("joy.fanout.exchange").durable(true).build();
    }
    //声明第一个队列
    @Bean
    public Queue testFanoutQueue1(){
        Queue queue = new Queue("queue1",true);
        return queue;
    }
    //将第一个队列绑定到fanout交换机上
    @Bean
    public Binding testFanoutBuilding(){
        return BindingBuilder.bind(testFanoutQueue1()).to(testFanoutExchange()).with("queue-1").noargs();
    }

    //声明第二个队列
    @Bean
    public Queue testFanoutQueue2(){
        return new Queue("queue2", true);
    }

    //将第二个队列绑定到fanout交换机上
    @Bean
    public Binding testFanoutBuilding1(){
        return BindingBuilder.bind(testFanoutQueue2()).to(testFanoutExchange()).with("queue-2").noargs();
    }
```

消费者的MyService类中新增两个方法消费消息：

```java
    /**
     * 监听第一个队列
     * @param data
     */
    @RabbitListener(bindings = @QueueBinding(value = @Queue(value = "queue1",durable = "true"),
            exchange = @Exchange(value = "joy.fanout.exchange",type = ExchangeTypes.FANOUT,durable = "true"),key = "queue-1"))
    public void process1(String data){
        System.out.println("fanout-queue1"+"===>"+data);
    }

    /**
     * 监听第二个队列
     * @param data
     */
    @RabbitListener(bindings = @QueueBinding(value = @Queue(value = "queue2",durable = "true"),
            exchange = @Exchange(value = "joy.fanout.exchange",type = ExchangeTypes.FANOUT,durable = "true"),key = "queue-2"))
    public void process2(String data){
        System.out.println("fanout-queue2"+"===>"+data);
    }
```

**这里在监听的方法上使用bingdings属性注解可以避免因队列不存在而报错，换句话说，这些注解的作用其实就是创建队列、交换机以及绑定队列和交换机的rountingKey，有了这些注解其实也可以不用上面的配置**。

生产者测试类中新增单元测试，并运行：

```java
    @Test
    public void sendToFanoutExchange(){
        //自定义交换机以及与其绑定的对列名
        rabbitTemplate.setExchange("joy.fanout.exchange");
        rabbitTemplate.setRoutingKey("queue-1");

        rabbitTemplate.convertAndSend("我是谁？我在哪？我要干什么？");
    }
```

消费者打印出：

    fanout-queue2===>我是谁？我在哪？我要干什么？
    fanout-queue1===>我是谁？我在哪？我要干什么？

此时无论指定routingKey为什么都会发送到所有绑定到该fanout类型交换机的队列上，就和广播一样。

## Direct交换机

最好理解的交换机，发送消息时指定routingKey，交换机根据该routingKey发送到绑定时bindingKey与该路由键一致的队列上。

配置如下：

```java
    //声明一个direct类型的交换机
    @Bean
    public Exchange testDirectExchange(){
        return ExchangeBuilder.directExchange("joy.direct.exchange").durable(true).build();
    }

    //将第三个队列绑定到direct交换机上
    @Bean
    public Binding testDirectBinding1(){
        return BindingBuilder.bind(directQueue()).to(testDirectExchange()).with("queue-3").noargs();
    }
    //声明第四个队列
    @Bean
    public Queue directQueue1(){
        return new Queue("queue4");
    }

    //将第四个队列绑定感到direct交换机上
    @Bean
    public Binding testDirectBinding2(){
        return BindingBuilder.bind(directQueue1()).to(testDirectExchange()).with("queue-4").noargs();
    }
```

或者在消费者注解指定，如果在配置中配置了，这里就可以直接使用用`@RabbitListener(queues="队列名")`的方式指定了：

```java
/**
     * 监听第三个队列
     * @param user
     */
    @RabbitListener(bindings = @QueueBinding(value = @Queue(value = "queue3",durable = "true"),
            exchange = @Exchange(value = "joy.direct.exchange",durable = "true"), key = "queue-3"))
    public void process3( User user){
        System.out.println("direct-queue3"+"===>"+user.toString());
    }

    /**
     * 监听第四个队列
     * @param user
     */
    @RabbitListener(bindings = @QueueBinding(value = @Queue(value = "queue4",durable = "true"),
            exchange = @Exchange(value = "joy.direct.exchange",durable = "true"),key = "queue-4"))
    public void process4( User user) {
        System.out.println("direct-queue4===>"+user.toString());
    }
```

这里和前面不同的一点是，这里是直接在方法参数里接收一个对象，如果需要这么做，那么该对象需要实现java的Serializable接口，并且指定一个messageConverter。拿常用的Json格式举例：首先需要在生产者中调整配置：

```java
    @Bean
    public AmqpTemplate rabbitTemplate(){
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory());
        rabbitTemplate.setMessageConverter(messageConverter());
        return rabbitTemplate;
    }

    @Bean
    public MessageConverter messageConverter(){
        return new Jackson2JsonMessageConverter();
    }
```

然后调整消费者的配置：

```java
    /**
     * 使用注解式驱动方法监听
     * @return
     */
    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory() {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory());
        factory.setConcurrentConsumers(3);
        factory.setMaxConcurrentConsumers(10);
        //添加消息转换器

        factory.setMessageConverter(messageConverter());
        return factory;
    }

    @Bean
    public MessageConverter messageConverter(){
        return new Jackson2JsonMessageConverter();
    }
```

新增单元测试方法并执行：

```java
    @Test
    public void sendToDirectExchange(){
        User user = new User();
        user.setName("刘会俊");
        user.setAge(24);
        //发送direct消息对象
        rabbitTemplate.convertAndSend("joy.direct.exchange", "queue-3", user);
        user.setName("刘半仙");
        user.setAge(124);
        rabbitTemplate.convertAndSend("joy.direct.exchange", "queue-4", user);
    }
```

消费者打印：

    direct-queue3===>User{name='刘会俊', age=24}
    direct-queue4===>User{name='刘半仙', age=124}

## Topic交换机

在该类型的交换机中，约定routingKey和bindingKey由以"."分隔的字符串组成，并且可以使用"*"和"#"进行模糊匹配，其中\*表示匹配一个单词，#表示匹配多个单词。

消费者新增两个监听：

```java
@RabbitListener(bindings = @QueueBinding(value = @Queue(value = "queue5",durable = "true"),
            exchange = @Exchange(value = "joy.topic.exchange",type = ExchangeTypes.TOPIC,durable = "true"),key = "51.#"))
    public void process5(User user){
        System.out.println("51.#===>"+user.toString());
    }

    @RabbitListener(bindings = @QueueBinding(value = @Queue(value = "queue6",durable = "true"),
            exchange = @Exchange(value = "joy.topic.exchange",type = ExchangeTypes.TOPIC,durable = "true"),key = "*.WEB.#"))
    public void process6(User user){
        System.out.println("*.WEB.#===>"+user.toString());
    }
```

其中第一个监听的队列queue5匹配规则是以51.开头，后面有多个字符串或0个字符串；而第二个监听的队列queue6匹配规则是第一部分有一个单词，第二部分为"WEB"字符串的队列。

生产者增加单元测试：

```java
    @Test
    public void sendToTopicExchange(){
        rabbitTemplate.setExchange("joy.topic.exchange");
        User user = new User();
        user.setName("刘二柱");
        user.setAge(18);
        rabbitTemplate.convertAndSend("51.APP.TS",user);
        user.setName("刘一手");
        user.setAge(20);
        rabbitTemplate.convertAndSend("51.WEB.TS",user);
    }
```

预期结果是：刘二柱会被发送到queue5，刘一手会发送到queue5和queue6。

运行结果为：



    51.#===>User{name='刘二柱', age=18}
    *.WEB.#===>User{name='刘一手', age=20}
    51.#===>User{name='刘一手', age=20}



## 延迟队列和死信队列

可以通过设置队列的ttl属性，或者发送消息时的消息属性expiration来实现延迟队列。当消息在ttl或者expiration时间没有消费时，则会进入死信队列（DLX），每一个队列实际上会有一个死信交换机属性，当我们给某个队列设置了死信交换机，并且给该交换机绑定了死信队列时，正常队列中没有消费者监听或者超时的消息都会经过死信交换机进入死信队列。

配置正常队列以及对应的死信队列：

```java
    //声明一个正常队列，并添加定时时间和死信交换器路由
    @Bean
    public Queue normalQueue(){
        Map<String, Object> args = new HashMap<>();
        args.put("x-message-ttl", 10000);
        args.put("x-dead-letter-exchange", "joy.dead.direct.exchange");
        args.put("x-dead-letter-routing-key", "dead-queue");
        Queue normalQueue = new Queue("normal-queue",true,false,true,args);
        return normalQueue;
    }

    //声明一个死信队列
    @Bean
    public Queue deadQueue(){return new Queue("dead-queue",true);}

    //声明一个正常交换机
    @Bean
    public Exchange normalExchange(){
        return ExchangeBuilder.directExchange("joy.normal.direct.exchange").durable(true).build();
    }

    //声明一个私心交换机
    @Bean
    public Exchange deadExchange(){
        return ExchangeBuilder.directExchange("joy.dead.direct.exchange").durable(true).build();
    }

    //声明一个正常绑定关系
    @Bean
    public Binding normalBinding(){
        return BindingBuilder.bind(normalQueue()).to(normalExchange()).with("normal-queue").noargs();
    }

    //将死信队列和死信交换机绑定上
    @Bean
    public Binding deadBinding(){
        return BindingBuilder.bind(deadQueue()).to(deadExchange()).with("dead-queue").noargs();
    }
    
    //声明一个正常队列1,绑定死信交换机和队列参数
    @Bean
    public Queue normalQueue1(){
        Map<String, Object> args = new HashMap<>();
        args.put("x-dead-letter-exchange", "joy.dead.direct.exchange");
        args.put("x-dead-letter-routing-key", "dead-queue");
        Queue normalQueue = new Queue("normal-queue1",true,false,true,args);
        return normalQueue;
    }

    //声明一个正常绑定关系
    @Bean
    public Binding normalBinding1(){
        return BindingBuilder.bind(normalQueue1()).to(normalExchange()).with("normal-queue1").noargs();
    }
```

当然也可以在消费者监听的`@Queue`注解里新增参数argument，这里不再赘述，与上面的配置基本类似。当然，不光可以给队列设置ttl，也可以给消息设置expiration超时，所以上面又设置了一个normal-queue1，这个队列同样绑定了死信，auto-delete为true，并且没有设置ttl。

消费者增加一个死信的监听器：

```java
     /**
     * 监听死信队列消息
     * @param user
     */
    @RabbitListener(queues = "dead-queue")
    public void process7(User user){
        System.out.println("dead-queue===>"+user.toString());
    }
```

增加单元测试：

```java
    @Test
    public void sendToNormalExchange(){
        rabbitTemplate.setExchange("joy.normal.direct.exchange");
        User user = new User();
        user.setName("刘三胖");
        user.setAge(18);
        rabbitTemplate.convertAndSend("normal-queue",user);
        user.setName("刘二丫");
        user.setAge(18);
        rabbitTemplate.convertAndSend("normal-queue1",user,message ->
        {
            message.getMessageProperties().setExpiration("10000");
            return message;
        });
        user.setName("刘狗剩");
        user.setAge(19);
        rabbitTemplate.convertAndSend("normal-queue1",user);
    }    
```

启动项目，会发现控制台中Queues页面多了两个队列如图：

![Spring整合RabbitMQ\normal-queue](Spring整合RabbitMQ\normal-queue.png)

其中AD表示自动删除，TTL表示队列设置了存活时间，DLX表示绑定了死信交换机，DLK表示死信交换机绑定了routingKey。

预期结果：normal-queue的监听器由于设置了10s超时，所以10s以后，死信监听器监听到消息；normal-queue1中的刘二丫由于给消息设置了过期，所以10s以后死信队列也会收到消息；而刘狗剩则会一直待在队列中。

运行结果：



    dead-queue===>User{name='刘三胖', age=18}
    dead-queue===>User{name='刘二丫', age=18}

这两条信息恰好是10s打印的，而狗剩那条消息，则永远的留在了normal-queue1中，可以查看控制台，此处就不再贴图。



# 参考

本文Github代码地址：[https://github.com/liuhuijun11832/spring-rabbit-demo.git](https://github.com/liuhuijun11832/spring-rabbit-demo.git)

参考：

1. 《RabbitMQ实战指南》 朱忠华 著；

2. 《Spring AMQP官方文档》[https://docs.spring.io/spring-amqp/docs/1.7.14.BUILD-SNAPSHOT/reference/html/](https://docs.spring.io/spring-amqp/docs/1.7.14.BUILD-SNAPSHOT/reference/html/) 。
