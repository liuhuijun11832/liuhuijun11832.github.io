---
title: Spring AOP随记
date: 2019-01-30 10:58:53
update:
categories: 编程技术
tags: 
- Java
- 杂记
---
# 简述
最近看到公司业务代码执行的时候有这么两句：
```java
	long startTime = System.currentMilles();
			...
	long endTime = System.currentMilles();
	LOGGER.info("执行时长{}",endTime-startTime);
	```
每个service层代码几乎都有这么两句，实在是臃肿。
<!-- more -->
# AOP
AOP是对OOP的一种补充，如果说OOP将万事万物都看作对象之间的关系的话，从上到下，例如餐具-盘子-瓷盘，那么AOP定义这种纵向关系之间的一种横向行为，例如盘子可以盛放菜肴，这也可以说是他们的公共行为，而不同材质的盘子适合盛放什么类型的食物或者适合做观赏性的盘子就可以理解为他们的核心业务。
我的理解就是AOP某个系统中定义的一些公共行为，专业名词为“横切关注点”，OOP则是其独有行为，称之为“核心关注点”。
AOP里几个比较关键的概念：

* pointCut：切点，定义拦截的行为或者标志；
* join point：连接点，由于Spring只支持方法级别的连接点，所以在Spring中，join point就是一个方法，但是广义的join point不光是方法，也能是变量或者类；
* aspect：切面，点构成面，即横切关注点抽象出来组织成一个切面；
* advice：通知，满足拦截的行为的标志以后执行的操作，通知分为前置、后置、异常、最终、环绕五大类（后面再写详细点）；
* weave：织入，将切面功能注入进目标方法并创建代理对象的过程；
* introduction：引入，通过动态代理在运行期为类动态添加一些行为或属性；

Spring Aop代理由Spring 的IOC容器生成，管理。所以AOP代理可以使用容器中的其他bean作为代理目标，一般情况下，Spring会使用JDK的动态代理来创建AOP代理，当要代理的对象不是接口时，会使用CGLIB的方式来创建代理，也可以强制使用cglib的方式，代码如下：
xml方式：`<aop:aspectj-autoproxy proxy-target-class="true" />`
﻿spring boot:
`@EnableAspectJAutoProxy﻿
`使用Spring AOP进行编程，通常来说有以下三步：
定义核心业务组件；
定义切点和切面，一个切点和切面可以横跨多个业务；
定义增强处理，这里就是AOP为业务组件织入的处理动作。

使用Spring AOP进行编程，通常来说有以下三步：

1. 定义核心业务组件；
1. 定义切点和切面，一个切点和切面可以横跨多个业务；
1. 定义增强处理，这里就是AOP为业务组件织入的处理动作。

# 实战举例
## 场景1：统计网站访问来源
假如我有一个个人网站，我想统计一下某个接口的访问数，或者主页的访问数，访问来源，并记录下这个访问，那么就可以使用AOP来实现。
step 1：定义切面，切面可以是一个类作为切面；
step 2：定义切点，需求简单，由于我的接口基本都在包com.blog.controller下，所以使用
execution﻿​表达式即可，可以参考[https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop)网站，这里随便记一下execution的格式，根据官网介绍，execution的格式类似于
	
	execution(modifiers-pattern?ret-type-patterndeclaring-type
    -pattern?name-pattern(param-pattern)
	throws-pattern?)
对应中文为execution(访问修饰符表达式？ 返回值类型表达式 名称表达式（参数表达式）异常表达式? )，除了名称表达式，其他表达式都可以不写，下面介绍几种常用的特殊通配符：
访问修饰符表达式：不写代表所有访问修饰符；
返回值类型表达式：\*在返回类型通配符中代表所有返回值类型；
名称通配符：\*在名称通配符中是代表所有的意思，.在名称通配符中代表当前包或者当前类，..两个点表示当前包以及子包；
参数表达式：不写表示无参方法，..表示0或多个参数，\*表示任何类型的一个参数，那么\*，String就表示一个任意类型的参数+一个String类型的参数；
异常表达式：格式为throws(*)表示所有异常。
那么贴出切点定义：


```java	
@Pointcut(value="execution(public*com.blog.controller..*.*(..))")
	public void pointCut(){
	
	}
```
 step3：定义增强处理，首先确认通知类型：
前置通知[Before advice]：在连接点前面执行，前置通知不会影响连接点的执行，除非此处抛出异常。 


正常返回通知[After returning advice]：在连接点正常执行完成后执行，如果连接点抛出异常，则不会执行。 


异常返回通知[After throwing advice]：在连接点抛出异常后执行。 


