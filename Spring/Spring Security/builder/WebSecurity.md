# WebSecurity
这个最简单的子类, 没有涉及到额外的接口.
它继承了 AbstractConfiguredSecurityBuilder 类并填上了泛型是 Filter, 代表它是一个创建 Filter 的Builder.
然后它还实现了 SecurityBuilder\<Filter\>, 这样它就符合 AbstractConfiguredSecurityBuilder 的第二个泛型要求.


它将 HttpSecurity 和 AuthenticationManager 联合起来.

它是负责构建Boot流程中的 FilterChainProxy 对象.
构建这个FilterChainProxy需要 SecurityFilterChain 对象,
而HttpSecurity就是负责构建 SecurityFilterChain 的Builder.
构建一个HttpSecurity需要还需要 AuthenticationManager, 
而 @EnanbleGlobalAuthentication 注解会在IOC中注册一个 AuthenticationManagerBuilder 和 ObjectProcessor.

通过阅读WebSecurity的源码我们会发现, 
这个类没有直接使用 HttpSecurity 和 AuthenticationManagerBuilder.
而在Spring提供的配置类中我们可以找到这个Builder的使用方式.

```java
public final class WebSecurity 
        extends AbstractConfiguredSecurityBuilder<Filter, WebSecurity>
        implements SecurityBuilder<Filter>, ApplicationContextAware {

	private final Log logger = LogFactory.getLog(getClass());

	private final List<RequestMatcher> ignoredRequests = new ArrayList<>();

    // 就是 HttpSecurity 
	private final List<SecurityBuilder<? extends SecurityFilterChain>> securityFilterChainBuilders = new ArrayList<SecurityBuilder<? extends SecurityFilterChain>>();

	private IgnoredRequestConfigurer ignoredRequestRegistry;

	private FilterSecurityInterceptor filterSecurityInterceptor;

	private HttpFirewall httpFirewall;

	private boolean debugEnabled;

    //
	private WebInvocationPrivilegeEvaluator privilegeEvaluator;

    // SpringEL 的默认上下文对象
	private DefaultWebSecurityExpressionHandler defaultWebSecurityExpressionHandler = new DefaultWebSecurityExpressionHandler();

	private SecurityExpressionHandler<FilterInvocation> expressionHandler = defaultWebSecurityExpressionHandler;

	private Runnable postBuildAction = new Runnable() {
		public void run() {
		}
	};
            

    // 构造器, 不允许重复配置 
	public WebSecurity(ObjectPostProcessor<Object> objectPostProcessor) {
		super(objectPostProcessor);
	}
}
```

```java
public WebSecurity httpFirewall(HttpFirewall httpFirewall) {
    this.httpFirewall = httpFirewall;
    return this;
}

public WebSecurity debug(boolean debugEnabled) {
    this.debugEnabled = debugEnabled;
    return this;
}

// 重要方法
public WebSecurity addSecurityFilterChainBuilder(
        SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder) {
    this.securityFilterChainBuilders.add(securityFilterChainBuilder);
    return this;
}

public WebSecurity privilegeEvaluator(
        WebInvocationPrivilegeEvaluator privilegeEvaluator) {
    this.privilegeEvaluator = privilegeEvaluator;
    return this;
}

public WebSecurity expressionHandler(
        SecurityExpressionHandler<FilterInvocation> expressionHandler) {
    Assert.notNull(expressionHandler, "expressionHandler cannot be null");
    this.expressionHandler = expressionHandler;
    return this;
}
```

```java
public void setApplicationContext(ApplicationContext applicationContext)
        throws BeansException {
    this.defaultWebSecurityExpressionHandler
            .setApplicationContext(applicationContext);
    try {
        this.defaultWebSecurityExpressionHandler.setPermissionEvaluator(applicationContext.getBean(
                PermissionEvaluator.class));
    } catch(NoSuchBeanDefinitionException e) {}

    this.ignoredRequestRegistry = new IgnoredRequestConfigurer(applicationContext);
    try {
        this.httpFirewall = applicationContext.getBean(HttpFirewall.class);
    } catch(NoSuchBeanDefinitionException e) {}
}
```

```java
protected Filter performBuild() throws Exception {
    Assert.state(
            !securityFilterChainBuilders.isEmpty(),
            () -> "At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. "
                    + "Typically this done by adding a @Configuration that extends WebSecurityConfigurerAdapter. "
                    + "More advanced users can invoke "
                    + WebSecurity.class.getSimpleName()
                    + ".addSecurityFilterChainBuilder directly");
    int chainSize = ignoredRequests.size() + securityFilterChainBuilders.size();
    List<SecurityFilterChain> securityFilterChains = new ArrayList<>(
            chainSize);
    for (RequestMatcher ignoredRequest : ignoredRequests) {
        securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
    }
    // 重点
    for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
        securityFilterChains.add(securityFilterChainBuilder.build());
    }
    FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
    if (httpFirewall != null) {
        filterChainProxy.setFirewall(httpFirewall);
    }
    filterChainProxy.afterPropertiesSet();

    Filter result = filterChainProxy;
    if (debugEnabled) {
        logger.warn("\n\n"
                + "********************************************************************\n"
                + "**********        Security debugging is enabled.       *************\n"
                + "**********    This may include sensitive information.  *************\n"
                + "**********      Do not use in a production system!     *************\n"
                + "********************************************************************\n\n");
        result = new DebugFilter(filterChainProxy);
    }
    postBuildAction.run();
    return result;
}
```


