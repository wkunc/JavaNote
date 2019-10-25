# HttpSecurity
负责构建 SecurityFilterChain 的Builder.
一个安全过滤器链由多个Filter组成.
所以拥有一个 List\<Filter> 来存储构成链的Filter.
和一个比较器保证过滤器的顺序正确.

这个 Builder 不会注册到IOC容器, 因为它是 webSecurity 的一部分构建内部的Filter链,
被WebSecurtiry显式的使用. 所以它的构造器才会是下面这个样子.

```java
public final class HttpSecurity extends
		AbstractConfiguredSecurityBuilder<DefaultSecurityFilterChain, HttpSecurity>
		implements SecurityBuilder<DefaultSecurityFilterChain>,
		HttpSecurityBuilder<HttpSecurity> {

	private final RequestMatcherConfigurer requestMatcherConfigurer;
	private List<Filter> filters = new ArrayList<>();
	private RequestMatcher requestMatcher = AnyRequestMatcher.INSTANCE;
	private FilterComparator comparator = new FilterComparator();

    // 唯一的一个构造器, 这个构造器暗示了这个 HttpSecurity 不会注册到IOC容器中.
    // 调用了父类构造器, 确定了 allowConfigurersOfSameType 是false, 意味着不能重复配置.
	@SuppressWarnings("unchecked")
	public HttpSecurity(ObjectPostProcessor<Object> objectPostProcessor,
			AuthenticationManagerBuilder authenticationBuilder,
			Map<Class<? extends Object>, Object> sharedObjects) {
		super(objectPostProcessor);
		Assert.notNull(authenticationBuilder, "authenticationBuilder cannot be null");

		setSharedObject(AuthenticationManagerBuilder.class, authenticationBuilder);
		for (Map.Entry<Class<? extends Object>, Object> entry : sharedObjects
				.entrySet()) {
			setSharedObject((Class<Object>) entry.getKey(), entry.getValue());
		}
		ApplicationContext context = (ApplicationContext) sharedObjects
				.get(ApplicationContext.class);
		this.requestMatcherConfigurer = new RequestMatcherConfigurer(context);
	}
}
```

实现 Builder 方法, 非常简单, 就是利用内部的 Filter 集合在排序(确保Filter的执行顺序是非常重要的)后调用构造器.
```java
@Override
protected DefaultSecurityFilterChain performBuild() throws Exception {
    Collections.sort(filters, comparator);
    return new DefaultSecurityFilterChain(requestMatcher, filters);
}
```

# 配置方法
SecurityFilterChain由两部分组成
1. RequsetMathcer: 判断是否在请求上生效 
2. List\<Filter>: 执行各项认为的Filter

## 配置Filter链的核心方法

实现 HttpSecurityBuilder 接口定义的方法.
(这个接口中的部分方法如: getConfigurer(), getShareObject() 其实以及在父类AbstractConfiguredSecurityBuilder 中实现了)
这些方法是Builder模式的基础.
建造者模式就是希望使用者可以根据自己的需要, 构建出不同状态的实例.
这个接口提供了3个, 直接影响内部的 List\<Filter> 的方法,
而 builde() 方法会依赖于这个集合, 所以就可以构建出不同的实例.
我们在使用时的方便配置方法, 也是通过这些添加 Filter 接口实现的.

```java
public <C> void setSharedObject(Class<C> sharedType, C object) {
    super.setSharedObject(sharedType, object);
}
// 基于构造时传入的 AuthenticationManagerBuilder 实现,
// 将原来AuthenticationManagerBuilder 的配置方法放到这个类中.
public HttpSecurity authenticationProvider(
        AuthenticationProvider authenticationProvider) {
    getAuthenticationRegistry().authenticationProvider(authenticationProvider);
    return this;
}
public HttpSecurity userDetailsService(UserDetailsService userDetailsService)
        throws Exception {
    getAuthenticationRegistry().userDetailsService(userDetailsService);
    return this;
}
private AuthenticationManagerBuilder getAuthenticationRegistry() {
    return getSharedObject(AuthenticationManagerBuilder.class);
}


// 主要是这三个方法, 这几个方法会在各种 Configerur 被用来添加特定的Filter使用.
public HttpSecurity addFilterAfter(Filter filter, Class<? extends Filter> afterFilter) {
    comparator.registerAfter(filter.getClass(), afterFilter);
    return addFilter(filter);
}

public HttpSecurity addFilterBefore(Filter filter,
        Class<? extends Filter> beforeFilter) {
    comparator.registerBefore(filter.getClass(), beforeFilter);
    return addFilter(filter);
}

public HttpSecurity addFilter(Filter filter) {
    Class<? extends Filter> filterClass = filter.getClass();
    if (!comparator.isRegistered(filterClass)) {
        throw new IllegalArgumentException(
                "The Filter class "
                        + filterClass.getName()
                        + " does not have a registered order and cannot be added without a specified order. Consider using addFilterBefore or addFilterAfter instead.");
    }
    this.filters.add(filter);
    return this;
}
```