返回通知[After (finally) advice]：在连接点执行完成后执行，不管是正常执行完成，还是抛出异常，都会执行返回通知中的内容。

 
环绕通知[Around advice]：环绕通知围绕在连接点前后，比如一个方法调用的前后。这是最强大的通知类型，能在方法调用前后自定义一些操作。环绕通知还需要负责决定是继续处理join point(调用ProceedingJoinPoint的proceed方法)还是中断执行。 
我的需求是在返回以后都要记录访问来源，所以使用返回通知类型，代码如下：


```java
	@After(value="pointCut())")
	public void after(JoinPoint joinPoint){
		PvLogpvLog=newPvLog();
		LOGGER.info("当前请求的方法名:{}",joinPoint.getSignature().getName());
		HttpServletRequestrequest=((ServletRequestAttributes)RequestContextHolder.getRequestAttributes()).getRequest();
		StringrealIp=HttpUtil.getIpAddr(request);
		LOGGER.info("当前请求来源IP地址为:{}",realIp);
		pvLog.setUpdateTime(newDate());
		pvLog.setReferer(request.getHeader("Referer"));
		pvLog.setVisitTime(1);
		pvLog.setIp(realIp);
		pvLogMapper.insert(pvLog);
	}
```
	
需要记住，执行这个方法传入参数不能是注解中定义的value里没有的，例如我代码中想传两个参数一个是连接点，一个是注解，所以value中就是pointCut()和@annotation()，如果你想传入一个参数那么就是args(参数名)。
拓展一下切入点指示符（PCD）：
execution:匹配方法执行连接点；
within：限制匹配某些类型中的连接点（spring aop中连接点通常指方法，以下相同）；
args：限制匹配指定参数的连接点，其中参数是指参数名；
@args：限制匹配指定带有指定注解的参数的连接点；
@annotation：限制匹配带有某个注解的方法。
spring aop可以使用&& ，|| ，！来对PCD进行逻辑运算。

## 场景2：统计一个方法执行的时间
step1:定义一个切面；
step2:定义切点和增强，这里我想更灵活一些，通过注解实现我指定的方法来监控执行时长，因此我需要一个自定义注解。
这是打印日志的级别：


```java
	public enum  LoggerEnums {

	    INFO,
	    DEBUG,
	    WARN,
	    ERROR;

	}```
自定义注解：


```java
	@Documented
	@Target({ElementType.METHOD})
	@Retention(RetentionPolicy.RUNTIME)
	@Inherited
	public @interface AopLog {
	
	    LoggerEnums value() default LoggerEnums.INFO;
	
	}```
自定义注解小扩展：
 
@Documented：该注解是否包含在javadoc中；
@Inherited：该注解是否允许被继承；
@Target：该注解表示可以被写在什么位置，枚举类型，常用有：TYPE表示接口、类、枚举、注解；METHOD表示方法，FIELD表示字段或枚举常量，PARAMETER表示方法参数，CONSTRUCTOR表示构造函数；
@Rentention：表示保留级别，分别有RESOURCE（只存在于源码，如@Override和@SuppressWarnings），CLASS（存在于源码和CLASS文件中），RUNTIIME（保留到运行时）


```java
	@Component
	@Aspect
	public class LogAspect {
	
	    private final static Logger LOGGER = LoggerFactory.getLogger(LogAspect.class);
	
	    /**
	     *@description: 定义切点
	     *@author: 刘会俊
	     *@params: []
	     *@return: void
	     */
	@Pointcut("@annotation(com.example.aisino.aop.AopLog)")
	    public void pointCut(){
	
	    }
	
	    /**
	     *@description: 定义环绕通知，计算方法执行时长
	     *@author: 刘会俊
	     *@params: [proceedingJoinPoint]
	     *@return: java.lang.Object
	     */
	    @Around("pointCut() && @annotation(AopLog)")
	    public Object printRequestLog(ProceedingJoinPoint proceedingJoinPoint){
	        long startTime = System.currentTimeMillis();
	        Object object = null;
	        MethodSignature methodSignature = (MethodSignature)proceedingJoinPoint.getSignature();
	        String methodName = methodSignature.getName();
	        AopLog aopLog = methodSignature.getMethod().getDeclaredAnnotation(AopLog.class);
	        try {
	            object = proceedingJoinPoint.proceed();
	        } catch (Throwable throwable) {
	            LOGGER.error("执行{}方法报错",methodName,throwable);
	        }
	        execTime(methodName, startTime, System.currentTimeMillis(),aopLog);
	        return object;
	    }
	
	    private void execTime(String name,long startTime,long endTime,AopLog aopLog ){
	        long usetime = endTime - startTime;
	        long threadId = Thread.currentThread().getId();
	        if (usetime > 1000) LOGGER.warn("当前线程{}执行{}方法执行时间可能过长，时间为{}秒",threadId,name,usetime/1000L);
	        else if (aopLog.value() == LoggerEnums.DEBUG) LOGGER.debug("当前线程<{}>执行<{}>方法执行时长为:{}毫秒",threadId,name,usetime);
	        else if (aopLog.value() == LoggerEnums.WARN) LOGGER.warn("当前线程<{}>执行<{}>方法执行时长为:{}毫秒",threadId,name,usetime);
	        else if (aopLog.value() == LoggerEnums.ERROR) LOGGER.error("当前线程<{}>执行<{}>方法执行时长为:{}毫秒",threadId,name,usetime);
	        else LOGGER.info("当前线程<{}>执行<{}>方法执行时长为:{}毫秒",threadId,name,usetime);
	    }
	
	}```

