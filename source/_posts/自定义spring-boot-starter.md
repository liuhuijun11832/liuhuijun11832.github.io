---
title: 自定义spring-boot-starter
categories: 编程技术
date: 2019-04-16 23:47:30
tags:
- Java
- Spring Boot
keywords: [Java,Spring]
description: Spring-Boot-Starter揭秘
---

# 简述

随着Spring Boot的流行，身为一个Java程序员怎么可以不会使用Spring-Boot-Starter呢？Spring Boot的自动装配原理是其核心注解@EnableAutoConfiguration里的@Import导入AutoConfigurationImportSelector.class，从而通过该类中SpringFactoriesLoader.loadFactoryNames方法扫描具有META-INF/spring.factories的jar包实现自动装配。

<!--more-->

# 代码演示

需要引入如下依赖：

```xml
<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-autoconfigure</artifactId>
      <version>2.1.0.RELEASE</version>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-configuration-processor</artifactId>
      <version>2.1.0.RELEASE</version>
      <optional>true</optional>
    </dependency>
  </dependencies>
```

> 注意：spring boot版本不一致可能会导致问题。

首先定义一个配置类用于接收配置的参数类，“joy.hello”是prefix的值，代表获取joy.hello的值放入该类：

```java
@ConfigurationProperties("joy.hello")
public class HelloServiceProperties {

    private static final String MSG = "world";

    private String msg = MSG;

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

接下来模拟一个能自动装配的bean：

```java
public class HelloService {

    private String msg;

    public void sayHello(){
        System.out.printf("hello,%s",this.msg);
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

该类的作用其实可以理解为RedisTemplate一类的bean。

最后一步，也是最重要的一步，就是开启配置类并且指定其初始化条件：

```java
@Configuration
@EnableConfigurationProperties(HelloServiceProperties.class)
@ConditionalOnClass(HelloService.class)
@ConditionalOnProperty(prefix = "joy.hello",value = "enabled",matchIfMissing = true,h)

public class HelloServiceAutoConfiguration {

    @Autowired
    private HelloServiceProperties helloServiceProperties;

    @Bean
    @ConditionalOnMissingBean(HelloService.class)
    public HelloService helloService(){
        HelloService helloService = new HelloService();
        helloService.setMsg(helloServiceProperties.getMsg());
        return helloService;
    }

}
```

首先，`@Configuration`表示会将该类作为一个spring的配置类，`@EnableConfigurationProperties`会开启对注解配置bean的支持，不然`@ConfigurationProperties`会报错。`@ConditionalOnClass`表示当项目中存在HelloService.class的时候才会自动装配，`@ConditionalOnProperty`表示去判断joy.hello.enabled的值，如果为空，则返回true(matchIfMissing控制)，如果不为空，则会去和havingValue的值比较，如果相当返回true否则为false。

最后，我们需要注册到META-INF/spring.factories里，内容如下：

`org.springframework.boot.autoconfigure.EnableAutoConfiguration=\  
 com.joy.autoconfigure.HelloServiceAutoConfiguration`

将项目打成jar包，新建一个spring boot项目，引入我们自定义的starter，然后在配置文件里即可定义joy.hello.msg的值：

```yaml
joy:
  hello:
    msg: liuhuijun
```

启动该spring boot的启动类：

```java
@SpringBootApplication
public class MyBlogApplication implements CommandLineRunner {


	@Autowired
	HelloService helloService;

	public static void main(String[] args) {
		SpringApplication.run(MyBlogApplication.class, args);
	}

	@Override
	public void run(String... args) throws Exception {
		helloService.sayHello();
	}
}
```

即可看到我们配置的值。

# 总结

官方推荐：

1. 如果是spring官方包，推荐命名为spring-boot-starter-xxx的形式；

2. 如果是第三方包，推荐命名为xxx-spring-boot-starter的形式，例如：mybatis-spring-boot-starter。

类似mybatis等的自动装配，跟上面代码类似：

```java
@Configuration
@ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})
@ConditionalOnBean({DataSource.class})
@EnableConfigurationProperties({MybatisProperties.class})
@AutoConfigureAfter({DataSourceAutoConfiguration.class})
public class MybatisAutoConfiguration {
    private static final Logger logger = LoggerFactory.getLogger(MybatisAutoConfiguration.class);
    private final MybatisProperties properties;
    private final Interceptor[] interceptors;
    private final ResourceLoader resourceLoader;
    private final DatabaseIdProvider databaseIdProvider;
    private final List<ConfigurationCustomizer> configurationCustomizers;

    public MybatisAutoConfiguration(MybatisProperties properties, ObjectProvider<Interceptor[]> interceptorsProvider, ResourceLoader resourceLoader, ObjectProvider<DatabaseIdProvider> databaseIdProvider, ObjectProvider<List<ConfigurationCustomizer>> configurationCustomizersProvider) {
        this.properties = properties;
        this.interceptors = (Interceptor[])interceptorsProvider.getIfAvailable();
        this.resourceLoader = resourceLoader;
        this.databaseIdProvider = (DatabaseIdProvider)databaseIdProvider.getIfAvailable();
        this.configurationCustomizers = (List)configurationCustomizersProvider.getIfAvailable();
    }

    @PostConstruct
    public void checkConfigFileExists() {
        if (this.properties.isCheckConfigLocation() && StringUtils.hasText(this.properties.getConfigLocation())) {
            Resource resource = this.resourceLoader.getResource(this.properties.getConfigLocation());
            Assert.state(resource.exists(), "Cannot find config location: " + resource + " (please add config file or check your Mybatis configuration)");
        }

    }
    //...省略大部分方法
  }
```
