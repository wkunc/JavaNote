# Servlet 的生命周期
## 加载和实例化

## 初始化
容器通过调用 Servlet 实例的 init 方法完成初始化, init方法定义在 Servlet 接口中, 并提供一个
唯一的 ServletConfig 接口实现的对象作为参数, 该对象每个 Servlet 实例一个.
```java
public void init(ServletCOnfig config) throws ServletException;
```
ServletConfig 接口定义了访问 Web 应用配置信息的方法, 也提供了一个方法去方法 ServletContexxt 对象
ServletContext 对象描述了 Servlet 的运行环境
```java
public interface ServletConfig {
    public String getServletName();
    public ServletCOntext getServletContext();
    public String getInitParameter(String name);
    public Enumeration<String> getInitParameterNames();
}
```
