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


