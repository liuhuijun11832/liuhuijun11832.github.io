---
title: 深入Tomcat/Jetty-抽象容器
categories: 编程技术
date: 2019-07-19 14:51:31
tags:
- Tomcat
- Jetty
keywords: [Tomcat,Jetty]
description: 深入学习Tomca和Jetty
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

执行当前容器里的`backgroundProcess()`，并且递归调用子容器的`backgroundProcess()`，如果子容器的backgroundProcessorDelay大于0，表明子容器有自己的线程，所以此时就不用父类来调用；

这个方法是接口中的默认方法，所以顶层engine启动后台线程以后，它会启动顶层engine以及子engine的周期性任务。

由于ContainerBase是所有组件基类，所以其他组件都可以由自己的周期性任务。

<!--more-->

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

# Servlet管理

Wrapper：核心：变量`Field Servlet`，方法`Method loadServlet`用来创建servlet并初始化（Servlet规范要求）。

执行时期：延迟加载，用到时才会加载该Servlet，除非在web.xml里设置了loadOnStartUp=true，但是Wrapper会在Tomcat启动时就会被创建。

机制：Pipeline-Valve链，每一个容器都有一个BasicValve，Wrapper的BasicValve为StandardWrapperValve。

步骤：

1. 创建Servlet实例；
2. 给当前请求创建一个Filter链；
3. 调用这个Filter链，链中的最后一个Filter会调用Servlet。

# Filter管理

可以在web.xml里配置，所以Filter的实例是在Context容器中管理的，本质上是用一个HashMap来保存Filter。

Filter生命周期很短，是和每次请求对应的，请求结束这个Filter链就结束了。

核心：ApplicationFilterChain，每个链包含了末尾的链所要调用的Servlet。

最后会调用servlet的service方法。

本质上和Pipeline-Valve都是一样的责任链模式，但是它的实现方式是：**FilterChain有doFilter方法，并且每一个Filter首先调用FilterChain的doFilter方法，由于Filter链中会保存当前Filter所在的位置，这样就会调用下一个Filter的doFilter方法，形成了一个链式调用。**

# Listener管理

可以在web.xml里配置，所以Listener也是在Context容器中管理的。Listener可以监听容器内生命状态的变化（比如Context容器的启动和停止，Session的创建和销毁），或者Context、Session的某个属性变了或者新的请求来了等等。

```java
// 监听属性值变化的监听器,每个请求的属性可能不一样，所以要保证线程安全，并且写多读少
private List<Object> applicationEventListenersList = new CopyOnWriteArrayList<>();

// 监听生命事件的监听器，Context容器的启动和停止是能保证线程安全，所以这里直接用数组
private Object applicationLifecycleListenersObjects[] = new Object[0];

```

Tomcat定义了ServletContextListener接口作为第三方扩展，在启动的时候遍历所有该类型的监听器触发事件，Spring就是实现了这个接口，监听Context的启停事件，和LifecycleListener不同的是，LifecycleListener定义在生命周期管理组件中，由基类LifecycleBase统一管理。

# 异步Servlet

用法：`@WebServlet(urlPatterns = {"/async"}, asyncSupported = true)`；

异步默认超时时长：30s；

原理：req.startAsync和ctx.complete，前者创建一个异步上下文AsyncContext对象，用来保存request和response等上下文信息；

CoyoteAdapter：flush数据把响应发回浏览器；如果是异步请求，就设置一个异步标志为truel，并在随后的ProtocolHandler判断该标志，如果是一个异步请求，那么它会把当前的Socket的协议处理者Processor缓存起来，将SocketWrapper对象响应的Processor存到一个Map数据结构里。

ctx.complete：调用request的action方法，通知连接器这个请求处理完了---->传入操作码processSocketEvent---->processSocket---->创建SocketProcess任务类---->交给Tomcat线程池处理。

适合场景：I/O密集型业务。

# 内嵌式的Tomcat

以Spring Boot为例，它抽象了WebServer接口，提供给Tomcat和Jetty去实现：

```java
public interface WebServer {
    void start() throws WebServerException;
    void stop() throws WebServerException;
    int getPort();
}
```

同时还提供了ServletWebServerFactory来创建容器，即返回WebServer。

