# Reactive Cores

和基于 servlet 的mvc一样,
spring-web 模块也提供了 reactive web Application 的基础.

对于一次请求处理的过程, 我们有两个级别的支持.

1. HttpHandler, 具有非阻塞I/O和Reactive Stream的 HTTP 请求处理抽象.
   它是一个低级别的抽象, 基本和基于 servlet 的抽象没什么区别.
   以及用于 Reactor Netty(TomcatHttpHandlerAdapter), Undertow(UndertowHttpHandlerAdapter),
   Tomcat(TomcatHttpHandlerAdapter), Jetty(JettyHttpHandlerAdapter),
   Servlet(TomcatHttpHandlerAdapter) 的 adapter.

2. WebHandler, 用于请求处理, 在此基础上构建具体的编程模型.

## Handler

HttpHandler 是一个简单的 contract, 只有一个方法来处理 request, response.
它是故意设计成最小的抽象, 为了成为不同HttpServerAPI的最小抽象.

```java
public interface HttpHandler {
    Mono<Void> handler(ServerHttpRequest request, ServerHttpResponse response);
}
```

不管是什么服务器, 都能从 HttpHandler 加上合适的adapter启动.
> Reactor Netty
> ```java
> HttpHandler handler = ...
>
> ReactorHttpHandlerAdapter adapter = new ReactorHttpHandlerAdapter(handler);
> HttpServer.create(host,port).newHandler(adapter).block();
> ```
>
> Tomcat
> ```java
> HttpHandler handler = ...
>
> Servlet servlet = new TomcatHttpHandlerAdapter(handler);
> Tomcat server = new Tomcat();
> File base = new File(System.getProperty("java.io.tmpdir"));
> Context rootContext = server.addContext("", base.getAbsolutePath());
> Tomcat.addServlet(rootContext, "main", servlet);
> rootContext.addServletMappingDecoded("/", "main");
> server.setHost(host);
> server.setPort(port);
> server.start();
> ```
>

## WebHandler API

org.springframework.web.server 包基于上面的 HttpHandler 构建了通用的Web API.
通过多个 WebExceptionHandler, 多个 WebFilter, 和单个 WebHandler 组成组件链.
处理请求.

WebHttpHandlerBuilder 可以将上面提到的组件合理的变成我们要的组件链.
并且还会通过 HttpWebhandlerAdapter 将组件链变成一个底层的HttpHandler抽象.
供各种服务器使用.

WebHttpHandlerBuilder 有两种使用方式,

1. 显式的调用方法, 手动的进行组件注册.
2. 和ApplicationContext一起使用, 它会自动的寻找自己需要的组件并构建.

虽然HttpHandler提供了一个简单通用的抽象来使用不同的HTTP服务器.
但WebHandler API的目的是提供Web程序中常用的一组功能, 比如:

1. Session
2. Requset Attribute
3. Locale 和 Principal
4. multipart data(文件上传)
5. 等等功能.

### Special Bean Type

| Bean name                  | Bean Type                  | Count | Description
|----------------------------|----------------------------|-------|-------------|
| any                        | WebExceptionHandler        | 0..N  |
| any                        | WebFilter                  | 0..N  |
| webHandler                 | WebHandler                 | 1     |
| webSessionManager          | WebSeesionManager          | 1..0  |
| serverCodecConfigurer      | ServerCodecConfigurer      | 0..1  |
| localeContextResolver      | LocaleContextResolver      | 0..1  |
| forwardedHeaderTransFormer | ForwardedHeaderTransFormer | 0..1  |

总结一下, 就是只要是 WebExceptionHandler, WebFilter 类型的Bean.
都会自动注册, 并且个数不限.
而 WebHandler 类型的Bean只有一个会被注册, 而且名字必须叫webHandler.

ServerCodecCOnfigurer 用于访问HttpMessageReader实例用以解析表单数据.
然后通过ServerWebExchange 上的方法公开这些数据.

# DispatcherHandler

讲完了WebHandler API的组成, 接下来探究Spring提供的默认配置.
首先它提供了多个Webhandler接口的实现类.
而我们在配置文件中可以找出一个 DispatcherHandler 配置成 webHandler.

这个DispatherHandler的作用和MVC框架中的DispatcherServlet一样.
用来转发请求, 它也是一个前端控制器的结构.

DispatherHandler 会从ApplicationContext中查找它需要的组件,
然后它也会被上面提到的 WebHttpHandlerBuilder 找到, 添加到组成链中去.

DispatherHandler 需要以下类型的组件:

1. HandlerMapping
2. HandlerAdapter
3. HandlerResultHandler

这几个组件名看起来都很眼熟, 因为在MVC框架中.
DispatcherServlet 也会查找 HandlerMapping, HandlerAdapter.
不过MVC查找的HandlerMapping, HandlerAdapter和Web-Flux查找的并不是同一个目标.

这是不同包中的同名接口, 不过这也能帮我们理解整个过程了.
名字相同的接口目的是相同的, 只不过针对不同的技术栈里面方法参数返回有所不同罢了.

那么这里的 HandlerMapping 就对应查找Handler的功能.
而 HandlerAdapter 就是调用不同类型handler的适配器.

DispatcherHandler 真的很简洁, 可能用响应式编程原因.
内部逻辑和 MVC 版的前端控制器区别不大, 推荐自己查看源码.
