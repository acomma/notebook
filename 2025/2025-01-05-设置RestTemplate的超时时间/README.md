## 1、样例代码

### 1.1、客户端

```java
@RestController
public class GreetingController {
    private final RestTemplate restTemplate;

    public GreetingController(RestTemplateBuilder restTemplateBuilder) {
        restTemplate = restTemplateBuilder
                .connectTimeout(Duration.ofSeconds(5)) // 建立连接的超时时间
                .readTimeout(Duration.ofSeconds(5))    // 读写数据的超时时间
                .build();
    }

    @GetMapping("/greet")
    public String greet() {
        ResponseEntity<String> response = restTemplate.getForEntity("http://localhost:8081/greet", String.class);
        return response.getBody();
    }
}
```

### 1.2、服务端

```java
@RestController
public class GreetingController {
    @GetMapping("/greet")
    public String greet() throws InterruptedException {
        TimeUnit.SECONDS.sleep(10);
        return "Hello World!";
    }
}
```

## 2、建立连接超时

`connectTimeout` 设置的是客户端和服务端建立连接的超时时间，超过这个时间后客户端不再尝试与服务端建立连接。为了模拟连接超时的场景需要做两件事情：

1. `connectTimeout` 的值设置得比 `readTimeout` 低，比如设置为 3 秒；
2. 让 `RestTemplate` 连接一个不存在的服务器，比如 `http://10.10.10.10:8081/greet`，`10.10.10.10` 在我的测试环境是不存在的。

修改后的客户端代码如下所示

```java
@RestController
public class GreetingController {
    private final RestTemplate restTemplate;

    public GreetingController(RestTemplateBuilder restTemplateBuilder) {
        restTemplate = restTemplateBuilder
                .connectTimeout(Duration.ofSeconds(3))
                .readTimeout(Duration.ofSeconds(5))
                .build();
    }

    @GetMapping("/greet")
    public String greet() {
        ResponseEntity<String> response = restTemplate.getForEntity("http://10.10.10.10:8081/greet", String.class);
        return response.getBody();
    }
}
```

建立连接超时时客户端会抛出异常 `java.net.ConnectException: HTTP connect timed out`，异常堆栈如下所示

```text
java.net.ConnectException: HTTP connect timed out
	at java.net.http/jdk.internal.net.http.MultiExchange.toTimeoutException(MultiExchange.java:581) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.MultiExchange.getExceptionalCF(MultiExchange.java:527) ~[java.net.http:na]
	at java.net.http/jdk.internal.net.http.MultiExchange.lambda$responseAsyncImpl$7(MultiExchange.java:447) ~[java.net.http:na]
	at java.base/java.util.concurrent.CompletableFuture.uniHandle(CompletableFuture.java:934) ~[na:na]
	at java.base/java.util.concurrent.CompletableFuture$UniHandle.tryFire(CompletableFuture.java:911) ~[na:na]
	at java.base/java.util.concurrent.CompletableFuture.postComplete(CompletableFuture.java:510) ~[na:na]
	at java.base/java.util.concurrent.CompletableFuture.completeExceptionally(CompletableFuture.java:2162) ~[na:na]
	at java.net.http/jdk.internal.net.http.Http1Exchange.lambda$cancelImpl$9(Http1Exchange.java:483) ~[java.net.http:na]
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136) ~[na:na]
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635) ~[na:na]
	at java.base/java.lang.Thread.run(Thread.java:833) ~[na:na]
```

测试过程中发现如果 `connectTimeout` 的值和 `readTimeout` 的值设置得一样或者 `connectTimeout` 的值比 `readTimeout` 的值高，则生效的是 `readTimeout` 的值。此时将会抛出异常 `java.util.concurrent.CancellationException: null`，异常堆栈如下所示

