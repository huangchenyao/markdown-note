[TOC]

# Springboot启动详解

springboot项目的入口都是如下的启动类：

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

其中，`@SpringBootApplication`与`SpringApplication.run()`就是启动的核心代码。

## 一、@SpringBootApplication注解

`@SpringBootApplication`是Springboot的核心注解，实际上是一个组合注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
  ...
}
```

核心是3个注解：

- `@SpringBootConfiguration`，实际上是`@Configuration`
- `@EnableAutoConfiguration`
- `@ComponentScan`

`@SpringBootApplication`= (默认属性)`@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`

所以启动类也可以写成

```java
@Configuration
@EnableAutoConfiguration
@ComponentScan
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### 1. @Configuration

`@Configuration`标注在类上，相当于把该类作为spring的xml配置文件中的`<beans>`，作用为：配置spring容器(应用上下文)，SpringBoot社区推荐使用基于JavaConfig的配置形式，所以，这里的启动类标注了`@Configuration`之后，本身其实也是一个IoC容器的配置类。

xml配置与config配置的区别例子：

#### (1) 表达形式层面

基于XML的：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd"
       default-lazy-init="true">
    
</beans>
```
基于JavaConfig的：

```java
@Configuration
public class MockConfiguration{
    //bean定义
}
```

任何一个标注了`@Configuration`的Java类定义都是一个JavaConfig配置类。



#### (2) 注册bean定义层面

基于XML的：

```xml
<bean id="mockService" class="..MockServiceImpl">
    ...
```

基于JavaConfig的：

```java
@Configuration
public class MockConfiguration{
    @Bean
    public MockService mockService(){
        return new MockServiceImpl();
    }
}
```

任何一个标注了`@Bean`的方法，其返回值将作为一个bean定义注册到Spring的IoC容器，方法名将默认成该bean定义的id。



#### (3) 表达依赖注入关系层面

基于XML的：

```xml
<bean id="mockService" class="..MockServiceImpl">
   <propery name ="dependencyService" ref="dependencyService" />
</bean>
<bean id="dependencyService" class="DependencyServiceImpl"></bean>
```

基于JavaConfig的：

```java
@Configuration
public class MockConfiguration{
    @Bean
    public MockService mockService(){
        return new MockServiceImpl(dependencyService());
    }

    @Bean
    public DependencyService dependencyService(){
        return new DependencyServiceImpl();
    }
}
```

如果一个bean的定义依赖其他bean，则直接调用对应的JavaConfig类中依赖bean的创建方法就可以了。

提到`@Configuration`就要提到他的搭档`@Bean`。使用这两个注解就可以创建一个简单的spring配置类，可以用来替代相应的xml配置文件。

```xml
<beans> 
    <bean id = "car" class="com.test.Car"> 
        <property name="wheel" ref = "wheel">property> 
    <bean> 
    <bean id = "wheel" class="com.test.Wheel">bean> 
</beans>
```

相当于：

```java
@Configuration 
public class Conf { 
    @Bean 
    public Car car() { 
        Car car = new Car(); 
        car.setWheel(wheel()); 
        return car; 
    }

    @Bean 
    public Wheel wheel() { 
        return new Wheel(); 
    } 
}
```

`@Configuration`的注解类标识这个类可以使用Spring IoC容器作为bean定义的来源。

`@Bean`注解告诉Spring，一个带有@Bean的注解方法将返回一个对象，该对象应该被注册为在Spring应用程序上下文中的bean。



### 2. @ComponentScan

`@ComponentScan`的功能其实就是自动扫描并加载符合条件的组件（比如`@Component`和`@Repository`等）或者bean定义，最终将这些bean定义加载到IoC容器中。



### 3. @EnableAutoConfiguration

SpringBoot里面提供各种`@Enable`开头的注解，`@EnableAutoConfiguration`的理念和做事方式其实一脉相承，简单概括一下就是，借助`@Import`的支持，收集和注册特定场景相关的bean定义。

`@EnableAutoConfiguration`会根据类路径中的jar依赖为项目进行自动配置，如：添加了`spring-boot-starter-web`依赖，会自动添加Tomcat和Spring MVC的依赖，Spring Boot会对Tomcat和Spring MVC进行自动配置。

```java
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    ...
}
```

借助`EnableAutoConfigurationImportSelector`，`@EnableAutoConfiguration`可以帮助SpringBoot应用将所有符合条件的`@Configuration`配置都加载到当前SpringBoot创建并使用的IoC容器。就像一只“八爪鱼”一样，借助于Spring框架原有的一个工具类：`SpringFactoriesLoader`的支持，`@EnableAutoConfiguration`可以智能的自动配置功效才得以大功告成！

#### SpringFactoriesLoader

SpringFactoriesLoader属于Spring框架私有的一种扩展方案，其主要功能就是从指定的配置文件`META-INF/spring.factories`加载配置。

```java
public abstract class SpringFactoriesLoader {
    //...
    public static  List loadFactories(Class<T> factoryClass, ClassLoader classLoader) {
        ...
    }

