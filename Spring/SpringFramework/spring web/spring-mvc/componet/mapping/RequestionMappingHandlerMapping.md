# RequestMappingHandlerMapping
![基于注解的HandlerMapping的类图]()

# AbstractHandlerMethodMapping\<T>
用于定义request和HandlerMethod之间的映射.
对于每个已注册的 HandlerMethod 维护唯一的映射关系.

这里的泛型很有意思, 之前的 AbstractUrlHandlerMapping 定义了 url 和 handler 之间的映射关系.
也就是说它是通过request的请求路径,
进行判断应该映射到哪一个 handler 对象, 它没有规定Handler的类型
handler 可以是任何类型, 比如说 HandlerMethod, Controller接口...等等

但是这个类不同的是, 它没有规定如何映射到handler, 
它只规定了handler的类型, 必须是 HandlerMethod.

并没有规定什么样的信息映射到HandlerMethod, 这一点有子类声明, 当然既然是http request, 我们可以用上面携带的信息进行映射, 就像RequestMappingHnalderMapping子类一样.

我们知道 RequestMapingHandlerMapping 不仅仅通过url来映射request, 
还可以用 header, requestParam, cookie 等等做出一系列的具体映射.
比基于 url 的映射很加的细粒度, 更加自由.

```java
public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {

    // 静态常量
	private static final String SCOPED_TARGET_NAME_PREFIX = "scopedTarget.";
	private static final HandlerMethod PREFLIGHT_AMBIGUOUS_MATCH =
			new HandlerMethod(new EmptyHandler(), ClassUtils.getMethod(EmptyHandler.class, "handle"));
	private static final CorsConfiguration ALLOW_CORS_CONFIG = new CorsConfiguration();

	static {
		ALLOW_CORS_CONFIG.addAllowedOrigin("*");
		ALLOW_CORS_CONFIG.addAllowedMethod("*");
		ALLOW_CORS_CONFIG.addAllowedHeader("*");
		ALLOW_CORS_CONFIG.setAllowCredentials(true);
	}

    // 是否要从 父容器中获取 HandlerMethod对象, 默认值false, 有setter方法没有getter方法
	private boolean detectHandlerMethodsInAncestorContexts = false;

    // 有getter, setter. 这是一个命名策略, 在注册时会给handler起名字
	@Nullable
	private HandlerMethodMappingNamingStrategy<T> namingStrategy;

    // 一个内部类(注意不是静态内部类), 它就是保存映射关系的地方.
	private final MappingRegistry mappingRegistry = new MappingRegistry();

}
```

实现 getHandlerInternal() 方法
```java
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
    String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
    // 从注册表上获取锁, 因为lookupHanlderMethod()方法会访问注册表,为了并发访问的安全性.
    this.mappingRegistry.acquireReadLock();
    try {
        HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
        return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
    }
    finally {
        this.mappingRegistry.releaseReadLock();
    }
}

// 具体的逻辑. 
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
    // Match ,就是一组 T 映射信息 和 HandlerMethod 的容器, 类似于 Map 类中 Entry 代表一对key value 一样.
    // 一个List<Match> 集合代表, 可能查找出来多个匹配项.
    List<Match> matches = new ArrayList<>();
    // 第一步通过 url 和 T (映射信息) 匹配, 获取到对应的T
    List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
    if (directPathMatches != null) {
        addMatchingMappings(directPathMatches, matches, request);
    }
    // 这个行为很奇怪, 如果经过上面步骤, matches 还是空的就将所有的都认为是匹配项, 添加到集合中. 不是应该没有匹配项了吗
    if (matches.isEmpty()) {
        // No choice but to go through all mappings...
        addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
    }

    // 如果集合不是空的, 就对其用指定的 Comparator 进行排序, 获得 best match.
    if (!matches.isEmpty()) {
        // 获取比较器, 看起来我们好像是new 一个 MatchComparator对象执行, 其实它是 comparator 的包装对象.
        // 真正的 comparator 由getMappingComparator()方法提供. 然后它是一个抽象方法由具体子类实现.
        // 比较T的任务当然要交给子类呀, 毕竟泛型都没有填.
        Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
        matches.sort(comparator);
        Match bestMatch = matches.get(0);
        // 如果由多个匹配项的情况下, 进行排序, 第一项和第二项可能是相等的, 也就是说是同样的匹配等级. 
        // 这时需要报错指明这个url匹配两个method.
        // 记住排序完成的集合的首项并不一定是最大的, 还可以有并列的情况.
        if (matches.size() > 1) {
            if (logger.isTraceEnabled()) {
                logger.trace(matches.size() + " matching mappings: " + matches);
            }
            if (CorsUtils.isPreFlightRequest(request)) {
                return PREFLIGHT_AMBIGUOUS_MATCH;
            }
            Match secondBestMatch = matches.get(1);
            if (comparator.compare(bestMatch, secondBestMatch) == 0) {
                Method m1 = bestMatch.handlerMethod.getMethod();
                Method m2 = secondBestMatch.handlerMethod.getMethod();
                String uri = request.getRequestURI();
                throw new IllegalStateException(
                        "Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");
            }
        }

        // 然后将这个最佳匹配的 HandlerMethod 放到 Request Attribute 中. 让request携带这它,
        request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.handlerMethod);
        // 将 lookupPath 以 HandlerMapping.pathWithinHandlerMapping 为key 放到 Request Attribute中.
        // 这个方法的目的主要是让子类扩展的, 具体子类还会向 request 中放一些有用的属性.
        handleMatch(bestMatch.mapping, lookupPath, request);
        return bestMatch.handlerMethod;
    }
    else {
        return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
    }
}

// 向给定的 List<Match> 集合中添加
private void addMatchingMappings(Collection<T> mappings, List<Match> matches, HttpServletRequest request) {
    for (T mapping : mappings) {
        // 抽象方法交给子类实现, 检查 T 是否和 request 匹配, 这些 T 都是通过url匹配得到的
        // 没有验证其他信息. 通过这个 getMatchingMapping 方法确定是否真正匹配, 匹配就返回一个新的实例, 不匹配返回null.
        T match = getMatchingMapping(mapping, request);
        // 如果 match 不为null, 说明 match 就是匹配这个request的T, 
        if (match != null) {
            // 向List中添加对象, T 对象已经确定, 通过mappingRegistry获取对应 HandlerMethod
            matches.add(new Match(match, this.mappingRegistry.getMappings().get(mapping)));
        }
    }
}
```



