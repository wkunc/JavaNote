# 流程分析

在DispatcherServlet工作的过程中, 会调用本次请求对应的 handlerAdapter.handle()方法.
RequestMappingHandlerAdapter 继承自 AbstractHandlerMethodAdapter.
所以其本意就是想要支持 HandlerMethod 类型的 handler, 重写了 support() 方法
(这个抽象类非常简单, 就不具体分析了)

```java
public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) {
    return handleInternal(request, response, (HandlerMethod)handler);
}
```

# RequestMappingHandlerAdapter

如果说 RequestMappingHandlerMapping 负责定义了一个基于方法的mvc模式,
它将每个request路由到对应的method上. 负责 @RequestMapping, @CorssOrigin 注解的解析.
实现了非常舒服的路由方法, 只需要简单的注解就可以将一个方法变成一个handler.

那么这个 RequestMappingHandlerAdapter 就负责让我们的方法编写起来更加方便.
超级多, 超级神奇的功能都是它实现的.

方法的执行, 方法参数的自动解析, 方法返回值的解决.
都是有这个 Adapter 实现的.

当然根据单一职责原则, 它不可能自己实现了如此众多的功能.
交给具体的模块类实现不同的功能然后组合在一起.
正因为如此, 它的成员变量多的恐怖.

```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {

    // 静态常量
	public static final MethodFilter INIT_BINDER_METHODS = method ->
			AnnotatedElementUtils.hasAnnotation(method, InitBinder.class);

	public static final MethodFilter MODEL_ATTRIBUTE_METHODS = method ->
			(!AnnotatedElementUtils.hasAnnotation(method, RequestMapping.class) &&
					AnnotatedElementUtils.hasAnnotation(method, ModelAttribute.class));


    // 成员变量, 26个
	private List<HandlerMethodArgumentResolver> customArgumentResolvers;
	private HandlerMethodArgumentResolverComposite argumentResolvers;
	private HandlerMethodArgumentResolverComposite initBinderArgumentResolvers;

	private List<HandlerMethodReturnValueHandler> customReturnValueHandlers;
	private HandlerMethodReturnValueHandlerComposite returnValueHandlers;

	private List<ModelAndViewResolver> modelAndViewResolvers;

	private ContentNegotiationManager contentNegotiationManager = new ContentNegotiationManager();

	private List<HttpMessageConverter<?>> messageConverters;

	private List<Object> requestResponseBodyAdvice = new ArrayList<>();

	private WebBindingInitializer webBindingInitializer;

	private AsyncTaskExecutor taskExecutor = new SimpleAsyncTaskExecutor("MvcAsync");

	private Long asyncRequestTimeout;

	private CallableProcessingInterceptor[] callableInterceptors = new CallableProcessingInterceptor[0];

	private DeferredResultProcessingInterceptor[] deferredResultInterceptors = new DeferredResultProcessingInterceptor[0];

	private ReactiveAdapterRegistry reactiveAdapterRegistry = ReactiveAdapterRegistry.getSharedInstance();

	private boolean ignoreDefaultModelOnRedirect = false;

	private int cacheSecondsForSessionAttributeHandlers = 0;

	private boolean synchronizeOnSession = false;

	private SessionAttributeStore sessionAttributeStore = new DefaultSessionAttributeStore();

	private ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();

	private ConfigurableBeanFactory beanFactory;


    //
	private final Map<Class<?>, SessionAttributesHandler> sessionAttributesHandlerCache = new ConcurrentHashMap<>(64);

	private final Map<Class<?>, Set<Method>> initBinderCache = new ConcurrentHashMap<>(64);

	private final Map<ControllerAdviceBean, Set<Method>> initBinderAdviceCache = new LinkedHashMap<>();

	private final Map<Class<?>, Set<Method>> modelAttributeCache = new ConcurrentHashMap<>(64);

	private final Map<ControllerAdviceBean, Set<Method>> modelAttributeAdviceCache = new LinkedHashMap<>();


    // 构造器
	public RequestMappingHandlerAdapter() {
		StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter();
		stringHttpMessageConverter.setWriteAcceptCharset(false);  // see SPR-7316

		this.messageConverters = new ArrayList<>(4);
		this.messageConverters.add(new ByteArrayHttpMessageConverter());
		this.messageConverters.add(stringHttpMessageConverter);
		try {
			this.messageConverters.add(new SourceHttpMessageConverter<>());
		}
		catch (Error err) {
			// Ignore when no TransformerFactory implementation is available
		}
		this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
	}
}
```

实现接口方法

```java
protected ModelAndView handleInternal(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    // 检查request
    ModelAndView mav;
    checkRequest(request);

    // 方法主体, 在不同的情况下调用 invokeHandlerMethod() 方法
    // 如果需要在Session上进行同步, 如果取到Session,
    // 就说明这个request不是一个新请求.需要在Session上同步.
    // 没有session说明是第一次访问, 无需同步
    // 如果没有设置需要在Session上同步, 那么也是直接调用.
    if (this.synchronizeOnSession) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            Object mutex = WebUtils.getSessionMutex(session);
            synchronized (mutex) {
                mav = invokeHandlerMethod(request, response, handlerMethod);
            }
        }
        else {
            // No HttpSession available -> no mutex necessary
            mav = invokeHandlerMethod(request, response, handlerMethod);
        }
    }
    else {
        // No synchronization on session demanded at all...
        mav = invokeHandlerMethod(request, response, handlerMethod);
    }

    if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
        if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
            applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
        }
        else {
            prepareResponse(response);
        }
    }

    return mav;
}
```

