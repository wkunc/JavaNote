# Struts2 流程

我们从 Struts2 提供的 Filter 看起
```java
/*
 * 这是它里面的实例域, 分别是 request 请求前的 预处理操作们, filters 的执行操作们, 和一个排除正则列表
 */
protected PrepareOperations prepare;
protected ExecuteOperations execute;
protected List<Pattern> excludePatterns = null;
```
首先是它的 init() 方法,这个方法是 servlet 规范中的方法, 会在 servlet 容器启动后调用

```java
public void init(FilterConfig filterConfig) throws ServletException {
    // createInitOperation(); 方法只是 InitOperations 构造器的包装
    InitOperations init = createInitOperations();
    Dispatcher dispatcher = null;
    try {
        // FilterHostConfig 类是 FilterConfig 的包装类, 简化了访问 FilterConfig 中访问初始化参数的过程
        FilterHostConfig config = new FilterHostConfig(filterConfig);
        init.initLogging(config);
        // Dispatcher 是转发请求到对应位置的关键类
        // 下面深入 initDispatcher() 方法
        dispatcher = init.initDispatcher(config);
        init.initStaticContentLoader(config, dispatcher);

        prepare = createPrepareOperations(dispatcher);
        execute = createExecuteOperations(dispatcher);
        this.excludedPatterns = init.buildExcludedPatternsList(dispatcher);

        postInit(dispatcher, filterConfig);
    } finally {
        if (dispatcher != null) {
            dispatcher.cleanUpAfterInit();
        }
        init.cleanup();
    }
}
```

## dispatcher = init.initDispatcher(config);
这是完整的initDispatcher()方法
```java
public Dispatcher initDispatcher( HostConfig filterConfig ) {
    
    /*
     *createDispatcher(filterConfig); 只是 Dispatcher 构造器的简单封装
     *它把filterConfig 中的 初始化参数取出组装成 HashMap, 
     *并且获得 ServletContexet 然后调用 Dispathcher 的构造器
     */
    Dispatcher dispatcher = createDispatcher(filterConfig);
    /*
     * 调用 Dispatcher 上的init(); 真正的初始化 Dispatcher 对象
     * 让我们继续追寻 Dispatcher 的初始化过程
     */
    dispatcher.init();
    return dispatcher;
}
```

## dispatcher.init();
为了方便理解我摘抄了完整的 init 方法和Disptcher 中的 **部分** 变量
```java
// Dispatcher 在多线程中的魔力
private static ThreadLocal<Dispatcher> instance = new ThreadLocal<>();
// 一个 Dispathcer 监听者列表
private static List<DispatcherListener> dispatcherListeners = new CopyOnWriteArrayList<>();
// 一个 Dispatcher 的错误处理器
private DispatcherErrorHandler errorHandler;
// 一个 配置管理器
protected ConfigurationManager configurationManager;
// 构造器中初始化完成的 Filter 的初始化参数 和 ServletContext
protected Map<String, String> initParams;
protected ServletContext servletContext;
//这个方法里依次调用了 Dispatcher 中的 7 个私有方法为 configurationManager 添加了不同的 ContainerProvider 对象
// 最后两个 私有方法
public void init() {
    // 第一次 configurationManager 一定是 null ,调用 crate 方法获得 configurationManager 实例 
    if (configurationManager == null) {
        // 这个方法调用等同于 = new ConfigurationManager("defulat");
        configurationManager = createConfigurationManager(Container.DEFAULT_NAME);
    }
    try {
        init_FileManager();
        init_DefaultProperties(); // [1]
        init_TraditionalXmlConfigurations(); // [2]
        init_LegacyStrutsProperties(); // [3]
        init_CustomConfigurationProviders(); // [5]
        init_FilterInitParameters() ; // [6]
        init_AliasStandardObjects() ; // [7]

        Container container = init_PreloadConfiguration();
        container.inject(this);
        init_CheckWebLogicWorkaround(container);

        if (!dispatcherListeners.isEmpty()) {
            for (DispatcherListener l : dispatcherListeners) {
                l.dispatcherInitialized(this);
            }
        }
        errorHandler.init(servletContext);

    } catch (Exception ex) {
        LOG.error("Dispatcher initialization failed", ex);
        throw new StrutsException(ex);
    }
}
```

##XmlconfigurationProvider 调用 DomHelper 负责加载 配置文件