```text
java.util.concurrent.CancellationException: null
	at java.base/java.util.concurrent.CompletableFuture.cancel(CompletableFuture.java:2478) ~[na:na]
	at java.net.http/jdk.internal.net.http.common.MinimalFuture.cancel(MinimalFuture.java:109) ~[java.net.http:na]
	at org.springframework.http.client.JdkClientHttpRequest$TimeoutHandler.lambda$new$0(JdkClientHttpRequest.java:234) ~[spring-web-6.2.1.jar:6.2.1]
	at java.base/java.util.concurrent.CompletableFuture$UniRun.tryFire$$$capture(CompletableFuture.java:787) ~[na:na]
	at java.base/java.util.concurrent.CompletableFuture$UniRun.tryFire(CompletableFuture.java) ~[na:na]
	at java.base/java.util.concurrent.CompletableFuture.postComplete(CompletableFuture.java:510) ~[na:na]
	at java.base/java.util.concurrent.CompletableFuture.complete(CompletableFuture.java:2147) ~[na:na]
	at java.base/java.util.concurrent.CompletableFuture$DelayedCompleter.run(CompletableFuture.java:2885) ~[na:na]
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:539) ~[na:na]
	at java.base/java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:264) ~[na:na]
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java) ~[na:na]
	at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304) ~[na:na]
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136) ~[na:na]
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635) ~[na:na]
	at java.base/java.lang.Thread.run(Thread.java:833) ~[na:na]
```

## 3、读写数据超时

`readTimeout` 设置的是客户端和服务端建立连接后等待服务端返回数据的时间，超过这个时间后客户端不再等待服务端返回数据并断开与服务端的连接。比如 `readTimeout` 设置为 5 秒中，服务端处理请求需要花费 10 秒中，5 秒钟后客户端就会抛出异常 `java.net.http.HttpTimeoutException: Request timed out`，异常堆栈如下所示

```text
java.net.http.HttpTimeoutException: Request timed out
	at org.springframework.http.client.JdkClientHttpRequest.executeInternal(JdkClientHttpRequest.java:124) ~[spring-web-6.2.1.jar:6.2.1]
	at org.springframework.http.client.AbstractStreamingClientHttpRequest.executeInternal(AbstractStreamingClientHttpRequest.java:71) ~[spring-web-6.2.1.jar:6.2.1]
	at org.springframework.http.client.AbstractClientHttpRequest.execute(AbstractClientHttpRequest.java:81) ~[spring-web-6.2.1.jar:6.2.1]
	at org.springframework.web.client.RestTemplate.doExecute(RestTemplate.java:900) ~[spring-web-6.2.1.jar:6.2.1]
	at org.springframework.web.client.RestTemplate.execute(RestTemplate.java:801) ~[spring-web-6.2.1.jar:6.2.1]
	at org.springframework.web.client.RestTemplate.getForEntity(RestTemplate.java:442) ~[spring-web-6.2.1.jar:6.2.1]
	at com.example.client.GreetingController.greet(GreetingController.java:24) ~[classes/:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77) ~[na:na]
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:na]
	at java.base/java.lang.reflect.Method.invoke(Method.java:568) ~[na:na]
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:257) ~[spring-web-6.2.1.jar:6.2.1]
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:190) ~[spring-web-6.2.1.jar:6.2.1]
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:118) ~[spring-webmvc-6.2.1.jar:6.2.1]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:986) ~[spring-webmvc-6.2.1.jar:6.2.1]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:891) ~[spring-webmvc-6.2.1.jar:6.2.1]
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87) ~[spring-webmvc-6.2.1.jar:6.2.1]
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1088) ~[spring-webmvc-6.2.1.jar:6.2.1]
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:978) ~[spring-webmvc-6.2.1.jar:6.2.1]
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1014) ~[spring-webmvc-6.2.1.jar:6.2.1]
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:903) ~[spring-webmvc-6.2.1.jar:6.2.1]
	at jakarta.servlet.http.HttpServlet.service(HttpServlet.java:564) ~[tomcat-embed-core-10.1.34.jar:6.0]
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:885) ~[spring-webmvc-6.2.1.jar:6.2.1]
	at jakarta.servlet.http.HttpServlet.service(HttpServlet.java:658) ~[tomcat-embed-core-10.1.34.jar:6.0]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:195) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:51) ~[tomcat-embed-websocket-10.1.34.jar:10.1.34]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:164) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100) ~[spring-web-6.2.1.jar:6.2.1]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.2.1.jar:6.2.1]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:164) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93) ~[spring-web-6.2.1.jar:6.2.1]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.2.1.jar:6.2.1]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:164) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201) ~[spring-web-6.2.1.jar:6.2.1]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.2.1.jar:6.2.1]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:164) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:167) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:90) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:483) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:115) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:93) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:344) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:397) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:63) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:905) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1741) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:52) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1190) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:659) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:63) ~[tomcat-embed-core-10.1.34.jar:10.1.34]
	at java.base/java.lang.Thread.run(Thread.java:833) ~[na:na]
```