主要执行的方法

```java
// 调用 @RequestMapping handler method, 准备一个 ModleAndView如果视图解析是必要的.
// createInvocableHandlerMethod(HandlerMethod)
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        //1 创建一个 WebDataBinder 工厂. 会找出需要调用的@InitBinder方法, 本地的和全局的.
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);

        //2. 创建一个 ModelFactory, 这个工厂在@RequestMapping方法调用前协助model初始化,
        // 1. 负责根据@SessionAttributes注解尝试从Session获取指定值,
        // 2. 调用 @ModelAttribute 方法填充 model
        // 这里调用的 getModelFactory() 方法只是, 准备需要调用的@ModelAttribute方法, 和@SessionAttributes信息.
        // 还没有进行执行方法和填充model.
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

        //3. 创建一个包装器, 为了将这些组件用起来, 包装器设计模式. 核心
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        invocableMethod.setDataBinderFactory(binderFactory);
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        // flash属性, 填充默认 model
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));

        // 4. 重点, 调用 ModelFactory.initModel()方法完成model初始化
        // 获取Session
        // 调用ModelAttribute方法.
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);
        mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

        AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
        asyncWebRequest.setTimeout(this.asyncRequestTimeout);

        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        asyncManager.setTaskExecutor(this.taskExecutor);
        asyncManager.setAsyncWebRequest(asyncWebRequest);
        asyncManager.registerCallableInterceptors(this.callableInterceptors);
        asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

        if (asyncManager.hasConcurrentResult()) {
            Object result = asyncManager.getConcurrentResult();
            mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
            asyncManager.clearConcurrentResult();
            LogFormatUtils.traceDebug(logger, traceOn -> {
                String formatted = LogFormatUtils.formatValue(result, !traceOn);
                return "Resume with async result [" + formatted + "]";
            });
            invocableMethod = invocableMethod.wrapConcurrentResult(result);
        }

        // 执行handlerMethod方法
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
        if (asyncManager.isConcurrentHandlingStarted()) {
            return null;
        }

        return getModelAndView(mavContainer, modelFactory, webRequest);
    }
    finally {
        webRequest.requestCompleted();
    }
}
```

1.

> 这个 DataBinder 就是Spring的数据绑定器, 可以将数据绑定到对象上.主要解决@InitBinder 方法
> 说简单点就是一个会将数据set到目标对象的工具.
> WebDataBinder 是它的Web版本, 支持web环境
> Spring 提供了许多不同用途的 WebDataBinder,
> 这里我们获取到一个 ServletRequestDataBinderFactory.
> 负责创建 ServletRequestDataBinder , 这个binder可以执行servlet请求参数到JavaBean的数据绑定, 包括 Multipart part.

2. 创建ModelFactory对象实例, 依赖于第一步创建的 WebDataBinderFactory 实例.

```java
private ModelFactory getModelFactory(HandlerMethod handlerMethod, WebDataBinderFactory binderFactory) {
    // 构造一个 SessionAttributesHandler, 它是 ModelFactory 用来处理 @SessionAttributes 的工具.
    // 这里获取这个工具是拥有自己的缓存的, 只有第一次是创建, 之后都是重复利用. 毕竟@SessionAttributes 是类级别的注解.
    // 同一个控制器中不同的 @RequestMapping 方法共享同一个.
    SessionAttributesHandler sessionAttrHandler = getSessionAttributesHandler(handlerMethod);

    Class<?> handlerType = handlerMethod.getBeanType();
    // 这里Spring MVC 为 @ModelAttribute 方法做了缓存, 避免每次都需要读取Class对象上的方法.
    // 从缓存中获取对应 Controllor 类上的 @ModelAttribute 方法.
    // 没有获取到, 说明是第一次调用这个handler类的方法, 通过一个静态工具类将对应的方法选出并添加到缓存中.
    Set<Method> methods = this.modelAttributeCache.get(handlerType);
    if (methods == null) {
        // 这里的静态方法不在此详细讲, 通过静态常量 MethodFilter
        // 过滤出那些被 @ModelAttribute 注解的同时没有被 @RequestMapping 注解的方法.
        methods = MethodIntrospector.selectMethods(handlerType, MODEL_ATTRIBUTE_METHODS);
        this.modelAttributeCache.put(handlerType, methods);
    }

    // ModelFactory 需要调用的 @ModelAttribute 方法集合. 由全局的和局部的一起构成.
    // 至于为什么是 InvocableHandlerMethod 吗, 首先纠正我自己之前对HandlerMethod的错误理解.
    // HandlerMethod 是一个Method的包装, 也就是说它不一定是 @RequestMapping 方法的包装.
    // 只不过RequestMappingHandlerMapping只对 @RequestMapping 创建 HandlerMethod 对象而已, 并不是它只能这么用.
    // 它其实可用在各种需要调用 Method 对象的地方, 它可以包装任何的 Method 对象, 简化调用过程.
    // 所以这里用 HandlerMethod 来包装 @ModelAttribute 方法简化调用过程,
    // 当然 InvocableHandlerMethod 是HandlerMethod 的子类.
    List<InvocableHandlerMethod> attrMethods = new ArrayList<>();

    // 这个是 @ControllerAdvice 中的 @ModelAttribute 方法集, 是在RequestMappingHandlerAdapter初始化时得到的.
    // Global methods first, 先应用全局的 @ModelAttribute 方法.
    // 封装成 InvocableHandlerMethod 对象, 放入 attraMethods 中
    this.modelAttributeAdviceCache.forEach((clazz, methodSet) -> {
        if (clazz.isApplicableToBeanType(handlerType)) {
            Object bean = clazz.resolveBean();
            for (Method method : methodSet) {
                attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
            }
        }
    });
    // 当前Controller的 @ModelAttribute 方法, 在缓存中获取, 或第一次从class上获取
    // 同上, 封装成 InvocableHandlerMethod 对象, 放入 attraMethods 中
    for (Method method : methods) {
        Object bean = handlerMethod.getBean();
        attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
    }
    // 创建ModelaFactory.
    //attrMethods, 包含所有需要调用的 @ModelAttribute 方法包含局部和全局的.
    //sessionAttrHandler, 包含这个控制器上的 @SessionAttributes 注解
    return new ModelFactory(attrMethods, binderFactory, sessionAttrHandler);
}
```

