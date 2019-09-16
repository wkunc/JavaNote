# 默认配置

# HttpHandler API 和其组件

```java
// WebHandler
@Bean
public DispatcherHandler webHandler() {
    return new DispatcherHandler();
}

// WebExceptionHandler
@Bean
@Order(0)
public WebExceptionHandler responseStatusExceptionHandler() {
    return new WebFluxResponseStatusExceptionHandler();
}

// WebFilter

// ServerCodecConfigurger
@Bean
@Order(0)
public WebExceptionHandler responseStatusExceptionHandler() {
    return new WebFluxResponseStatusExceptionHandler();
}

// LocalContextReslver
@Bean
public LocaleContextResolver localeContextResolver() {
    return createLocaleContextResolver();
}
protected LocaleContextResolver createLocaleContextResolver() {
    return new AcceptHeaderLocaleContextResolver();
}

// WebSessionManager , 没有在@EnableWebFlux 注解中. 而是在通过WebHttpHandlerBuilder构建
// HttpHandler 的时候加入的默认实现DefaultWebSessionManager.

// ForwardedHeaderTransformer, 没有默认配置, 不论是配置文件中, 还是Builder过程中都没有.
```

# DispatcherHandler 和 其组件
```java
@Bean
public DispatcherHandler webHandler() {
    return new DispatcherHandler();
}
```

默认配置了3个 HandlerMapping, 3个 HandlerAdapter, 和 4个 ResultHandler.
```java
@Bean
public RequestMappingHandlerMapping requestMappingHandlerMapping() {
    RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
    mapping.setOrder(0);
    mapping.setContentTypeResolver(webFluxContentTypeResolver());
    mapping.setCorsConfigurations(getCorsConfigurations());

    PathMatchConfigurer configurer = getPathMatchConfigurer();
    Boolean useTrailingSlashMatch = configurer.isUseTrailingSlashMatch();
    if (useTrailingSlashMatch != null) {
        mapping.setUseTrailingSlashMatch(useTrailingSlashMatch);
    }
    Boolean useCaseSensitiveMatch = configurer.isUseCaseSensitiveMatch();
    if (useCaseSensitiveMatch != null) {
        mapping.setUseCaseSensitiveMatch(useCaseSensitiveMatch);
    }
    Map<String, Predicate<Class<?>>> pathPrefixes = configurer.getPathPrefixes();
    if (pathPrefixes != null) {
        mapping.setPathPrefixes(pathPrefixes);
    }

    return mapping;
}

@Bean
public RouterFunctionMapping routerFunctionMapping() {
    RouterFunctionMapping mapping = createRouterFunctionMapping();
    mapping.setOrder(-1); // go before RequestMappingHandlerMapping
    mapping.setMessageReaders(serverCodecConfigurer().getReaders());
    mapping.setCorsConfigurations(getCorsConfigurations());

    return mapping;
}


@Bean
public HandlerMapping resourceHandlerMapping() {
    ResourceLoader resourceLoader = this.applicationContext;
    if (resourceLoader == null) {
        resourceLoader = new DefaultResourceLoader();
    }
    ResourceHandlerRegistry registry = new ResourceHandlerRegistry(resourceLoader);
    registry.setResourceUrlProvider(resourceUrlProvider());
    addResourceHandlers(registry);

    AbstractHandlerMapping handlerMapping = registry.getHandlerMapping();
    if (handlerMapping != null) {
        PathMatchConfigurer configurer = getPathMatchConfigurer();
        Boolean useTrailingSlashMatch = configurer.isUseTrailingSlashMatch();
        Boolean useCaseSensitiveMatch = configurer.isUseCaseSensitiveMatch();
        if (useTrailingSlashMatch != null) {
            handlerMapping.setUseTrailingSlashMatch(useTrailingSlashMatch);
        }
        if (useCaseSensitiveMatch != null) {
            handlerMapping.setUseCaseSensitiveMatch(useCaseSensitiveMatch);
        }
    }
    else {
        handlerMapping = new EmptyHandlerMapping();
    }
    return handlerMapping;
}

// Adapter

@Bean
public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
    RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
    adapter.setMessageReaders(serverCodecConfigurer().getReaders());
    adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer());
    adapter.setReactiveAdapterRegistry(webFluxAdapterRegistry());

    ArgumentResolverConfigurer configurer = new ArgumentResolverConfigurer();
    configureArgumentResolvers(configurer);
    adapter.setArgumentResolverConfigurer(configurer);

    return adapter;
}

@Bean
public HandlerFunctionAdapter handlerFunctionAdapter() {
    return new HandlerFunctionAdapter();
}

@Bean
public SimpleHandlerAdapter simpleHandlerAdapter() {
    return new SimpleHandlerAdapter();
}

// ResultHandler

@Bean
public ResponseEntityResultHandler responseEntityResultHandler() {
    return new ResponseEntityResultHandler(serverCodecConfigurer().getWriters(),
            webFluxContentTypeResolver(), webFluxAdapterRegistry());
}

@Bean
public ResponseBodyResultHandler responseBodyResultHandler() {
    return new ResponseBodyResultHandler(serverCodecConfigurer().getWriters(),
            webFluxContentTypeResolver(), webFluxAdapterRegistry());
}

@Bean
public ViewResolutionResultHandler viewResolutionResultHandler() {
    ViewResolverRegistry registry = getViewResolverRegistry();
    List<ViewResolver> resolvers = registry.getViewResolvers();
    ViewResolutionResultHandler handler = new ViewResolutionResultHandler(
            resolvers, webFluxContentTypeResolver(), webFluxAdapterRegistry());
    handler.setDefaultViews(registry.getDefaultViews());
    handler.setOrder(registry.getOrder());
    return handler;
}

@Bean
public ServerResponseResultHandler serverResponseResultHandler() {
    List<ViewResolver> resolvers = getViewResolverRegistry().getViewResolvers();
    ServerResponseResultHandler handler = new ServerResponseResultHandler();
    handler.setMessageWriters(serverCodecConfigurer().getWriters());
    handler.setViewResolvers(resolvers);
    return handler;
}
```
