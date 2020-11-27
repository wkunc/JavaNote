# HttpServletBean
```java
public final void init() throws ServletException {
    PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig()), this.requiredProperties);
    if (!pvs.isEmpty()) {
        try {
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
            bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
            initBeanWrapper(bw);
            bw.setPropertyValues(pvs, true);
        }
        catch (BeansException ex) {
            if (logger.isErrorEnable()){
                logger.error("Failed to set bean properties on servlet '" + getServletName() + ",", ex);
            }
            throw ex;
        }
    }
    // 重点方法, 在FrameworkServlet中进行了Context初始化
    initServletBean();
}
```

# FrameworkServlet
```java
@Override
protected final void initServletBean() throws ServletException {
    getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
    if (logger.isInfoEnabled()) {
        logger.info("Initializing Servlet '" + getServletName() + "'");
    }
    long startTime = System.currentTimeMillis();

    // 重点的两个方法, 在initWebApplicationContext() 中初始化Context
    try {
        this.webApplicationContext = initWebApplicationContext();
        initFrameworkServlet();
    }
    catch (ServletException | RuntimeException ex) {
        logger.error("Context initialization failed", ex);
        throw ex;
    }

    if (logger.isDebugEnabled()) {
        String value = this.enableLoggingRequestDetails ?
                "shown which may lead to unsafe logging of potentially sensitive data" :
                "masked to prevent unsafe logging of potentially sensitive data";
        logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
                "': request parameters and headers will be " + value);
    }

    if (logger.isInfoEnabled()) {
        logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
    }
}
```

```java
protected WebApplicationContext initWebApplicationContext() {
    // 获取根Context, 就是 ContextLisenter注册的Context
    WebApplicationContext rootContext =
            WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    WebApplicationContext wac = null;

    // 如果this.webApplicationContext 不为空, 说明它是用SPI机制注册的
    if (this.webApplicationContext != null) {
        // A context instance was injected at construction time -> use it
        wac = this.webApplicationContext;
        if (wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
            if (!cwac.isActive()) {
                // The context has not yet been refreshed -> provide services such as
                // setting the parent context, setting the application context id, etc
                if (cwac.getParent() == null) {
                    // The context instance was injected without an explicit parent -> set
                    // the root application context (if any; may be null) as the parent
                    cwac.setParent(rootContext);
                }
                configureAndRefreshWebApplicationContext(cwac);
            }
        }
    }
    // 说明DispatcherServlet是通过web.xml注册的
    if (wac == null) {
        // No context instance was injected at construction time -> see if one
        // has been registered in the servlet context. If one exists, it is assumed
        // that the parent context (if any) has already been set and that the
        // user has performed any initialization such as setting the context id
        wac = findWebApplicationContext();
    }
    if (wac == null) {
        // No context instance is defined for this servlet -> create a local one
        wac = createWebApplicationContext(rootContext);
    }

    // 重点部分
    // onRefresh() 方法在FrameworkServlet是一个空方法
    // 由子类实现,而 DispatcherServlet 中会初始化策略
    if (!this.refreshEventReceived) {
        // Either the context is not a ConfigurableApplicationContext with refresh
        // support or the context injected at construction time had already been
        // refreshed -> trigger initial onRefresh manually here.
        synchronized (this.onRefreshMonitor) {
            onRefresh(wac);
        }
    }

    if (this.publishContext) {
        // Publish the context as a servlet context attribute.
        String attrName = getServletContextAttributeName();
        getServletContext().setAttribute(attrName, wac);
    }

    return wac;
}
```

# DispatcherServlet
```java
@Override
protected void onRefresh(ApplicationContext context) {
    initStartegies(context);
}
// 除了 MultipartResovler 是没有默认配置的,其他的都是有默认配置的
// 这些初始化方法的逻辑基本都是, 从传入的 context 中获取想要类型,名字的Bean,
// 如果Context中没有,就根据默认配置创建指定的Bean.
// 默认配置由一个叫defaultStartegies的Properties的对象维护
// 实际上就是一个类加载时加载到内存的属性文件, 位于 spring-webmvc 包内
protected void initStartegies(ApplicationContext context) {
    // 初始化文件解析器
    initMultipartResolver(context);
    // 本地解析器
    initLocaleResolver(context);
    // 主题解析器
    initThemeResolver(context);
    // HandlerMapping
    intiHandlerMappings(context);
    // HadnlerAdapter
    initHandlerAdapters(context);
    // 异常处理器
    initHandlerExceptionResolvers(context);
    // 翻译器, 当使用 @Controller 处理请求,
    // 但是 handler 方法没有明确指明 viewName. 
    // 通过这个翻译器会将请求的文件扩展名去除作为viewName
    intiRequestToViewNameTranslator(context);
    // 视图解析器
    initViewResolvers(context);
    // flash属性管理器
    initFlashMapManager(context);
}
```
