

[TOC]

# Spring学习

## Spring框架

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

  

