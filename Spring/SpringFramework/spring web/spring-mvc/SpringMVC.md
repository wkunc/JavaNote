# @EnableWebMvc

这个注解导入了一个 Spring 提供的一个配置类**org.springframework.web.servlet.config.annotation**
当然, 这个类只是对父类中的 Bean 提供了默认配置, 真正的Bean 信息在其父类中

# WebMvcConfigurationSupport

This class registers the following HandlerMappings:

1. RequestMappingHandlerMapping
2. HandlerMapping
3. BeanNameUrlHandlerMapping
4. HandlerMapping
5. HandlerMapping

Registers these HandlerAdapters:

1. RequestMappingHandlerAdapter
2. HttpRequestHandlerAdapter
3. SimpleControllerHandlerAdapter

Register a HandlerExceptionResolverComposite
with this chain of exception resolvers:

1. ExceptionHandlerExceptionResolver
2. ResponseStatusExceptionResolver
3. DefalutHandlerExceptionResolver

Registers an AntPathMatcher and a UrlPathHelper to be used by:

1. the **RequestMappingHandlerMapping**
2. the **HandlerMapping** for ViewControllers
3. and the **HanderMapping** for serving resouces

Note that those beans can be configured with a PathMatchConfigurer.
Both the RequestMappingHandlerAdapter and the ExceptionHandlerExceptionResolver
are configured with default instances of the following by default:

1. ContentNegotiationManager
2. DefaultFormattingConversionService
3. org.springframework.validation.beanvalidation.OptionalValidatorFactoryBean if a JSR-303 implementation is available
   on the classpath
4. range of HttpMessageConverters depending on the third-party libraries available on the classpath.

上面是类的描述

以下是自己的理解

# RequestMappingHandlerMapping

支持被 @RequestMapping 注解的方法

```java
@Bean
public RequestMappingHandlerMapping requestMappingHandlerMapping() {
    RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
    mapping.setOrder(0);
    //设置拦截器列表
    //在 getInterceptors()方法中调用了一个空方法是用来自定义用的
    mapping.setInterceptors(getInterceptors());
    //设置内容协商管理器
    mapping.setContentNegotiationManager(mvcContentNegotiationManager());
    //设置 CORS (跨域资源请求)
    mapping.setCorsConfigurations(getCorsConfigurations());

    // 路径匹配配置
    // getPathMatchConfigurer() 方法中有一个 configurePathMatch() 方法是用来自定义的
    PathMatchConfigurer configurer = getPathMatchConfigurer();
    if (configurer.isUseSuffixPatternMatch() != null) {
        mapping.setUseSuffixPatternMatch(configurer.isUseSuffixPatternMatch());
    }
    if (configurer.isUseRegisteredSuffixPatternMatch() != null) {
        mapping.setUseRegisteredSuffixPatternMatch(configurer.isUseRegisteredSuffixPatternMatch());
    }
    if (configurer.isUseTrailingSlashMatch() != null) {
        mapping.setUseTrailingSlashMatch(configurer.isUseTrailingSlashMatch());
    }
    UrlPathHelper pathHelper = configurer.getUrlPathHelper();
    if (pathHelper != null) {
        mapping.setUrlPathHelper(pathHelper);
    }
    PathMatcher pathMatcher = configurer.getPathMatcher();
    if (pathMatcher != null) {
        mapping.setPathMatcher(pathMatcher);
    }

    return mapping;
}
```

# PathMatcher 和 UrlPathHelper

```java
	@Bean
	public PathMatcher mvcPathMatcher() {
        //从 PathMatchConfigurer 中获取一个全局的 PathMatcher
        //它可以通过 PathMatchConfigurer 进行配置
		PathMatcher pathMatcher = getPathMatchConfigurer().getPathMatcher();
		return (pathMatcher != null ? pathMatcher : new AntPathMatcher());
	}

    @Bean
    public UrlPathHelper mvcUrlPathHelper() {
        UrlPathHelper pathHelper = getPathMatchConfigurer().getUrlPathHelper();
        return (pathHelper != null ? pathHelper : new UrlPathHelp());
    }
```

# ContentNegotiationManager

