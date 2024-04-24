# Servlet 接口

Servlet 接口是 Java Servlet API 的核心抽象. 所有 Servlet 类必须字节或间接的实现该接口,
或者更通常做法是通过继承一个实现了该接口的类从而复用许多共性功能.
目前有 GenericServlet 和 HttpServlet 这个两个类实现了 Servlet 接口.
大多数情况下, 开发者只需要继承 HttpServlet 去实现自己的 Servlet 即可.

## 请求处理方法

Servlet 接口定义了定义了用于客户端请求处理的 service() 方法.
当有请求到达时, 该方法由 Servlet 容器路由到一个 Servlet 实例.

Web 应用程序的并发请求处理通常需要 Web 开发人员设计合适多线程执行的 Servlet, 从而保证
service() 方法能在一个特定时间点处理多线程并发执行.
(Servlet 默认是线程不安全的, 需要开发人员处理多线程问题)

通常 Web 容器对于并发请求将使用同一个 servlet 处理, 并且在不同线程中并发执行 service 方法

## 实例数量

未托管在分布式环境中的 servlet ,servlet 容器对于每一个 Servlet 声明必须只能产生一个实例

# Servlet 的生命周期

所有Servlet必须直接或间接的实现 GenericServlet 或 HttpServlet 抽象类

## 加载和实例化

Servlet容器负责加载和实例化 Servlet. 加载和实例化可以发生在容器启动时, 或者延迟初始化直到
容器决定由请求需要处理时.

## 初始化

容器通过调用 Servlet 实例的 init 方法完成初始化, init方法定义在 Servlet 接口中, 并提供一个
唯一的 ServletConfig 接口实现的对象作为参数, 该对象每个 Servlet 实例一个.

```java
public void init(ServletCOnfig config) throws ServletException;
```

## 实例数量

未托管在分布式环境中的 servlet ,servlet 容器对于每一个 Servlet 声明必须只能产生一个实例

ServletConfig 接口定义了访问 Web 应用配置信息的方法,
也提供了一个方法去方法 ServletContexxt 对象 ServletContext 对象描述了 Servlet 的运行环境

```java
public interface ServletConfig {
    public String getServletName();
    public ServletCOntext getServletContext();
    public String getInitParameter(String name);
    public Enumeration<String> getInitParameterNames();
}
```

# 请求处理

## 异步处理

有时候, Filter 以及 Servlet 在生成响应之前等待一些资源或完成请求处理.
比如, Servlet 在生成响应可能等待一个可用的 JDBC 连接, 或者一个远程 web 服务的响应,
或者一个 JMS 消息, 或者一个应用程序事件. 在 Servlet 中等待是一个低效的操作,
因为这是一个阻塞操作, 从而白白占用一个线程或其他一些受限资源.
许多线程为等待一个缓慢的资源比如数据库经常发生阻塞, 可能引起线程饥饿,且降低 web 容器的服务质量

Servlet 3.0 引入了异步处理请求的能力, 线程可以返回到容器, 从而执行更多的任务.
当开始异步处理请求时, 另一个线程或回调可以产生响应,或者调用完成(complete),或者请求分配(dispatch)

一个典型