    public static List loadFactoryNames(Class factoryClass, ClassLoader classLoader) {
        ....
    }
}
```

配合`@EnableAutoConfiguration`使用的话，它更多是提供一种配置查找的功能支持，即根据`@EnableAutoConfiguration`的完整类名`org.springframework.boot.autoconfigure.EnableAutoConfiguration`作为查找的Key，获取对应的一组`@Configuration`类。

所以，`@EnableAutoConfiguration`自动配置就变成了：从classpath中搜寻所有的`META-INF/spring.factories`配置文件，并将其中`org.springframework.boot.autoconfigure.EnableAutoConfiguration`对应的配置项通过反射实例化为对应的标注了`@Configuration`的JavaConfig形式的IoC容器配置类，然后汇总为一个并加载到IoC容器。

## 二、SpringApplication执行流程

SpringApplication的run方法的实现是我们本次旅程的主要线路，该方法的主要流程大体可以归纳如下：

1） 如果我们使用的是SpringApplication的静态run方法，那么，这个方法里面首先要创建一个SpringApplication对象实例，然后调用这个创建好的SpringApplication的实例方法。在SpringApplication实例初始化的时候，它会提前做几件事情：

- 根据classpath里面是否存在某个特征类`org.springframework.web.context.ConfigurableWebApplicationContext`来决定是否应该创建一个为Web应用使用的ApplicationContext类型。
- 使用`SpringFactoriesLoader`在应用的classpath中查找并加载所有可用的`ApplicationContextInitializer`。
- 使用`SpringFactoriesLoader`在应用的classpath中查找并加载所有可用的`ApplicationListener`。
- 推断并设置main方法的定义类。

2） SpringApplication实例初始化完成并且完成设置后，就开始执行run方法的逻辑了，方法执行伊始，首先遍历执行所有通过`SpringFactoriesLoader`可以查找到并加载的`SpringApplicationRunListener`。调用它们的`started()`方法，告诉这些`SpringApplicationRunListener`，“嘿，SpringBoot应用要开始执行咯！”。

3） 创建并配置当前Spring Boot应用将要使用的Environment（包括配置要使用的PropertySource以及Profile）。

4） 遍历调用所有`SpringApplicationRunListener`的`environmentPrepared()`的方法，告诉他们：“当前SpringBoot应用使用的Environment准备好了咯！”。

5） 如果SpringApplication的showBanner属性被设置为true，则打印banner。

6） 根据用户是否明确设置了`applicationContextClass`类型以及初始化阶段的推断结果，决定该为当前SpringBoot应用创建什么类型的`ApplicationContext`并创建完成，然后根据条件决定是否添加ShutdownHook，决定是否使用自定义的`BeanNameGenerator`，决定是否使用自定义的`ResourceLoader`，当然，最重要的，将之前准备好的Environment设置给创建好的`ApplicationContext`使用。

7） ApplicationContext创建好之后，SpringApplication会再次借助`Spring-FactoriesLoader`，查找并加载classpath中所有可用的`ApplicationContext-Initializer`，然后遍历调用这些`ApplicationContextInitializer`的`initialize`（applicationContext）方法来对已经创建好的`ApplicationContext`进行进一步的处理。

8） 遍历调用所有`SpringApplicationRunListener`的`contextPrepared()`方法。

9） 最核心的一步，将之前通过`@EnableAutoConfiguration`获取的所有配置以及其他形式的IoC容器配置加载到已经准备完毕的`ApplicationContext`。

10） 遍历调用所有`SpringApplicationRunListener`的`contextLoaded()`方法。

11） 调用`ApplicationContext`的`refresh()`方法，完成IoC容器可用的最后一道工序。

12） 查找当前`ApplicationContext`中是否注册有`CommandLineRunner`，如果有，则遍历执行它们。

13） 正常情况下，遍历执行`SpringApplicationRunListener`的`finished()`方法、（如果整个过程出现异常，则依然调用所有`SpringApplicationRunListener`的`finished()`方法，只不过这种情况下会将异常信息一并传入处理）

### 总览

SpringBoot启动流程主要分为三个部分：

- 第一部分进行SpringApplication的初始化模块，配置一些基本的环境变量、资源、构造器、监听器；
- 第二部分实现了应用具体的启动方案，包括启动流程的监听模块、加载配置环境模块、及核心的创建上下文环境模块；
- 第三部分是自动化配置模块，该模块作为springboot自动配置核心，在后面的分析中会详细讨论。在下面的启动程序中我们会串联起结构中的主要功能。

### 启动

每个SpringBoot程序都有一个主入口，也就是main方法，main里面调用`SpringApplication.run()`启动整个spring-boot程序，该方法所在类需要使用`@SpringBootApplication`注解，以及`@ImportResource`注解(if need)，`@SpringBootApplication`包括三个注解，功能如下：

- `@EnableAutoConfiguration`：SpringBoot根据应用所声明的依赖来对Spring框架进行自动配置。
- `@SpringBootConfiguration`(内部为`@Configuration`)：被标注的类等于在spring的XML配置文件中(`applicationContext.xml`)，装配所有bean事务，提供了一个spring的上下文环境。
- `@ComponentScan`：组件扫描，可自动发现和装配Bean，默认扫描SpringApplication的run方法里的`Booter.class`所在的包路径下文件，所以最好将该启动类放到根包路径下。

### SpringBoot启动类

进入run方法

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
  return new SpringApplication(primarySources).run(args);
}
```