```java
public interface ServletWebServerFactory {
    WebServer getWebServer(ServletContextInitializer... initializers);
}
```

其中`ServletContextInitializer`就是ServletContext的初始化器，并且在getWebServer方法中会调用onStartUp方法，所以如果想在Servlet容器中注册自己的Servlet，可以实现该接口。

WebServletFactoryCutomizerBeanPostProcessor：这是一个BeanPostProcessor，在postProcessBeforeInitialization过程中寻找WebServerFactoryCustomizer类型的Bean，并依次调用WebServerFactoryCustomizer接口的customize方法做一些定制化。

## 启动过程

Spring核心：ApplicationContext；

抽象类：AbstractApplicationContext；

方法：refresh()或者onRefresh()；

原理：ServletWebServerApplicationContext重写了onRefresh方法创建内嵌式Web容器；

```java
public WebServer getWebServer(ServletContextInitializer... initializers) {
    //1. 实例化一个 Tomcat，可以理解为 Server 组件。
    Tomcat tomcat = new Tomcat();
    
    //2. 创建一个临时目录
    File baseDir = this.baseDirectory != null ? this.baseDirectory : this.createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    
    //3. 初始化各种组件
    Connector connector = new Connector(this.protocol);
    tomcat.getService().addConnector(connector);
    this.customizeConnector(connector);
    tomcat.setConnector(connector);
    tomcat.getHost().setAutoDeploy(false);
    this.configureEngine(tomcat.getEngine());
    
    //4. 创建定制版的 "Context" 组件。
    this.prepareContext(tomcat.getHost(), initializers);
    return this.getTomcatWebServer(tomcat);
}
```

## 注册Servlet的方式

### 注解式

Spring Boot 的配置类需要开启@ServletComponentScan用于扫描@WebServlet、@WebFilter、@WebListener等。

### Java Config

在Spring的配置类中加入Bean：

```java
@Bean
public ServletRegistrationBean servletRegistrationBean() {
    return new ServletRegistrationBean(new HelloServlet(),"/hello");
}
```

### 动态注册

实现前文提到的上下文初始化类：

```java
@Component
public class MyServletRegister implements ServletContextInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {
    
        //Servlet 3.0 规范新的 API，动态注册新的Servlet
        ServletRegistration myServlet = servletContext
                .addServlet("HelloServlet", HelloServlet.class);
                
        myServlet.addMapping("/hello");
        
        myServlet.setInitParameter("name", "Hello Servlet");
    }

}
```

> 这里需要注意的是，其实ServletRegistrationBean也是通过实现ServletContextInitializer来实现的，会交给Spring来管理。而ServletContainerInitializer的实现类是被Tomcat管理的。

## Web容器定制

在Spring Boot 2.0中，可以通过两种方式定制Web容器：

- 通过Web容器工程ConfigurableServletWebServerFactory来定制参数：

```java
@Component
public class MyGeneralCustomizer implements
  WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {
  
    public void customize(ConfigurableServletWebServerFactory factory) {
        factory.setPort(8081);
        factory.setContextPath("/hello");
     }
}
```

- 通过特定web容器的工厂，比如TomcatServletWebServerFactory来进一步定制：

```java
@Component
public class MyTomcatCustomizer implements
        WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.setPort(8081);
        factory.setContextPath("/hello");
        factory.addEngineValves(new TraceValve() );

    }
}
//实现追踪分布式项目中的路径
class TraceValve extends ValveBase {
    @Override
    public void invoke(Request request, Response response) throws IOException, ServletException {

        request.getCoyoteRequest().getMimeHeaders().
        addValue("traceid").setString("1234xxxxabcd");

        Valve next = getNext();
        if (null == next) {
            return;
        }

        next.invoke(request, response);
    }

}
```

# Jetty-HandlerWrapper

Jetty通过HndlerWrapper实现责任链。

`WebAppContext` -> `SessionHandler` ->` SecurityHandler` ->` ServletHandler`。

核心：`protected Handler _handler`，持有下一个Hadnler的引用，并且会在handle方法里执行下一个Handler。

ScopeHandler：核心Handler，被间接或者直接地继承，`_handler`是持有的下一个Handler的引用，并且会在handler方法里调用下一个Handler；

