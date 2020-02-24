# SpringBoot内置服务器启动流程

SpringBoot可以直接把web程序达成jar包，直接启动，这就得益于SpringBoot内置了容器，可以直接启动。

以Tomcat为例，来看看SpringBoot是如何启动Tomcat的。

```java
public ConfigurableApplicationContext run(String... args) {
  StopWatch stopWatch = new StopWatch();
  stopWatch.start();
  ConfigurableApplicationContext context = null;
  Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
  // 设置系统属性java.awt.headless，为true则启用headless模式支持
  configureHeadlessProperty();
  // 通过SpringFactoriesLoader检索META-INF/spring.factories，
  // 找到声明的所有SpringApplicationRunListener的实现类并将其实例化，
  // 之后逐个调用其started()方法，广播SpringBoot要开始执行了
  SpringApplicationRunListeners listeners = getRunListeners(args);
  // 发布应用开始启动事件
	listeners.starting();
  try {
    // 初始化参数
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
    // 创建并配置当前SpringBoot应用将要使用的Environment（包括配置要使用的PropertySource以及Profile）
    // 并遍历调用所有的SpringApplicationRunListener的environmentPrepared()方法，广播Environment准备完毕。
    ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
    configureIgnoreBeanInfo(environment);
    // 打印banner
    Banner printedBanner = printBanner(environment);
    // 创建应用上下文
    context = createApplicationContext();
    // 通过SpringFactoriesLoader检索META-INF/spring.factories，获取并实例化异常分析器
    exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                                                     new Class[] { ConfigurableApplicationContext.class }, context);
    // 为ApplicationContext加载environment，之后逐个执行ApplicationContextInitializer的initialize()方法来进一步封装ApplicationContext
    // 并调用所有的SpringApplicationRunListener的contextPrepared()方法，【EventPublishingRunListener只提供了一个空的contextPrepared()方法】
    // 之后初始化IoC容器，并调用SpringApplicationRunListener的contextLoaded()方法，广播ApplicationContext的IoC加载完成
    // 这里就包括通过@EnableAutoConfiguration导入的各种自动配置类。
    prepareContext(context, environment, listeners, applicationArguments, printedBanner);
    // 刷新上下文
    refreshContext(context);
    // 再一次刷新上下文,其实是空方法，可能是为了后续扩展
    afterRefresh(context, applicationArguments);
    stopWatch.stop();
    if (this.logStartupInfo) {
      new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
    }
    // 发布应用已经启动的事件
    listeners.started(context);
    // 遍历所有注册的ApplicationRunner和CommandLineRunner，并执行其run()方法。
    // 我们可以实现自己的ApplicationRunner或者CommandLineRunner，来对SpringBoot的启动过程进行扩展。
    callRunners(context, applicationArguments);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, listeners);
    throw new IllegalStateException(ex);
  }

  try {
    // 应用已经启动完成的监听事件
    listeners.running(context);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, null);
    throw new IllegalStateException(ex);
  }
  return context;
}
```

#### 其实这个方法我们可以简单的总结下步骤为 ：

1. 配置属性
2. 获取监听器，发布应用开始启动事件
3. 初始化输入参数
4. 配置环境，输出banner
5. 创建上下文 
6. 预处理上下文
7. 刷新上下文 
8. 再刷新上下文
9. 发布应用已经启动事件 
10. 发布应用启动完成事件

其实上面这段代码，如果只要分析tomcat内容的话，只需要关注两个内容即可，上下文是如何创建的，上下文是如何刷新的，分别对应的方法就是`createApplicationContext()` 和`refreshContext(context)`，接下来我们来看看这两个方法做了什么。

```java
protected ConfigurableApplicationContext createApplicationContext() {
  Class<?> contextClass = this.applicationContextClass;
  if (contextClass == null) {
    try {
      switch (this.webApplicationType) {
        case SERVLET:
          contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
          break;
        case REACTIVE:
          contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
          break;
        default:
          contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
      }
    }
    catch (ClassNotFoundException ex) {
      throw new IllegalStateException(
        "Unable create a default ApplicationContext, please specify an ApplicationContextClass", ex);
    }
  }
  return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

这里就是根据我们的`webApplicationType` 来判断创建哪种类型的Servlet,代码中分别对应着Web类型(SERVLET),响应式Web类型（REACTIVE),非Web类型（default),我们建立的是Web类型，所以肯定实例化 `DEFAULT_SERVLET_WEB_CONTEXT_CLASS`指定的类，也就是`AnnotationConfigServletWebServerApplicationContext`类

![image-20200221192244657](/Users/huangchenyao/Documents/学习/markdown-note/Java/SpringBoot内置服务器/AnnotationConfigServletWebServerApplicationContext继承关系.png)

刷新上下文方法：
```java
private void refreshContext(ConfigurableApplicationContext context) {
  refresh(context);
  if (this.registerShutdownHook) {
    try {
      context.registerShutdownHook();
    }
    catch (AccessControlException ex) {
      // Not allowed in some environments.
    }
  }
}

