[TOC]

# ACS 240s超时问题分析

## TCP原理

### 三次握手

![三次握手](/Users/huangchenyao/Documents/markdown-note/Java/ACS 240s超时问题分析.assets/三次握手.png)



### 四次挥手

![四次挥手](/Users/huangchenyao/Documents/markdown-note/Java/ACS 240s超时问题分析.assets/四次挥手.png)



## 长连接/短连接

### 长连接

每次通信完毕后，不会关闭连接，这样可以做到连接的复用。 **长连接的好处是省去了创建连接的耗时。**

![长连接example](/Users/huangchenyao/Documents/markdown-note/Java/ACS 240s超时问题分析.assets/长连接example.png)

### 短连接

每次通信时，创建 Socket；一次通信结束，调用 socket.close()。这就是一般意义上的短连接，短连接的好处是管理起来比较简单，存在的连接都是可用的连接，不需要额外的控制手段。

![短连接example](/Users/huangchenyao/Documents/markdown-note/Java/ACS 240s超时问题分析.assets/短连接example.png)

如果要做长连接，需要client端和service端http请求的请求头都带上`Connection: Keep-Alive`。

- 在HTTP 1.0中，需要自己配置请求头，加上`Connection: Keep-Alive`。

- 在HTTP 1.1中，所有的连接默认都是持续连接，默认都带上`Connection: Keep-Alive`，除非特殊声明不支持。



## NAT

网络地址转换（英语：Network Address Translation，缩写：NAT；又称网络掩蔽、IP掩蔽）在计算机网络中是一种在IP数据包通过路由器或防火墙时重写来源IP地址或目的IP地址的技术。这种技术被普遍使用在有多台主机但只通过一个公有IP地址访问互联网的私有网络中。

例子：内网：192.168.1.2, 公网：202.20.65.4，nat：202.20.65.5

发送：(Dst=202.20.65.4, Src=192.168.1.2)，nat后，(Dst=202.20.65.4，Src=202.20.65.5)

接收：(Dst= 202.20.65.5,Src=202.20.65.4)，nat后，(Dst=192.168.1.2, Src=202.20.65.4)



## 240s超时

### 表现

ACS容器平台的SDN有”240s超时重用“特性。即当应用在ACS容器平台上访问外部系统时，如果建立的TCP连接的连续空闲时间超过了240s，该连接会被ACS断开，表现在应用里面为大量的connection timeout。

240s特性，只影响从acs集群出去的流量，即acs上的应用访问外部系统；不影响外部进入acs集群的流量，即外部系统访问acs上的应用。

### 原理

参考paas



## Java Http Client

### 测试环境

- Java 14
- SpringBoot 2.2.6

### java.net原生包

基本使用方法如下：

```java
String link = "http://127.0.0.1:8080/test/value";
URL url = new URL(link);
HttpURLConnection httpURLConnection = (HttpURLConnection) url.openConnection();
httpURLConnection.setRequestMethod("GET");
InputStream is = connection.getInputStream();
// 读取数据
httpURLConnection.disconnect();
// httpURLConnection.getInputStream().close();
```

JDK自带的HttpURLConnection，默认启用`Connection: Keep-Alive`配置。

在完成获取请求返回结果后或调用`getInputStream().close()`，JDK会清理连接并作为以后使用的连接缓存。不过前提是header中的Connection值不能为close，否则将被视为该连接不需要持久化。

disconnect方法部分源码：

```java
public void disconnect() {
	// ...
  if (http != null) {
    if (inputStream != null) {
      HttpClient hc = http;

      // un-synchronized
      boolean ka = hc.isKeepingAlive();

      try {
        inputStream.close();
      } catch (IOException ioe) { }

      // if the connection is persistent it may have been closed
      // or returned to the keep-alive cache. If it's been returned
      // to the keep-alive cache then we would like to close it
      // but it may have been allocated

      if (ka) {
        hc.closeIdleConnection();
      }


    } else {
      // We are deliberatly being disconnected so HttpClient
      // should not try to resend the request no matter what stage
      // of the connection we are in.
      http.setDoNotRetry(true);

      http.closeServer();
    }

    //      poster = null;
    http = null;
    connected = false;
  }
  // ...
}
```

```java
public void closeIdleConnection() {
  HttpClient http = kac.get(url, null);
  if (http != null) {
    http.closeServer();
  }
}
```

```java
public void closeServer() {
  try {
    keepingAlive = false;
    serverSocket.close();
  } catch (Exception e) {}
}
```

可以看到调用`disconnect()`方法时，会去关闭socket，除非设定为长连接而且刚好这次http请求完了，空闲的tcp被分配给了其他线程使用。

所以需要做长连接，除了请求和返回头都有`Connection: Keep-Alive`外，关闭时需要使用`getInputStream().close()`代替`disconnect()`，才能做长连接。

### RestTemplate

基本使用方法

