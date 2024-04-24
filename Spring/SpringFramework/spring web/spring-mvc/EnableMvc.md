# HandlerMapping

在Spring官方提供的配置中有5个HandlerMapping

## DefaultServletHandlerMapping

这个 buildHandlerMapping() 方法返回的实际上是一个配置好的 SimpleUrlHandlerMapping
这个 HandlerMapping 是为了将静态资源访问转发回Servlet容器的 defaultServlet.
适用于 DispatcherServlet 映射了 "/" 的情况.
默认情况下不会配置, 因为 configurer.buildHandlerMapping() 方法默认返回null,
只有调用过 configurer.enable() 才会生效.

```java
@Bean
@Nullable
public HandlerMapping defaultServletHandlerMapping() {
    DefaultServletHandlerConfigurer configurer = new DefaultServletHandlerConfigurer(this.servletContext);
    configureDefaultServletHandling(configurer);
    return configurer.buildHandlerMapping();
}
```

## ResourceHandlerMapping

这个registry.getHandlerMapping() 返回的也是个 SimpleUrlHandlerMapping

```java
@Bean
@Nullable
public HandlerMapping resourceHandlerMapping() {
    // Assert语句...
    ResourceHandlerRegistry registry = new ResourceHandlerRegistry(this.applicationContext,
            this.servletContext, mvcContentNegotiationManager(), mvcUrlPathHelper());
    addResourceHandlers(registry);

    AbstractHandlerMapping handlerMapping = registry.getHandlerMapping();
    if (handlerMapping == null) {
        return null;
    }
    handlerMapping.setPathMatcher(mvcPathMatcher());
    handlerMapping.setUrlPathHelper(mvcUrlPathHelper());
    handlerMapping.setInterceptors(getInterceptors());
    handlerMapping.setCorsConfigurations(getCorsConfigurations());
    return handlerMapping;
}
```

## viewControllerHandlerMapping

这个registry.getHandlerMapping() 返回的也是个 SimpleUrlHandlerMapping

```java
@Bean
@Nullable
public HandlerMapping viewControllerHandlerMapping() {
    ViewControllerRegistry registry = new ViewControllerRegistry(this.applicationContext);
    addViewControllers(registry);

    AbstractHandlerMapping handlerMapping = registry.buildHandlerMapping();
    if (handlerMapping == null) {
        return null;
    }
    handlerMapping.setPathMatcher(mvcPathMatcher());
    handlerMapping.setUrlPathHelper(mvcUrlPathHelper());
    handlerMapping.setInterceptors(getInterceptors());
    handlerMapping.setCorsConfigurations(getCorsConfigurations());
    return handlerMapping;
}
```

## BeanNameHandlerMapping

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

## RequestMappingHandlerMapping

```java
@Bean
public RequestMappingHandlerMapping requestMappingHandlerMapping() {
    RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
    mapping.setOrder(0);
    mapping.setInterceptors(getInterceptors());
    mapping.setContentNegotiationManager(mvcContentNegotiationManager());
    mapping.setCorsConfigurations(getCorsConfigurations());

    PathMatchConfigurer configurer = getPathMatchConfigurer();

    Boolean useSuffixPatternMatch = configurer.isUseSuffixPatternMatch();
    if (useSuffixPatternMatch != null) {
        mapping.setUseSuffixPatternMatch(useSuffixPatternMatch);
    }
    Boolean useRegisteredSuffixPatternMatch = configurer.isUseRegisteredSuffixPatternMatch();
    if (useRegisteredSuffixPatternMatch != null) {
        mapping.setUseRegisteredSuffixPatternMatch(useRegisteredSuffixPatternMatch);
    }
    Boolean useTrailingSlashMatch = configurer.isUseTrailingSlashMatch();
    if (useTrailingSlashMatch != null) {
        mapping.setUseTrailingSlashMatch(useTrailingSlashMatch);
    }

    UrlPathHelper pathHelper = configurer.getUrlPathHelper();
    if (pathHelper != null) {
        mapping.setUrlPathHelper(pathHelper);
    }
    PathMatcher pathMatcher = configurer.getPathMatcher();
    if (pathMatcher != null) {
        mapping.setPathMatcher(pathMatcher);
    }
    Map<String, Predicate<Class<?>>> pathPrefixes = configurer.getPathPrefixes();
    if (pathPrefixes != null) {
        mapping.setPathPrefixes(pathPrefixes);
    }

    return mapping;
}
```

# HandlerAdapter

HttpRequestHandlerAdapter
RequestMappingHandlerAdapter
SimpleControllerHandlerAdapter