数据源配置：


	```java
	@EnableAspectJAutoProxy
	@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
	@MapperScan("com.blog.mapper")
	public class MyBlogApplication {

	    /**
		 *@description: DataSourceBuilder是spring 默认创建DataSource的建造者
		 *@author: 刘会俊
		 *@params: []
		 *@return: javax.sql.DataSource
		 */
		@Bean(name="ds1")
		@ConfigurationProperties(prefix = "spring.datasource.db1")
		public DataSource dataSource1(){
			return DataSourceBuilder.create().build();
		}
	
	
		@Bean(name="ds2")
		@ConfigurationProperties(prefix = "spring.datasource.db2")
		public DataSource dataSource2(){
			return DataSourceBuilder.create().build();
		}
	    /**
		 *@description: 由于这里有三个同为DataSource的bean，所以spring在设置jdbc连接的数据源时不知道用哪个，使用Primary注解表示spring优先使用这个datasource，这样就可以实现动态切换了
		 *
		 *@author: 刘会俊
		 *@params: []
		 *@return: javax.sql.DataSource
		 */
		@Primary
		@Bean(name = "dynamicDateSource" )
		public DataSource dynamicDatasource(){
			DynamicDatasource dynamicDatasource = new DynamicDatasource();
			dynamicDatasource.setDefaultTargetDataSource(dataSource1());
			Map<Object, Object> map = new HashMap<>();
			map.put("ds1", dataSource1());
			map.put("ds2", dataSource2());
			dynamicDatasource.setTargetDataSources(map);
			return dynamicDatasource;
		}
	
		@Bean
		public PlatformTransactionManager transactionManager(){
			return new DataSourceTransactionManager(dynamicDatasource());
		}
	
		public static void main(String[] args) {
			SpringApplication.run(MyBlogApplication.class, args);
		}
	}
```

## 场景3：动态切换﻿﻿数据源

step1:定义切面；
step2:
定义切换数据源方法（假设你已经定义了两个DataSource的Spring Bean），即自定义一个数据源key的获取方法，数据源在mybatis中的表现形式就是：


```java
	public class DataSourceContextHolder {

    public static final String DEFAULT_DATASOURCE="ds1";

    private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();

    public static void setDatasource(String datasource){
        contextHolder.set(datasource);
    }

    public static String getDatasource(){
        return contextHolder.get();
    }
```
定义一个数据源路由类，以及两个key，分别为DS1和DS2，作为Spring Bean管理：
```java
	public class DynamicDatasource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        // 从自定义的位置获取数据源标识
        return DataSourceContextHolder.getDataSource();
   	 }
	}```
	
使用自定义注解：


```java
	@Target(value = {ElementType.TYPE,ElementType.METHOD})
	@Retention(RetentionPolicy.RUNTIME)
	@Component
	public @interface DataSource {
	
	    String value() default "";
	
	}
```
定义切点和增强：


```java
	@Component
	@Aspect
	@Order(1)
	public class DataSourceAop {
    private final static Logger LOGGER = LoggerFactory.getLogger(DataSourceAop.class);

    @Pointcut(value = "@annotation(com.blog.aop.DataSource)")
    public void pointCut(){

    }

    @Before(value = "pointCut()")
    public void changeDataSource(JoinPoint joinPoint){
        Class clazz = joinPoint.getTarget().getClass();
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        String name = methodSignature.getName();
        Class[] clazzes = methodSignature.getParameterTypes();
        try {
            Method method = clazz.getMethod(name, clazzes);
            if(method.isAnnotationPresent(DataSource.class) || clazz.isAnnotationPresent(DataSource.class)){
                DataSource dataSource = method.getAnnotation(DataSource.class);
                LOGGER.info("开始切换为数据源:{}",dataSource.value());
                if(!StringUtils.isEmpty(dataSource.value())){
                    DataSourceContextHolder.setDatasource(dataSource.value());
                }else DataSourceContextHolder.setDatasource(DataSourceContextHolder.DEFAULT_DATASOURCE);
            }
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
    }

	}
```
如果使用配置文件xml的形式：

