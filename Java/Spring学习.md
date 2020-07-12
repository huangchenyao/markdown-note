

[TOC]

# Spring学习

## Spring框架划分

![Spring 框架图示](/Users/huangchenyao/Documents/markdown-note/Java/Spring学习.assets/spring_framework.gif)

![spring overview](/Users/huangchenyao/Documents/markdown-note/Java/Spring学习.assets/spring-overview.png)

Spring 框架是一个分层架构，由 7 个定义良好的模块组成。Spring 模块构建在核心容器之上，核心容器定义了创建、配置和管理 bean 的方式。

组成 Spring 框架的每个模块（或组件）都可以单独存在，或者与其他一个或多个模块联合实现。每个模块的功能如下：

- Spring Core：框架的最基础部分，提供 IoC 容器，对 bean 进行管理。

- Spring Context：继承BeanFactory，提供上下文信息，扩展出JNDI、EJB、电子邮件、国际化等功能。

- Spring DAO：提供了JDBC的抽象层，还提供了声明性事务管理方法。

- Spring ORM：提供了JPA、JDO、Hibernate、MyBatis 等ORM映射层。

- Spring AOP：集成了所有AOP功能。

- Spring Web：提供了基础的 Web 开发的上下文信息，现有的Web框架，如JSF、Tapestry、Structs等，提供了集成。

- Spring Web MVC：提供了 Web 应用的 Model-View-Controller 全功能实现。

  

最新官方文档的划分：

| [Overview](https://docs.spring.io/spring/docs/5.3.0-SNAPSHOT/spring-framework-reference/overview.html#overview) | history, design philosophy, feedback, getting started.       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [Core](https://docs.spring.io/spring/docs/5.3.0-SNAPSHOT/spring-framework-reference/core.html#spring-core) | IoC Container, Events, Resources, i18n, Validation, Data Binding, Type Conversion, SpEL, AOP. |
| [Testing](https://docs.spring.io/spring/docs/5.3.0-SNAPSHOT/spring-framework-reference/testing.html#testing) | Mock Objects, TestContext Framework, Spring MVC Test, WebTestClient. |
| [Data Access](https://docs.spring.io/spring/docs/5.3.0-SNAPSHOT/spring-framework-reference/data-access.html#spring-data-tier) | Transactions, DAO Support, JDBC, O/R Mapping, XML Marshalling. |
| [Web Servlet](https://docs.spring.io/spring/docs/5.3.0-SNAPSHOT/spring-framework-reference/web.html#spring-web) | Spring MVC, WebSocket, SockJS, STOMP Messaging.              |
| [Web Reactive](https://docs.spring.io/spring/docs/5.3.0-SNAPSHOT/spring-framework-reference/web-reactive.html#spring-webflux) | Spring WebFlux, WebClient, WebSocket.                        |
| [Integration](https://docs.spring.io/spring/docs/5.3.0-SNAPSHOT/spring-framework-reference/integration.html#spring-integration) | Remoting, JMS, JCA, JMX, Email, Tasks, Scheduling, Caching.  |
| [Languages](https://docs.spring.io/spring/docs/5.3.0-SNAPSHOT/spring-framework-reference/languages.html#languages) | Kotlin, Groovy, Dynamic Languages.                           |



## Core

### IoC Inverse of Control（控制反转）

一种设计思想，就是将原本在程序中手动创建对象的控制权，交由Spring框架来管理。

IoC两种方式：

- 依赖查找(Dependency Lookup)：容器中的受控对象通过容器的API来查找自己所依赖的资源和协作对象。
- 依赖注入(Dependency Injection)：由容器动态地将某种依赖关系的目标对象实例注入到应用系统中的各个关联的组件之中。

Spring IoC 容器的设计主要是基于以下两个接口：

- **BeanFactory**
- **ApplicationContext**

其中 ApplicationContext 是 BeanFactory 的子接口之一，换句话说：**BeanFactory 是 Spring IoC 容器所定义的最底层接口，**而 ApplicationContext 是其最高级接口之一，并对 BeanFactory 功能做了许多的扩展，所以在**绝大部分的工作场景下**，都会使用 ApplicationContext 作为 Spring IoC 容器。

#### Spring IoC 的容器的初始化和依赖注入

虽然 Spring IoC 容器的生成十分的复杂，但是大体了解一下 Spring IoC 初始化的过程还是必要的。这对于理解 Spring 的一系列行为是很有帮助的。

**注意：**Bean 的定义和初始化在 Spring IoC 容器是两大步骤，它是先定义，然后初始化和依赖注入的。

Bean 的定义分为 3 步：

1. Resource 定位
   Spring IoC 容器先根据开发者的配置，进行资源的定位，在 Spring 的开发中，通过 XML 或者注解都是十分常见的方式，定位的内容是由开发者提供的。
2. BeanDefinition 的载入
   这个时候只是将 Resource 定位到的信息，保存到 Bean 定义（BeanDefinition）中，此时并不会创建 Bean 的实例。
3. BeanDefinition 的注册
   这个过程就是将 BeanDefinition 的信息发布到 Spring IoC 容器中
   **注意：**此时仍然没有对应的 Bean 的实例。

做完了以上 3 步，Bean 就在 Spring IoC 容器中被定义了，而没有被初始化，更没有完成依赖注入，也就是没有注入其配置的资源给 Bean，那么它还不能完全使用。

对于初始化和依赖注入，Spring Bean 还有一个配置选项——**【lazy-init】**，其含义就是**是否初始化 Spring Bean**。在没有任何配置的情况下，它的默认值为 default，实际值为 false，也就是 **Spring IoC 默认会自动初始化 Bean**。如果将其设置为 true，那么只有当我们使用 Spring IoC 容器的 getBean 方法获取它时，它才会进行 Bean 的初始化，完成依赖注入。

#### IoC 是如何实现的

最后我们简单说说IoC是如何实现的。想象一下如果我们自己来实现这个依赖注入的功能，我们怎么来做？ 无外乎：

1. 读取标注或者配置文件，看看Target依赖的是哪个Source，拿到类名
2. 使用反射的API，基于类名实例化对应的对象实例
3. 将对象实例，通过构造函数或者setter，传递给 Target

我们发现其实自己来实现也不是很难，Spring实际也就是这么做的。这么看的话其实IoC就是一个工厂模式的升级版。3

