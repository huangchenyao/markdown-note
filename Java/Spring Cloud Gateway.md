[TOC]

# Spring Cloud Gateway

## 网关对比

### Zuul

Zuul是Netflix开源的微服务网关。

#### zuul 1.x

![Image for post](/Users/huangchenyao/Documents/markdown-note/Java/Spring Cloud Gateway.assets/0*kPzgZrACokyPJJfy.png)

zuul 1.x是基于servlet框架的，所以系统是阻塞多线程模型。简单来说，对于每个请求，servlet容器要为该请求分配一个线程专门负责处理这个请求，直到响应返回客户端这个线程才会被释放返回容器线程池。如果后台服务调用比较耗时，那么这个线程就会被阻塞，阻塞期间线程资源被占用，不能干其它事情。我们知道Servlet容器线程池的大小是有限制的，当前端请求量大，而后台慢服务比较多时，很容易耗尽容器线程池内的线程，造成容器无法接受新的请求，Netflix为此还专门研发了Hystrix熔断组件来解决慢服务耗尽资源问题。

优点：

- 编程模型简单
- 开发调试运维简单

缺点：

- 线程上下文切换开销
- 连接数限制
- 延迟阻塞耗尽线程连接资源

适用：计算密集型（CPU多）



#### Zuul 2.x

![Image for post](/Users/huangchenyao/Documents/markdown-note/Java/Spring Cloud Gateway.assets/0*jrG2ldEVRRJcgpkj.png)

zuul 2.x采用了Netty实现异步非阻塞编程模型。

一般异步模式的本质都是使用队列Queue(或称总线Bus)，在上图中，你可以简单理解为前端有一个队列专门负责处理用户请求，后端有个队列专门负责处理后台服务调用，中间有个事件环线程(Event Loop Thread)，它同时监听前后两个队列上的事件，有事件就触发回调函数处理事件。这种模式下需要的线程比较少，基本上每个CPU核上只需要一个事件环处理线程，前端的连接数可以很多，连接来了只需要进队列，不需要启动线程，事件环线程由事件触发，没有多线程阻塞问题。

优点：

- 线程开销小
- 连接数容易扩展

缺点：

- 开发调试比较复杂
- 开发模型复杂

适用：I/O密集型



### Spring Cloud Gateway

在Spring Cloud Gateway出现之前，网关基本都是使用Zuul 1.x。但是由于 Zuul 1.x 存在的一些问题，比如阻塞式的 API，不支持 WebSocket 等，一直被人所诟病，而且 Zuul 升级新版本依赖于 Netflix 公司，经过几次跳票之后，Spring 开源社区决定推出自己的网关组件，替代 Netflix Zuul。

SpringCloudGateway是Spring官方基于Spring 5.0，Spring Boot 2.0和Project Reactor等技术开发的网关，Spring云网关旨在提供一种简单而有效的路由API的方法。

### 对比

|                        | Gateway    | Zuul 1.x    | Zuul 2.x   |
| :--------------------: | ---------- | ----------- | ---------- |
|        开源时间        | 2017年7月  | 2013年6月   | 2017年11月 |
|          架构          | 基于netty  | 基于servlet | 基于netty  |
|        运行方式        | 异步非阻塞 | 同步阻塞    | 异步非阻塞 |
|    Spring Cloud集成    | √          | √           | ×          |
| 支持长连接、Web Socket | √          | ×           | √          |
|       限流、限速       | √          | 第三方      | 即将推出   |
|          监控          | √          | √           | √          |
|        上手难度        | 中等       | 简单        | 复杂       |



## Spring Cloud Gateway使用

```xml
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.1.4.RELEASE</version>
	<relativePath/>
</parent>

<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-dependencies</artifactId>
			<version>Greenwich.SR6</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>
```





## Spring Cloud Gateway限流