## init

这个类的初始化没有依赖ApplicationContextAware接口, 而是通过 InitlizBean接口时间的
```java
public void afterPropertiesSet() {
    initHandlerMethods();
}

// 扫描 ApplicationContext 的 bean, 并注册扫描到的 handlerMethod
protected void initHandlerMethods() {
    // getCandidateBeanNames() 获取所有候选bean的名字. 就是获取ApplicationContext中的所有Obeject.clas 的Bean
    for (String beanName : getCandidateBeanNames()) {
        // beanName 不能以 scopedTarget 开头.
        if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
            processCandidateBean(beanName);
        }
    }
    handlerMethodsInitialized(getHandlerMethods());
}

// 1
// 确定 Bean 的类型, 如果是确实是一个Handler Bean的话就调用 detectHanlderMethods()方法.
// lazy 的Bean会被忽略, 因为此时它还没有被创建.
protected void processCandidateBean(String beanName) {
    Class<?> beanType = null;
    try {
        beanType = obtainApplicationContext().getType(beanName);
    }
    catch (Throwable ex) {
        // An unresolvable bean type, probably from a lazy bean - let's ignore it.
        if (logger.isTraceEnabled()) {
            logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
        }
    }

    // isHandler() 确定bean是否为一个合格的Handler, 抽象方法由子类实现.
    // 子类的实现是判断 Class 是否有 @Controller 或 @RequestMapping注解.
    // 因为IOC容器可以自己注册单例Bean, 让容器帮忙管理, 所有不是所有的Bean上都会有@Controller(它只等价于@Bean).
    // 如果我们自己构建一个@RequsetMapping的对象然后注册到IOC容器中, 那么它就只有@RequestMapping注解, 没有@Controller.
    // 所以, Spring MVC 认为, 两个注解只有一个就认为是Handler Bean.
    if (beanType != null && isHandler(beanType)) {
        detectHandlerMethods(beanName);
    }
}

// 2 明确HandlerMethod, 我们定义的HandlerMethod都在类上面的, 之前只是找到Controller类.
// 接下来, 获取Controller类中的HandlerMethod方法, 重点不是获取Method对象因为获取它很容易.
// 重点是根据Method对象构建 T 对象的信息. 这个过程封装在getMappingForMethod(method, userType)中
protected void detectHandlerMethods(Object handler) {
    // 上面一直是使用BeanName作为操作的对象, 应该是为了节省内存.
    Class<?> handlerType = (handler instanceof String ?
            obtainApplicationContext().getType((String) handler) : handler.getClass());

    if (handlerType != null) {
        // 确定用户定义类型, 确保不是通过 CGlib 生成的代理.
        Class<?> userType = ClassUtils.getUserClass(handlerType);

        // 调用了一个静态方法, selectMethods() 方法会选出类上的方法, 然后对方法执行这里传入的 lamda.
        // 这个lamda 调用了一个抽象方法 getMappingForMethod().
        // 这样就得到了每一个 HandlerMethod - T 的集合. 一个控制器类中的映射信息就获取完毕了.
        Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
                (MethodIntrospector.MetadataLookup<T>) method -> {
                    try {
                        // 重点, 调用这个抽象方法返回值是根据method创建出的映射信息对象实例. 在子类中就是RequestMappingInfo
                        return getMappingForMethod(method, userType);
                    }
                    catch (Throwable ex) {
                        throw new IllegalStateException("Invalid mapping on handler class [" +
                                userType.getName() + "]: " + method, ex);
                    }
                });
        if (logger.isTraceEnabled()) {
            logger.trace(formatMappings(userType, methods));
        }
        // 注册找到的映射
        methods.forEach((method, mapping) -> {
            Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
            registerHandlerMethod(handler, invocableMethod, mapping);
        });
    }
}


// 它会根据传入的Method, Class 生成对用的 T(映射信息), 具体子类已经实现了这个方法.
// 获取方法上的 @RequestMapping 对象, 然后根据信息生成 RequestMappingInfo.
protected abstract T getMappingForMethod(Method method, Class<?> handlerType);

// 注册对应的映射关系. handler 是我们的 @Controller 类实例, method 是一个@RequestMapping方法, T 是根据 Method 上的信息(也就是@RequestMappin注解中的信息)生成的映射信息对象.
protected void registerHandlerMethod(Object handler, Method method, T mapping) {
    this.mappingRegistry.register(mapping, handler, method);
}


// 3
// 为了给子类扩展用的, 在handlerMethod注册完成后调用, 这里简单的打印了一个日志
protected void handlerMethodsInitialized(Map<T, HandlerMethod> handlerMethods) {
    // Total includes detected mappings + explicit registrations via registerMapping
    int total = handlerMethods.size();
    if ((logger.isTraceEnabled() && total == 0) || (logger.isDebugEnabled() && total > 0) ) {
        logger.debug(total + " mappings in " + formatMappingName());
    }
}
```