* `_outerScope`：根据它是否为null来判断使用`doScope()`还是`doHandler()`，头节点肯定为null，其他节点的该字段肯定指向Handler链中头节点，言下之意---如果是头节点，就执行doScope，如不是头节点，执行doHandler；
* `__outerScope`：使用`ThreadLocal<ScopeHandler>`进行包装，在需要时取出赋值给`_outerScope`，由于一般不能在上下文作为参数传递，所以这里作为线程私有变量；
* `_nextScope`：表示下一个`ScopeHandler`，和`_handler`区别在于，`_handler`的下一位可能是Wrapperx，而`_nextScope`表示下一个必须是`ScopeHandler`。

通过这几个参数，保证让ScopeHandler链上的doScope方法在doHandle、handle方法之前执行，并且保证不同ScopeHandler的doScope都是按照它在链上的先后顺序执行。

ContextHandler：ScopeHandler的子类，类似于Tomcat中的Context组件，对应一个Web应用，功能是给Servlet的执行维护一个上下文环境，并且将请求转发到相应的Servlet，doHandler里做了请求的修正，类加载器设置，以及调用nextScope。

# Spring框架中的设计模式

简单工厂：`interface BeanFactory`，使用方式：`beanFatory.get("userService")`；

工厂方法：`interface FactoryBean`，使用方式：定义一个类`UserFactory`实现`FactoryBean`，那`UserFactory`所产生的实例就都是User了；

单例模式：`private final Map<String,Object> singletonObjects = new ConcurrentHashMap<String,Object>;`先到HashMap中获取对象，如果没有拿到，则通过`Class.forName(String)`反射创建一个实例并添加到`singletonObjects`里。

代理模式：

* 抽象接口：代理角色和被代理角色都要实现该接口；
* 目标对象：被代理的对象，用于实现业务；
* 代理对象：内部含有对目标对象的引用，在执行目标对象的前后执行一部分逻辑。

静态代理：

```java
//抽象接口
public interface IStudentDao(){
    void save();
}
//目标对象
public class StudentDao implements IStudentDao(){
	public void save(){
		System.out.println("保存成功");
	}
}
//代理对象
public class StudentDaoProxy implements IStudentDao(){
	private IStudentDao target;
	public StudentDaoProxy(IStudentDao target){
		this.target = target;
	}
	public void save(){
		System.out.pritln("我增强了某某某");
		target.save();
		System.out.pritln("增强结束了");
	}
}
```

Spring Aop采用的是动态代理：

```java
//代理对象不是自己生成，而是由InvocationHandler生成和管理
public class MyInvocationHandler implements InvocationHandler {

    private Object object;

    public MyInvocationHandler(Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("开始事务");
        Object result = method.invoke(object, args);
        System.out.println("结束事务");
        return result;
    }
}
//测试
public static void main(String[] args) {
        IStudentDao stuDao = new StudentDao();
        InvocationHandler handler = new MyInvocationHandler(stuDao);
        IStudentDao studentDao = (IStudentDao) Proxy.newProxyInstance(stuDao.getClass().getClassLoader(), stuDao.getClass().getInterfaces(), handler);
        studentDao.save();
    }
```

> Spring Aop有两种代理方式：JDK（`JdkDynamicAopProxy implements InvocationHandler`）；Cglib（`CglibAopProxy`）。

```java
JDK 8
class Proxy:

 private static final Class<?>[] constructorParams =
        { InvocationHandler.class };
//newProxyInstance：
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)
        throws IllegalArgumentException {
    //通过ProxyClassFactory调用ProxyGenerator生成了代理类
    Class<?> cl = getProxyClass0(loader, intfs);
    //找到参数为InvocationHandler.class的构造函数
    final Constructor<?> cons = cl.getConstructor(constructorParams);
    //创建代理类实例
    return cons.newInstance(new Object[]{h});
}


//在ProxyGenerator类中：
public static byte[] generateProxyClass(final String name,Class<?>[] interfaces, int accessFlags)){}
private byte[] generateClassFile() {}
//根据接口生成实现类的字节码文件
```

而Cglib使用的是字节码拼接，不依赖接口，更强大更灵活。