# 总结

如果光看这个 WebSecurity builder 并没有什么特殊, 没有人使用的Builder是不完整的.


这个配置类也是非常的简单.
首先利用Spring IOC功能注入所有的 WebSecurityConfigurer 实例,
其实就是找到 WebSecurityAdapter 的实现类(当然我们也可以不继承这个提供的适配器, 自己实现SecurityConfigurer来达到配置WebSecurity的目的) 
会在方法中 new WebSecurity()

```java
@Configuration
public class WebSecurityConfiguration implements ImportAware, BeanClassLoaderAware {
	private WebSecurity webSecurity;

	private Boolean debugEnabled;

	private List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers;

	private ClassLoader beanClassLoader;

	@Autowired(required = false)
	private ObjectPostProcessor<Object> objectObjectPostProcessor;

	@Bean
	public static DelegatingApplicationListener delegatingApplicationListener() {
		return new DelegatingApplicationListener();
	}

	@Bean
	@DependsOn(AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public SecurityExpressionHandler<FilterInvocation> webSecurityExpressionHandler() {
		return webSecurity.getExpressionHandler();
	}

    /*
    * 重点方法, 使用之前构建的 WebSecurity Builder构建Filter.
    * 主要是在使用的 build() 方法前, 会检测是否正确配置了 WebSecurity.
    * 保证一定会有一个 WebSecurtiyAdapter 被应用.
    */
	@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public Filter springSecurityFilterChain() throws Exception {
		boolean hasConfigurers = webSecurityConfigurers != null
				&& !webSecurityConfigurers.isEmpty();
        // 重点, 确保了安全系统的默认配置生效
        // 通过这里我们会发现, 对Builder的配置都写在 WebSecurityConfigurerAdapter 中.
		if (!hasConfigurers) {
			WebSecurityConfigurerAdapter adapter = objectObjectPostProcessor
					.postProcess(new WebSecurityConfigurerAdapter() {
					});
			webSecurity.apply(adapter);
		}
		return webSecurity.build();
	}

	@Bean
	@DependsOn(AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public WebInvocationPrivilegeEvaluator privilegeEvaluator() throws Exception {
		return webSecurity.getPrivilegeEvaluator();
	}

	@Autowired(required = false)
	public void setFilterChainProxySecurityConfigurer(
			ObjectPostProcessor<Object> objectPostProcessor,
			@Value("#{@autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()}") List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers)
			throws Exception {
        // 调用构造器并且使用默认的 生命周期回调ObjectPostProcessor执行各种生命周期方法和 Aware 方法
		webSecurity = objectPostProcessor
				.postProcess(new WebSecurity(objectPostProcessor));
		if (debugEnabled != null) {
			webSecurity.debug(debugEnabled);
		}

		Collections.sort(webSecurityConfigurers, AnnotationAwareOrderComparator.INSTANCE);

		Integer previousOrder = null;
		Object previousConfig = null;

        // 对 Configurer 进行排序
		for (SecurityConfigurer<Filter, WebSecurity> config : webSecurityConfigurers) {
			Integer order = AnnotationAwareOrderComparator.lookupOrder(config);
			if (previousOrder != null && previousOrder.equals(order)) {
				throw new IllegalStateException(
						"@Order on WebSecurityConfigurers must be unique. Order of "
								+ order + " was already used on " + previousConfig + ", so it cannot be used on "
								+ config + " too.");
			}
			previousOrder = order;
			previousConfig = config;
		}

        // 应用这些配置 WebSecurity 的 Configurer 对象
		for (SecurityConfigurer<Filter, WebSecurity> webSecurityConfigurer : webSecurityConfigurers) {
			webSecurity.apply(webSecurityConfigurer);
		}
		this.webSecurityConfigurers = webSecurityConfigurers;
	}

	@Bean
	public static AutowiredWebSecurityConfigurersIgnoreParents autowiredWebSecurityConfigurersIgnoreParents(
			ConfigurableListableBeanFactory beanFactory) {
		return new AutowiredWebSecurityConfigurersIgnoreParents(beanFactory);
	}

    // Oreder 语义的排序器
	private static class AnnotationAwareOrderComparator extends OrderComparator {
        //...
	}


    // 利用Spring的回调接口, 获取@EnableWebSecurity注解上的 debug 标志位信息. 启用打印内部信息
	public void setImportMetadata(AnnotationMetadata importMetadata) {
		Map<String, Object> enableWebSecurityAttrMap = importMetadata
				.getAnnotationAttributes(EnableWebSecurity.class.getName());
		AnnotationAttributes enableWebSecurityAttrs = AnnotationAttributes
				.fromMap(enableWebSecurityAttrMap);
		debugEnabled = enableWebSecurityAttrs.getBoolean("debug");
		if (webSecurity != null) {
			webSecurity.debug(debugEnabled);
		}
	}

	public void setBeanClassLoader(ClassLoader classLoader) {
		this.beanClassLoader = classLoader;
	}
}

```

