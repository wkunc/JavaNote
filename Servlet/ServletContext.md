# ServletContext

## ServletContext 接口介绍

ServletContext (Servlet 上下文) 接口定义了 Servlet 运行在 Web 应用的视图
容器提供商负责提供ServletContext接口的实现

Servlet 可以使用 ServletContext 对象记录事件, 获取 URL 引用的资源, 存取当前上下文的其他Servlet可以访问的属性

ServletContext 是 Web 服务器中已知路径的根

## ServletContext 接口作用范围

每一个部署到容器的 Web Application 都有一个 ServletContext 接口实例与之关联

未完待续...

## 初始化参数

Servlet 接口的一下方法允许,servlet 获得 Webapp 在部署时的初始化参数

```java
public String getInitParameter(String name);
public Enumeratoin<String> getInitParameterNames();
```

## 配置方法

### 编程式添加和配置Srvlet

编程式添加 Servlet, FIlter, Listener 到上下文对框架开发者是很有用的
框架可以使用这个方法声明一个控制器 Servlet

这些方法将会返回一个 ServletRegistration(serlvet 注册器) 或 ServletRegistration.Dynamic对象
允许我们进一步配置如: init-params, url-mapping 等 Servlet 的其他参数

```java
public ServletRegistration.Dynamic addServlet(String servletName, String className);
public ServletRegistration.Dynamic addServlet(String servletName, Servlet);
public ServletRegistration.Dynamic addServlet(String servletName, Class<? extends Sservlet> servletClass);
pubilc <T extends Servlet> T createServlet(Class<T> clazz) throws ServletException;
public ServletRegistraion getServletRegistration(String servlet Name);
public Map<String, ? extends ServletRegistration> getServletRegistrations();
```

