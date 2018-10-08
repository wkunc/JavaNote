# 应用生命周期事件
应用实践设施给 Web 开发人员更好地控制 ServletContext, HttpSession 和 ServletRequest 的生命周期,
可以更好地代码分解, 并在管理 Web 应用使用的资源上提高了效率

# 事件监听器
ServletContext 事件
* ServletContextListener
* ServletContextAttributeListener

Http 会话事件
* HttpSessionListener
* HttpSessionIdListener
* HttpSessionActivationListener
* HttpSessionBindingListener

Servlet 请求事件
* ServletRequestListener
* ServletRequestAttributeListener
* AsyncListener

# 注意
每一个监听器类都必须有一个无参构造器