## MappingRegistry
上面的执行过程和初始化流程中涉及到了 MappingRegistry 这个注册表.(谁让它保存着映射信息呢)
所以接下来让我们探究一下MappingRegistry是如何保存映射信息.

```java
class MappingRegistry {

    // 所有信息的总集, 在注册完成其他信息后都会向这里注册一份.
    // 主要用在 unregister(T) 方法中, 首先根据T获取所有对应信息,然后从下面的各个 map 中移除对应项.
    private final Map<T, MappingRegistration<T>> registry = new HashMap<>();

    // 这个很简单, 就是 RequestInfo 和 HandlerMethod 的映射, 
    // 这样我们获得最佳 RequestInfo 之后就可以根据其找到被映射的 HandlerMethod 处理器.
    private final Map<T, HandlerMethod> mappingLookup = new LinkedHashMap<>();

    // 保存 url 和 List<T> 的映射关系, 把T看成RequestInfo会更好理解, 一个url会对应多个RequestInfo.
    // 如@PostMapping(path="/user") 和 @GetMapping(path="/user")映射的是同一个url, 但是它们对应的方法不同.
    // 根据其他信息如: requestMethod, header, cookie 等信息自然可以分辨出最佳handler.
    // MultiValueMap<String, T>可以看成 Map<String, List<T>>
    private final MultiValueMap<String, T> urlLookup = new LinkedMultiValueMap<>();

    // name 和 HandlerMethod 映射, 还没搞懂有什么用
    private final Map<String, List<HandlerMethod>> nameLookup = new ConcurrentHashMap<>();

    // Cors 配置信息
    private final Map<HandlerMethod, CorsConfiguration> corsLookup = new ConcurrentHashMap<>();

    // 线程安全
    private final ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();

    // 没有声明构造器 
}
```

首先我们看到运行过程中的调用方法
getMappingByUrl()方法非常的简单, 
MultiValueMap可以看出 Map\<String, List\<T\>\>的简写.
所以对其调用 get()方法返回的自然是 List\<T\>.
```java
public List<T> getMappingsByUrl(String urlPath) {
    return this.urlLookup.get(urlPath);
}
```

这个方法在addMatchingMappings中调用, getMappings().get(T); 的调用获取了 HandlerMethod 组装成 Match对象.
```java
public Map<T, HandlerMethod> getMappings() {
    return this.mappingLookup;
}
```