4.

按以下属性填充模型:
> 1. 检索列为@SessionAttributes的"已知" session 属性
> 2. 调用 @ModelAttribute 方法
> 3. 查找也列为@SessionAttributes的 @ModelAttribute 方法参数, 确保它们存在于模型中.

```java
modelFactory.initModel(webRequest, mavContainer, invocableMethod);
public void initModel(NativeWebRequest request, ModelAndViewContainer container, HandlerMethod handlerMethod) throws Exception {

    // 获取根据@SessionAttributes注解获取 Seesion 中对应属性.
    // 具体就是调用request.getSession(false).getAttribute();
    Map<String, ?> sessionAttributes = this.sessionAttributesHandler.retrieveAttributes(request);

    // 添加到ModelAndView容器中, 准确的说是添加到Container的 Model中.
    container.mergeAttributes(sessionAttributes);

    // 调用创建ModelFactory时传入的所有的 @ModelAttribute 方法.
    invokeModelAttributeMethods(request, container);

    for (String name : findSessionAttributeArguments(handlerMethod)) {
        if (!container.containsAttribute(name)) {
            Object value = this.sessionAttributesHandler.retrieveAttribute(request, name);
            if (value == null) {
                throw new HttpSessionRequiredException("Expected session attribute '" + name + "'", name);
            }
            container.addAttribute(name, value);
        }
    }
}

// 调用 @ModelAttribute 方法, 来填充Model, 当且仅当Model不存在.
// 当这个方法调用后, 当前的Model上就拥有了 @ModelAttribute 方法声明的模型数据.
private void invokeModelAttributeMethods(NativeWebRequest request, ModelAndViewContainer container)
        throws Exception {

    while (!this.modelMethods.isEmpty()) {
        // 调用下一个 @ModelAttribute 方法, 这个下一个并不是注册时的顺序, 而是和其依赖有关的.
        // 如果一个 @ModelAttribute 方法的参数中出现了 @ModelAttribute 注解, 那么这个方法就依赖于对应的值存在时才应该调用.
        // 尽量保证调用每个 @ModelAttribute 方法之前, 能够正确填充其方法参数.
        InvocableHandlerMethod modelMethod = getNextModelMethod(container).getHandlerMethod();
        ModelAttribute ann = modelMethod.getMethodAnnotation(ModelAttribute.class);
        Assert.state(ann != null, "No ModelAttribute annotation");
        if (container.containsAttribute(ann.name())) {
            if (!ann.binding()) {
                container.setBindingDisabled(ann.name());
            }
            continue;
        }

        // 调用当前的 @ModelAttribute 方法, 然后如果有返回值的话, 将其和name 注册到 ModelAndViewContainer 中,
        // 其实是内部的Model上, 给后面的方法调用时提供参数解析.
        // 这个key的确定是: 1.先使用注解声明的值, 如果有的话, 2. 否则为返回值生成一个name, 规则挺有意思的.
        Object returnValue = modelMethod.invokeForRequest(request, container);
        if (!modelMethod.isVoid()){
            String returnValueName = getNameForReturnValue(returnValue, modelMethod.getReturnType());
            if (!ann.binding()) {
                container.setBindingDisabled(returnValueName);
            }
            if (!container.containsAttribute(returnValueName)) {
                // 重点, 绑定模型数据到模型上.
                container.addAttribute(returnValueName, returnValue);
            }
        }
    }
}
```