protected void refresh(ApplicationContext applicationContext) {
  Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
  ((AbstractApplicationContext) applicationContext).refresh();
}
```

强转为父类`AbstractApplicationContext`执行`refresh()`方法：

```java
public void refresh() throws BeansException, IllegalStateException {
  synchronized (this.startupShutdownMonitor) {
    // Prepare this context for refreshing.
    prepareRefresh();

    // Tell the subclass to refresh the internal bean factory.
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // Prepare the bean factory for use in this context.
    prepareBeanFactory(beanFactory);

    try {
      // Allows post-processing of the bean factory in context subclasses.
      postProcessBeanFactory(beanFactory);

      // Invoke factory processors registered as beans in the context.
      invokeBeanFactoryPostProcessors(beanFactory);

      // Register bean processors that intercept bean creation.
      registerBeanPostProcessors(beanFactory);

      // Initialize message source for this context.
      initMessageSource();

      // Initialize event multicaster for this context.
      initApplicationEventMulticaster();

      // Initialize other special beans in specific context subclasses.
      onRefresh();

      // Check for listener beans and register them.
      registerListeners();

      // Instantiate all remaining (non-lazy-init) singletons.
      finishBeanFactoryInitialization(beanFactory);

      // Last step: publish corresponding event.
      finishRefresh();
    }

    catch (BeansException ex) {
      if (logger.isWarnEnabled()) {
        logger.warn("Exception encountered during context initialization - " +
                    "cancelling refresh attempt: " + ex);
      }

      // Destroy already created singletons to avoid dangling resources.
      destroyBeans();

      // Reset 'active' flag.
      cancelRefresh(ex);

      // Propagate exception to caller.
      throw ex;
    }

    finally {
      // Reset common introspection caches in Spring's core, since we
      // might not ever need metadata for singleton beans anymore...
      resetCommonCaches();
    }
  }
}
```

这里`onRefresh()`方法是调用其子类的实现，根据我们上文的分析，我们这里的子类是`ServletWebServerApplicationContext`

```java
@Override
protected void onRefresh() {
  super.onRefresh();
  try {
    createWebServer();
  }
  catch (Throwable ex) {
    throw new ApplicationContextException("Unable to start web server", ex);
  }
}

private void createWebServer() {
  WebServer webServer = this.webServer;
  ServletContext servletContext = getServletContext();
  if (webServer == null && servletContext == null) {
    ServletWebServerFactory factory = getWebServerFactory();
    this.webServer = factory.getWebServer(getSelfInitializer());
  }
  else if (servletContext != null) {
    try {
      getSelfInitializer().onStartup(servletContext);
    }
    catch (ServletException ex) {
      throw new ApplicationContextException("Cannot initialize servlet context", ex);
    }
  }
  initPropertySources();
}
```

`createWebServer()`就是启动web服务，但是还没有真正启动，`webServer`是通过`ServletWebServerFactory`来获取的

![image-20200224172349136](/Users/huangchenyao/Documents/学习/markdown-note/Java/SpringBoot内置服务器/ServletWebServerFactory继承.png)

tomcat的实现

```java
@Override
public WebServer getWebServer(ServletContextInitializer... initializers) {
  if (this.disableMBeanRegistry) {
    Registry.disableRegistry();
  }
  Tomcat tomcat = new Tomcat();
  File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
  tomcat.setBaseDir(baseDir.getAbsolutePath());
  Connector connector = new Connector(this.protocol);
  connector.setThrowOnFailure(true);
  tomcat.getService().addConnector(connector);
  customizeConnector(connector);
  tomcat.setConnector(connector);
  tomcat.getHost().setAutoDeploy(false);
  configureEngine(tomcat.getEngine());
  for (Connector additionalConnector : this.additionalTomcatConnectors) {
    tomcat.getService().addConnector(additionalConnector);
  }
  prepareContext(tomcat.getHost(), initializers);
  return getTomcatWebServer(tomcat);
}
```

根据上面的代码，我们发现其主要做了两件事情，第一件事就是把Connnctor(我们称之为连接器)对象添加到Tomcat中，第二件事就是`configureEngine`，那这个`Engine`是什么呢？我们查看`tomcat.getEngine()`的源码：

```java
public Engine getEngine() {
  Service service = this.getServer().findServices()[0];
  if (service.getContainer() != null) {
    return service.getContainer();
  } else {
    Engine engine = new StandardEngine();
    engine.setName("Tomcat");
    engine.setDefaultHost(this.hostname);
    engine.setRealm(this.createDefaultRealm());
    service.setContainer(engine);
    return engine;
  }
}
```

根据上面的源码，我们发现，原来这个Engine是容器，我们继续跟踪源码，找到`Container`接口

![image-20200224173414024](/Users/huangchenyao/Documents/学习/markdown-note/Java/SpringBoot内置服务器/Container接口继承.png)

上图中，我们看到了4个子接口，分别是`Engine`，`Host`，`Context`，`Wrapper`。`Engine`是最高级别的容器，其子容器是`Host`，`Host`的子容器是`Context`，`Context`的子容器是`Wrapper`，所以这4个容器的关系就是父子关系，也就是`Engine`>`Host`>`Context`>`Wrapper`。

#### 总结

SpringBoot的启动是通过`new SpringApplication()`实例来启动的，启动过程主要做如下几件事情：> 1. 配置属性 > 2. 获取监听器，发布应用开始启动事件 > 3. 初始化输入参数 > 4. 配置环境，输出banner > 5. 创建上下文 > 6. 预处理上下文 > 7. 刷新上下文 > 8. 再刷新上下文 > 9. 发布应用已经启动事件 > 10. 发布应用启动完成事件

而启动Tomcat就是在第7步中“刷新上下文”；Tomcat的启动主要是初始化2个核心组件，连接器(Connector)和容器（Container），一个Tomcat实例就是一个Server，一个Server包含多个Service，也就是多个应用程序，每个Service包含多个连接器（Connetor）和一个容器（Container),而容器下又有多个子容器，按照父子关系分别为：Engine,Host,Context,Wrapper，其中除了Engine外，其余的容器都是可以有多个。