```java
    //使用了 Builder 模式
	@Bean
	public ContentNegotiationManager mvcContentNegotiationManager() {
		if (this.contentNegotiationManager == null) {
			ContentNegotiationConfigurer configurer = new ContentNegotiationConfigurer(this.servletContext);
			configurer.mediaTypes(getDefaultMediaTypes());
            // 这是一个空方法, 用来自定义
			configureContentNegotiation(configurer);
			this.contentNegotiationManager = configurer.buildContentNegotiationManager();
		}
		return this.contentNegotiationManager;
	}
```

# viewControllerHandlerMapping

```java
	@Bean
	public HandlerMapping viewControllerHandlerMapping() {
        //很明显这也是一个 Builder 模式
		ViewControllerRegistry registry = new ViewControllerRegistry(this.applicationContext);
        //这个方法是空方法用于自定义
		addViewControllers(registry);

		AbstractHandlerMapping handlerMapping = registry.buildHandlerMapping();
		handlerMapping = (handlerMapping != null ? handlerMapping : new EmptyHandlerMapping());
		handlerMapping.setPathMatcher(mvcPathMatcher());
		handlerMapping.setUrlPathHelper(mvcUrlPathHelper());
		handlerMapping.setInterceptors(getInterceptors());
		handlerMapping.setCorsConfigurations(getCorsConfigurations());
		return handlerMapping;
	}
```

# BeanNameUrlHandlerMapping

好像是用来解决 Controller bean 上的路径的

```java
	@Bean
	public BeanNameUrlHandlerMapping beanNameHandlerMapping() {
		BeanNameUrlHandlerMapping mapping = new BeanNameUrlHandlerMapping();
		mapping.setOrder(2);
		mapping.setInterceptors(getInterceptors());
		mapping.setCorsConfigurations(getCorsConfigurations());
		return mapping;
	}
```

# resourceHandlerMapping

```java
    //同样使用了 Builder 模式
	@Bean
	public HandlerMapping resourceHandlerMapping() {
		Assert.state(this.applicationContext != null, "No ApplicationContext set");
		Assert.state(this.servletContext != null, "No ServletContext set");

		ResourceHandlerRegistry registry = new ResourceHandlerRegistry(this.applicationContext,
				this.servletContext, mvcContentNegotiationManager(), mvcUrlPathHelper());
        // 同样的这个方法用于自定义
		addResourceHandlers(registry);

		AbstractHandlerMapping handlerMapping = registry.getHandlerMapping();
		if (handlerMapping != null) {
			handlerMapping.setPathMatcher(mvcPathMatcher());
			handlerMapping.setUrlPathHelper(mvcUrlPathHelper());
			handlerMapping.setInterceptors(new ResourceUrlProviderExposingInterceptor(mvcResourceUrlProvider()));
			handlerMapping.setCorsConfigurations(getCorsConfigurations());
		}
		else {
			handlerMapping = new EmptyHandlerMapping();
		}
		return handlerMapping;
	}
```

# mvcResourceUrlProvider

```java
    // A ResourceUrlProvider bean for use with MVC dispatcher
	@Bean
	public ResourceUrlProvider mvcResourceUrlProvider() {
		ResourceUrlProvider urlProvider = new ResourceUrlProvider();
		UrlPathHelper pathHelper = getPathMatchConfigurer().getUrlPathHelper();
		if (pathHelper != null) {
			urlProvider.setUrlPathHelper(pathHelper);
		}
		PathMatcher pathMatcher = getPathMatchConfigurer().getPathMatcher();
		if (pathMatcher != null) {
			urlProvider.setPathMatcher(pathMatcher);
		}
		return urlProvider;
	}
```

# defaultServletHandlerMapping

```java
	@Bean
	public HandlerMapping defaultServletHandlerMapping() {
        //还是 Builder 模式
		DefaultServletHandlerConfigurer configurer = new DefaultServletHandlerConfigurer(this.servletContext); //还是一个用于自定义的方法 configureDefaultServletHandling(configurer);
		HandlerMapping handlerMapping = configurer.buildHandlerMapping();
		return (handlerMapping != null ? handlerMapping : new EmptyHandlerMapping());
	}
```

# requestMappingHandlerAdapter