run方法中去创建了一个SpringApplication实例，并运行，

```java
public SpringApplication(Class<?>... primarySources) {
  this(null, primarySources);
}

@SuppressWarnings({ "unchecked", "rawtypes" })
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
  this.resourceLoader = resourceLoader;
  Assert.notNull(primarySources, "PrimarySources must not be null");
  this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
  this.webApplicationType = WebApplicationType.deduceFromClasspath();
  setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
  setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
  this.mainApplicationClass = deduceMainApplicationClass();
}
```

这里主要是为SpringApplication对象赋一些初值。构造函数执行完毕后，我们回到run方法，

```java
public ConfigurableApplicationContext run(String... args) {
  StopWatch stopWatch = new StopWatch();
  stopWatch.start();
  ConfigurableApplicationContext context = null;
  Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
  configureHeadlessProperty();
  SpringApplicationRunListeners listeners = getRunListeners(args);
  listeners.starting();
  try {
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
    ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
    configureIgnoreBeanInfo(environment);
    Banner printedBanner = printBanner(environment);
    context = createApplicationContext();
    exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                                                     new Class[] { ConfigurableApplicationContext.class }, context);
    prepareContext(context, environment, listeners, applicationArguments, printedBanner);
    refreshContext(context);
    afterRefresh(context, applicationArguments);
    stopWatch.stop();
    if (this.logStartupInfo) {
      new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
    }
    listeners.started(context);
    callRunners(context, applicationArguments);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, listeners);
    throw new IllegalStateException(ex);
  }

  try {
    listeners.running(context);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, null);
    throw new IllegalStateException(ex);
  }
  return context;
}
```

#### 该方法中实现了如下几个关键步骤：

1. 创建了应用的监听器`SpringApplicationRunListeners`并开始监听

2. 加载SpringBoot配置环境(`ConfigurableEnvironment`)，如果是通过web容器发布，会加载`StandardEnvironment`，其最终也是继承了`ConfigurableEnvironment`

3. 配置环境(`Environment`)加入到监听器对象中(`SpringApplicationRunListeners`)

4. 创建run方法的返回对象：`ConfigurableApplicationContext`(应用配置上下文)

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

方法会先获取显式设置的应用上下文(`applicationContextClass`)，如果不存在，再加载默认的环境配置（通过是否是`web environment`判断），默认选择`AnnotationConfigApplicationContext`注解上下文（通过扫描所有注解类来加载bean），最后通过BeanUtils实例化上下文对象，并返回。

5. 回到run方法内，prepareContext方法将`listeners、environment、applicationArguments、banner`等重要组件与上下文对象关联

6. 接下来的`refreshContext(context)`方法(初始化方法如下)将是实现`spring-boot-starter-*`(mybatis、redis等)自动化配置的关键，包括`spring.factories`的加载，bean的实例化等核心工作。

```java
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
```

配置结束后，Springboot做了一些基本的收尾工作，返回了应用环境上下文。回顾整体流程，Springboot的启动，主要创建了配置环境(environment)、事件监听(listeners)、应用上下文(applicationContext)，并基于以上条件，在容器中开始实例化我们需要的Bean。