```xml
	<bean id="routingDataSource" class="com.blog.util.DynamicDatasource">
        <property name="targetDataSources">
            <map key-type="java.lang.String">
                <!-- 指定lookupKey和与之对应的数据源 -->
                <entry key="ds1" value-ref="ds1"></entry>
                <entry key="ds2" value-ref="ds2"></entry>
            </map>
        </property>
        <!-- 这里可以指定默认的数据源 -->
        <property name="defaultTargetDataSource" ref="ds1" />
    </bean>
```
如果使用spring boot的方式：
配置文件如下，需要注意的是在spring boot 2.0以后，默认数据源成了HikariDataSource，而Hikari读取的数据库连接地址名称叫jdbc-url，而不是spring读取的url，所以我们要把url改成jdbc-url，直接让Hikari来读取：

```yaml
	spring:
	  datasource:
	    db1:
	      type: com.zaxxer.hikari.HikariDataSource
	      hikari:
	        minimum-idle=5
	        connection-test-query=SELECT 1
	      username: root
	      password: root
	      jdbc-url: jdbc:mysql://localhost:3306/blog?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&useSSL=false&allowMultiQueries=true
	    db2:
	      type: com.zaxxer.hikari.HikariDataSource
	      hikari:
	        minimum-idle=5
	        connection-test-query=SELECT 1
	      username: root
	      password: root
	      jdbc-url: jdbc:mysql:/0.0.0.0:3306/blog?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&useSSL=false&allowMultiQueries=true
```
然后启动类需要禁用spring boot的单数据源自动配置，并且注册两个数据源和动态数据源如下：


```java
	@EnableAspectJAutoProxy
	@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
	@MapperScan("com.blog.mapper")
	public class MyBlogApplication {

	    /**
		 *@description: DataSourceBuilder是spring 默认创建DataSource的建造者
		 *@author: 刘会俊
		 *@params: []
		 *@return: javax.sql.DataSource
		 */
		@Bean(name="ds1")
		@ConfigurationProperties(prefix = "spring.datasource.db1")
		public DataSource dataSource1(){
			return DataSourceBuilder.create().build();
		}
	
	
		@Bean(name="ds2")
		@ConfigurationProperties(prefix = "spring.datasource.db2")
		public DataSource dataSource2(){
			return DataSourceBuilder.create().build();
		}
	    /**
		 *@description: 由于这里有三个同为DataSource的bean，所以spring在设置jdbc连接的数据源时不知道用哪个，使用Primary注解表示spring优先使用这个datasource，这样就可以实现动态切换了
		 *
		 *@author: 刘会俊
		 *@params: []
		 *@return: javax.sql.DataSource
		 */
		@Primary
		@Bean(name = "dynamicDateSource" )
		public DataSource dynamicDatasource(){
			DynamicDatasource dynamicDatasource = new DynamicDatasource();
			dynamicDatasource.setDefaultTargetDataSource(dataSource1());
			Map<Object, Object> map = new HashMap<>();
			map.put("ds1", dataSource1());
			map.put("ds2", dataSource2());
			dynamicDatasource.setTargetDataSources(map);
			return dynamicDatasource;
		}
	
		@Bean
		public PlatformTransactionManager transactionManager(){
			return new DataSourceTransactionManager(dynamicDatasource());
		}
	
		public static void main(String[] args) {
			SpringApplication.run(MyBlogApplication.class, args);
		}
	}
```


如果不想改变spring 的配置文件的数据库连接url，也可以先初始化一个DatasourceProperties的bean，然后利用这个spring 读取的配置文件将其中一些参数传递给HikariDataSource，如下：


```java
	@Bean(name="ds2Prop")
	 @ConfigurationProperties(prefix = "spring.datasource.db2")
	 public DataSourceProperties dataSource2(){
	 	return new DataSourceProperties();
	 }
	@Bean(name = "ds2")
	 public HikariDataSource hkds(){
	 	return dataSource2().initializeDataSourceBuilder().type(HikariDataSource.class).build();
	 }
```

总结：多数据源配置稍微麻烦一些，第一步建立一个线程安全的数据源标识符存放和切换的类，第二步是一个继承了AbstractRoutingDataSource的子类用来重写父类方法来获取自定义数据源标识，第三步是建立多个DataSource的Bean，以及动态切换数据源的spring bean，并将多个数据源放入目标数据源的map里，加入事物控制，最后建立切面，通过读取连接点的注解或者连接点的类上的注解，在前置通知里调用工具类进行切换标识。