```java
	@Bean
	public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
		RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
		adapter.setContentNegotiationManager(mvcContentNegotiationManager());
		adapter.setMessageConverters(getMessageConverters());
		adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer());
		adapter.setCustomArgumentResolvers(getArgumentResolvers());
		adapter.setCustomReturnValueHandlers(getReturnValueHandlers());

		if (jackson2Present) {
			adapter.setRequestBodyAdvice(
					Collections.<RequestBodyAdvice>singletonList(new JsonViewRequestBodyAdvice()));
			adapter.setResponseBodyAdvice(
					Collections.<ResponseBodyAdvice<?>>singletonList(new JsonViewResponseBodyAdvice()));
		}

		AsyncSupportConfigurer configurer = new AsyncSupportConfigurer();
		configureAsyncSupport(configurer);
		if (configurer.getTaskExecutor() != null) {
			adapter.setTaskExecutor(configurer.getTaskExecutor());
		}
		if (configurer.getTimeout() != null) {
			adapter.setAsyncRequestTimeout(configurer.getTimeout());
		}
		adapter.setCallableInterceptors(configurer.getCallableInterceptors());
		adapter.setDeferredResultInterceptors(configurer.getDeferredResultInterceptors());

		return adapter;
	}
```

# FormattingConversionService

```java
	@Bean
	public FormattingConversionService mvcConversionService() {
		FormattingConversionService conversionService = new DefaultFormattingConversionService();
		addFormatters(conversionService);
		return conversionService;
	}
```

# mvcValidator

```java
	@Bean
	public Validator mvcValidator() {
		Validator validator = getValidator();
		if (validator == null) {
			if (ClassUtils.isPresent("javax.validation.Validator", getClass().getClassLoader())) {
				Class<?> clazz;
				try {
					String className = "org.springframework.validation.beanvalidation.OptionalValidatorFactoryBean";
					clazz = ClassUtils.forName(className, WebMvcConfigurationSupport.class.getClassLoader());
				}
				catch (ClassNotFoundException ex) {
					throw new BeanInitializationException("Could not find default validator class", ex);
				}
				catch (LinkageError ex) {
					throw new BeanInitializationException("Could not load default validator class", ex);
				}
				validator = (Validator) BeanUtils.instantiateClass(clazz);
			}
			else {
				validator = new NoOpValidator();
			}
		}
		return validator;
	}
```

# CompositeUriComponentsContributor

```java
	@Bean
	public CompositeUriComponentsContributor mvcUriComponentsContributor() {
		return new CompositeUriComponentsContributor(
				requestMappingHandlerAdapter().getArgumentResolvers(), mvcConversionService());
	}
```

# HttpRequestHandlerAdapter

```java
	@Bean
	public HttpRequestHandlerAdapter httpRequestHandlerAdapter() {
		return new HttpRequestHandlerAdapter();
	}
```

```java
	@Bean
	public SimpleControllerHandlerAdapter simpleControllerHandlerAdapter() {
		return new SimpleControllerHandlerAdapter();
	}
```

# HandlerExceptionResolver

```java
	@Bean
	public HandlerExceptionResolver handlerExceptionResolver() {
		List<HandlerExceptionResolver> exceptionResolvers = new ArrayList<HandlerExceptionResolver>();
		configureHandlerExceptionResolvers(exceptionResolvers);
		if (exceptionResolvers.isEmpty()) {
			addDefaultHandlerExceptionResolvers(exceptionResolvers);
		}
		extendHandlerExceptionResolvers(exceptionResolvers);
		HandlerExceptionResolverComposite composite = new HandlerExceptionResolverComposite();
		composite.setOrder(0);
		composite.setExceptionResolvers(exceptionResolvers);
		return composite;
	}
```

# ViewResolver

```java
	@Bean
	public ViewResolver mvcViewResolver() {
		ViewResolverRegistry registry = new ViewResolverRegistry(
				mvcContentNegotiationManager(), this.applicationContext);
		configureViewResolvers(registry);

		if (registry.getViewResolvers().isEmpty()) {
			String[] names = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
					this.applicationContext, ViewResolver.class, true, false);
			if (names.length == 1) {
				registry.getViewResolvers().add(new InternalResourceViewResolver());
			}
		}

		ViewResolverComposite composite = new ViewResolverComposite();
		composite.setOrder(registry.getOrder());
		composite.setViewResolvers(registry.getViewResolvers());
		composite.setApplicationContext(this.applicationContext);
		composite.setServletContext(this.servletContext);
		return composite;
	}
```

# HandlerMappingIntrospector

```java
	@Bean @Lazy
	public HandlerMappingIntrospector mvcHandlerMappingIntrospector() {
		return new HandlerMappingIntrospector();
	}
```