方便配置方法, Spring Security 为不同的功能方面提供了大概 25 个左右的Filter.
这些 Filter 都是由不同的 Configurer 对象构建并配置到 HttpSecurity 的List\<Filter>
```java
// 和 AuthenticationBuilder 中的 apply() 方法类似, 只不过同一个配置, 只会添加一次.
// 因为 AuthenticationBuilder 中的配置是用来添加 AuthenticationProvider 实现的, 
// 允许有多个同类型的 AuthticationProvider, 因为即使都是 DaoAuthenticationPorvider 由于底层的数据存储不同, 
// 认证的返回也是不同的, 允许注册多个同类型的是为了构建更复杂的认证.

// 而这个不允许重复注册的原因是, 一个Filter做的事不需要同样做两遍.
private <C extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity>> C getOrApply(
        C configurer) throws Exception {
    C existingConfig = (C) getConfigurer(configurer.getClass());
    if (existingConfig != null) {
        return existingConfig;
    }
    return apply(configurer);
}
public <C> C getSharedObject(Class<C> sharedType) {
    return (C) this.sharedObjects.get(sharedType);
}
```

利用上面方法实现的方便注册Filter方法.
```java
public SessionManagementConfigurer<HttpSecurity> sessionManagement() throws Exception {
    return getOrApply(new SessionManagementConfigurer<>());
}

public RememberMeConfigurer<HttpSecurity> rememberMe() throws Exception {
    return getOrApply(new RememberMeConfigurer<>());
}

public SecurityContextConfigurer<HttpSecurity> securityContext() throws Exception {
    return getOrApply(new SecurityContextConfigurer<>());
}
```

## 配置RequestMatcher的方法
有两套, 一种简单是 requestMathcer() 简单的指定一下使用的RequestMatcher.(有三个基于这个方法的方便配置方法)
复杂的: requestMathcer() 通过一系列添加方法, 可以组合多个 RequestMatcher 实现(), 实现复杂的是否匹配逻辑.

```java
public RequestMatcherConfigurer requestMatchers() {
    return requestMatcherConfigurer;
}

public HttpSecurity requestMatcher(RequestMatcher requestMatcher) {
    this.requestMatcher = requestMatcher;
    return this;
}

// 下面三个方法是方便的配置方法. 就像 cors(), fromLogin() 等方法一样, 是通过别的方法实现的.
public HttpSecurity antMatcher(String antPattern) {
    return requestMatcher(new AntPathRequestMatcher(antPattern));
}

public HttpSecurity mvcMatcher(String mvcPattern) {
    HandlerMappingIntrospector introspector = new HandlerMappingIntrospector(getContext());
    return requestMatcher(new MvcRequestMatcher(introspector, mvcPattern));
}

public HttpSecurity regexMatcher(String pattern) {
    return requestMatcher(new RegexRequestMatcher(pattern, null));
}


// 两个为了完成高级的RequestMatcher配置, 写的两个内部类
public final class MvcMatchersRequestMatcherConfigurer extends RequestMatcherConfigurer {

    private MvcMatchersRequestMatcherConfigurer(ApplicationContext context,
            List<MvcRequestMatcher> matchers) {
        super(context);
        this.matchers = new ArrayList<>(matchers);
    }

    public RequestMatcherConfigurer servletPath(String servletPath) {
        for (RequestMatcher matcher : this.matchers) {
            ((MvcRequestMatcher) matcher).setServletPath(servletPath);
        }
        return this;
    }

}

public class RequestMatcherConfigurer
        extends AbstractRequestMatcherRegistry<RequestMatcherConfigurer> {

    protected List<RequestMatcher> matchers = new ArrayList<>();

    private RequestMatcherConfigurer(ApplicationContext context) {
        setApplicationContext(context);
    }

    @Override
    public MvcMatchersRequestMatcherConfigurer mvcMatchers(HttpMethod method,
            String... mvcPatterns) {
        List<MvcRequestMatcher> mvcMatchers = createMvcMatchers(method, mvcPatterns);
        setMatchers(mvcMatchers);
        return new MvcMatchersRequestMatcherConfigurer(getContext(), mvcMatchers);
    }

    @Override
    public MvcMatchersRequestMatcherConfigurer mvcMatchers(String... patterns) {
        return mvcMatchers(null, patterns);
    }

    @Override
    protected RequestMatcherConfigurer chainRequestMatchers(
            List<RequestMatcher> requestMatchers) {
        setMatchers(requestMatchers);
        return this;
    }

    private void setMatchers(List<? extends RequestMatcher> requestMatchers) {
        this.matchers.addAll(requestMatchers);
        requestMatcher(new OrRequestMatcher(this.matchers));
    }

    public HttpSecurity and() {
        return HttpSecurity.this;
    }
}
```