这个方法在初始化时注册映射关系时用到.
```java
// 这个是上面调用的内部类 mappingRegistry 的方法.
// 什么有三个参数呢, 想要调用一个 Method 对象, 必须传入其对应的类的实例, 和方法参数. Method.invoke(Object, args); 
// 这里的 handler 就是我们的被 @Controllor 注解的类的实例对象的BeanName

// 因为根据初始化时的调用关系这边的Handler对象, 其实还只是一个BeanName, 
// 所以在 createHandlerMethod()方法中有进行了判断和获取,
// 至于为什么明明就是个字符串还声明成Object呢, 当然是为了适用性, 这样在显式的注册映射信息的时候也不会错.
// 当有我们显示的通过外部类的 register() 方法注册.
public void register(T mapping, Object handler, Method method) {
    this.readWriteLock.writeLock().lock();
    try {
        // HandlerMethod 包装了 handler 和对应的 Method, 这样就可调用了.
        // 这个 createHanlderMethod()方法是外部类的, 因为是内部类所以可以直接调用
        HandlerMethod handlerMethod = createHandlerMethod(handler, method);

        // 确定没有注册过这一对 mapping - HandlerMethod 的映射, 重复注册会报错.
        // 就是检查 mappingLookup 中有没有这一项
        assertUniqueMethodMapping(handlerMethod, mapping);

        // 向 Map<T, HandlerMethod> 插入.
        this.mappingLookup.put(mapping, handlerMethod);

        // 然后会获取这个T RequestInfo 对应的直接 Url 集合, 一个@RequestMapping 可能对应多个url.
        // 因为在 @RequestMapping 中path 被定义为 String [];
        // 我们可以这样写 @GetRequestMapping(path={"/user", "/account"});
        // 并给每个url 都注册mapping, 注意这里不是直接的 url-mapping, 而是 url-list<mapping> 
        // 这个urlLookup是一个方便类.add()方法会自己向里对应的List中添加.
        List<String> directUrls = getDirectUrls(mapping);
        for (String url : directUrls) {
            this.urlLookup.add(url, mapping);
        }

        // 如果 NamingStrategy 存在, 就进行名字的注册, 这个NamingStrategy在当前类中没有被初始化,
        // 在具体子类 RequestMappingHandlerMapping 的默认构造器中设置了默认值.
        // 默认的命名策略是, 确定@RequestMapping没有显式设置name属性的情况下,
        // 用HandlerMethod属于的控制器类的简单类名的全大写 + # + Method.getName()
        // 如 DEMOHANDLER#getUser
        String name = null;
        if (getNamingStrategy() != null) {
            name = getNamingStrategy().getName(handlerMethod, mapping);
            addMappingName(name, handlerMethod);
        }

        // Cors配置的解析 initCorsConfiguration()方法是外部类的, 它是一个空实现
        // 而子类覆盖了它, 负责读取类上和方法上的 @CrossOrigin 注解, 根据注解信息生成CorsConfiguration.
        CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
        if (corsConfig != null) {
            this.corsLookup.put(handlerMethod, corsConfig);
        }

        // 向总信息放置一份方便根据T 删除所有信息.
        this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));
    }
    finally {
        this.readWriteLock.writeLock().unlock();
    }
}
```

## 总结

AbstractHandlerMethodMapping 中一共有5个抽象方法, 分别在不同的逻辑中调用.
```java

// 在初始化逻辑中, 用于判断获取的BeanClass是不是目标, 即是否有@Contrllor或@RequestMapping注解.
protected abstract boolean isHandler(Class<?> beanType);

// 在初始化逻辑中, 确定Bean之后, 对Bean上的每个方法调用. 
// 给定Method, Class 获取具体的映射信息.
// 首先读取 Method 上的@RequestMapping注解, 然后读取其所属的控制器类上的@RequestMapping, 最后两者合并成为最终的请求信息.
protected abstract T getMappingForMethod(Method method, Class<?> handlerType);

// 在注册 url-T 对象关系映射的时候调用, 这个方法只被内部类 MappingRegistry 调用.
// 用来获取一个@RequestMapping对应的多个url.
protected abstract Set<String> getMappingPathPatterns(T mapping);

// 在运行时, 用于获取所有的匹配项. 根据请求和T重新创建一个新的T.
protected abstract T getMatchingMapping(T mapping, HttpServletRequest request);

// 在运行时, 用于获取最匹配的选项.
protected abstract Comparator<T> getMappingComparator(HttpServletRequest request);
```


# RequestMappingInfoHandlerMapping