```java
SimpleClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
RestTemplate restTemplate = new RestTemplate(requestFactory);
restTemplate.setInterceptors(Collections.singletonList((request, body, execution) -> {
  HttpHeaders headers = request.getHeaders();
  //            headers.set("Connection", "keep-alive");
  return execution.execute(request, body);
}));
String url = "http://127.0.0.1:8080/test/value";
String response = restTemplate.getForObject(url, String.class);
```

客户端默认启用`Connection: Keep-Alive`配置。

服务端SpringBoot 2.2.6默认启用长连接，默认断开长连接时间为60s。

![resttemplate测试默认断开连接](/Users/huangchenyao/Documents/markdown-note/Java/ACS 240s超时问题分析.assets/resttemplate测试默认断开连接.png)

`SimpleClientHttpRequestFactory`获取到的`request`，实际上是封装了`HttpURLConnection`，

`RestTemplate`简易版源码如下：

```java
getxxx() {
  execute(GET);
}
postxxx() {
  execute(POST);
}

execute(xxx) {
  doExecute();
}

doExecute() {
  ClientHttpResponse response;
  try {
  } catch(Exception e) {
  } finally {
    response.close();
  }
}

// 多态，用SimpleClientHttpRequestFactory获取到的response就是SimpleClientHttpResponse
final class SimpleClientHttpResponse extends AbstractClientHttpResponse {
  close() {
    responseStream.close();
  }
}
```

可以看到`SimpleClientHttpRequestFactory`实际上关闭的时候调用的是`httpURLConnection.getInputStream().close()`方法，这个方法是不会去关闭长连接的。

### Apache HttpClient

```java
private PoolingHttpClientConnectionManager poolConnManager;
private CloseableHttpClient httpClient;

// 连接池配置
private void setPoolConnManager() {
  try {
    SSLContextBuilder builder = new SSLContextBuilder();
    builder.loadTrustMaterial(null, new TrustSelfSignedStrategy());
    SSLConnectionSocketFactory sslConnectionSocketFactory = new SSLConnectionSocketFactory(builder.build());
    // 配置同时支持 HTTP 和 HTTPS
    Registry<ConnectionSocketFactory> socketFactoryRegistry = RegistryBuilder.<ConnectionSocketFactory>create()
      .register("http", PlainConnectionSocketFactory.getSocketFactory())
      .register("https", sslConnectionSocketFactory)
      .build();
    // 初始化连接管理器
    poolConnManager = new PoolingHttpClientConnectionManager(socketFactoryRegistry);
    poolConnManager.setMaxTotal(640);// 同时最多连接数
    // 设置最大路由
    poolConnManager.setDefaultMaxPerRoute(320);
  } catch (NoSuchAlgorithmException | KeyStoreException | KeyManagementException e) {
    e.printStackTrace();
  }
}

// 连接配置
private void setHttpClient() {
  RequestConfig config = RequestConfig.custom()
    .setConnectTimeout(5000)
    .setConnectionRequestTimeout(5000)
    .setSocketTimeout(5000)
    .build();
  // keep-alive策略
  ConnectionKeepAliveStrategy myStrategy = (response, context) -> 5 * 1000;

  httpClient = HttpClients.custom()
    // 设置连接池管理
    .setConnectionManager(poolConnManager)
    .setDefaultRequestConfig(config)
    .setKeepAliveStrategy(myStrategy)
    // 设置重试次数
    .setRetryHandler(new DefaultHttpRequestRetryHandler(2, false)).build();
}
```

`  ConnectionKeepAliveStrategy myStrategy = (response, context) -> 5 * 1000;`

用于设置keep-alive策略，在超时时间内的，会复用TCP连接，超出超时时间的，会先断开连接，再新建连接进行请求。

![apache httpclient example](/Users/huangchenyao/Documents/markdown-note/Java/ACS 240s超时问题分析.assets/apache httpclient example.png)

与`RestTemplate`结合使用，除了上面的配置连接池和客户端，还需要把`RestTemplate`的客户端设置为上面生成的客户端。

```java
HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory(httpClient);
RestTemplate restTemplate = new RestTemplate(factory);
```



## Zuul

Zuul是微服务网关，是微服务架构中对外提供服务的统一入口，将HttpServletRequest的相关数据（HTTP方法、参数、请求头等），转换为HttpClient的请求实例（HttpRequest），再使用CloseableHttpClient进行转发。

Zuul默认使用**Apache HttpClient**作为代理。

Zuul的连接池默认是永久保持，需要修改配置`zuul.host.time-to-live`。



## 结论

- 原生的使用起来比较麻烦，如果要避免240s的问题，需要设置请求头`Connection: Close`，或者手动关闭连接。
- RestTemplate，如果要避免240s的问题，`SimpleClientHttpRequestFactory`实际上就是使用原生的，所以需要设置请求头`Connection: Close`。或者改成使用Apache HttpClient，配置就跟Apache HttpClient走。
- Apache HttpClient，如果要避免240s的问题，需要自行设置keep-alive策略。
- Zuul，需要修改配置`zuul.host.time-to-live`。

