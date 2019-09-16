# 原理
基于Filter实现的安全框架, 第一步当然是在web.xml中配置Filter, 
Spring推荐配置一个叫springSecurityFilterChain(名字其实无所谓,这是默认名字)的
DelegatingFilterProxy(它实现了Filter接口)对象实例,
它是一个Filter的静态代理对象, 它内部会代理一个Filter, 
然后在IOC容器中我们要配置一个和其同名的Filter对象.这个对象会被注入到我们配置的Filter中.

Spring Security提供了一个FilterChainProxy类,
它还是一个代理对象, 它会代理它内部的所有Filter对象, 
这样多个Filter就被组合在一起,对外表现为一个.

总的流程就是request首先通过 DelegatingFilterProxy 这个过滤器,
内部将流程委托给 FilterChainProxy 对象, 
在FilterChainProxy对象内部将逻辑委托给一组Filter.

DelegatingFilterProxy 这个类在spring.web.jar中提供,并不是Spring Securtiy提供的.
它只是合理利用了Spring框架提供的功能,
所以这个DelegatingFilterProxy的目的就是:
将Filter的初始化委托给SpringIOC容器, 
让Filter的生命周期交给Spring而不是Servlet容器如tomcat.

而FilterChainProxy类SpringSecurity自己提供, 
通常实现web应用的安全性我们会用到很多Filter对象(单一职责原则,每个Filter负责一个方面).
这样的话需要注册多次, 使用一个FilterChainProxy对象代理一整个Filter链.

FilterChainProxy 类对象实例由 WebSecurity (WebSecurity是一个builder)对象构建.
FilterChainProxy 类中持有多个 SecurityFilterChain(一个Filter链的接口表示) 对象.
而Spring Security 提供了 HttpSecurity (也是一个builder) 来创建一个 SecurityFilterChain 对象.

# AbstractSecurityWebApplicationInitializer
官方推荐使用这个来注册
