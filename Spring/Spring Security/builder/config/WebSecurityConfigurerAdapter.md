# WebSecurityConfigurerAdapter

它是 Spring Security 提供的WebSecurityConfigurer实现类,
它虽然是抽象类但是它没有抽象方法.
Spring Security 提供的 WebSecurityConfiguration 类在调用 WebSecurity build()
方法前会保证其被正确的配置(在没有被配置的情况下使用 new WebSecurityConfiguration() {} 的方式进行填充配置).

```java
public abstract class WebSecurityConfigurerAdapter implements
		WebSecurityConfigurer<WebSecurity> {
	private final Log logger = LogFactory.getLog(WebSecurityConfigurerAdapter.class);

	private ApplicationContext context;

	private ContentNegotiationStrategy contentNegotiationStrategy = new HeaderContentNegotiationStrategy();

	private ObjectPostProcessor<Object> objectPostProcessor = new ObjectPostProcessor<Object>() {
		public <T> T postProcess(T object) {
			throw new IllegalStateException(
					ObjectPostProcessor.class.getName()
							+ " is a required bean. Ensure you have used @EnableWebSecurity and @Configuration");
		}
	};

    // 相当于一共有三个 AuthenticationManager, 主要使用 authenticationBuilder,
    // 其他两个用来作为其构造的 AuthenticationManager 的 parentAuthenticationManager.
    // 如果 config(auth) 方法被重写就使用 localConfigureAuthenticationBldr,
    // 否则使用 AuthenticationConfiguration
	private AuthenticationConfiguration authenticationConfiguration;
	private AuthenticationManagerBuilder authenticationBuilder;
	private AuthenticationManagerBuilder localConfigureAuthenticationBldr;

	private boolean disableLocalConfigureAuthenticationBldr;
	private boolean authenticationManagerInitialized;
	private AuthenticationManager authenticationManager;
	private AuthenticationTrustResolver trustResolver = new AuthenticationTrustResolverImpl();
	private HttpSecurity http;
	private boolean disableDefaults;

	protected WebSecurityConfigurerAdapter() {
		this(false);
	}

	protected WebSecurityConfigurerAdapter(boolean disableDefaults) {
		this.disableDefaults = disableDefaults;
	}
}
```

```java
protected final ApplicationContext getApplicationContext() {
    return this.context;
}

@Autowired
public void setApplicationContext(ApplicationContext context) {
    this.context = context;

    ObjectPostProcessor<Object> objectPostProcessor = context.getBean(ObjectPostProcessor.class);
    LazyPasswordEncoder passwordEncoder = new LazyPasswordEncoder(context);

    authenticationBuilder = new DefaultPasswordEncoderAuthenticationManagerBuilder(objectPostProcessor, passwordEncoder);
    localConfigureAuthenticationBldr = new DefaultPasswordEncoderAuthenticationManagerBuilder(objectPostProcessor, passwordEncoder) {
        @Override
        public AuthenticationManagerBuilder eraseCredentials(boolean eraseCredentials) {
            authenticationBuilder.eraseCredentials(eraseCredentials);
            return super.eraseCredentials(eraseCredentials);
        }

    };
}

@Autowired(required = false)
public void setTrustResolver(AuthenticationTrustResolver trustResolver) {
    this.trustResolver = trustResolver;
}

@Autowired(required = false)
public void setContentNegotationStrategy(
        ContentNegotiationStrategy contentNegotiationStrategy) {
    this.contentNegotiationStrategy = contentNegotiationStrategy;
}

/*
* 重点注入方法, 获取了 @EnableGolbalAuthentication 注解导入的 AuthenticationConfiguration.
* 不直接注入IOC容器中的 AuthenticationManagerBuilder 是有原因的.
* 使用 AuthenticationConfiguration 提供的方法可以更好的获取的 AuthenticationManager.
* 屏蔽了使用Builder.build() 方法的细节
*/
@Autowired
public void setObjectPostProcessor(ObjectPostProcessor<Object> objectPostProcessor) {
    this.objectPostProcessor = objectPostProcessor;
}

@Autowired
public void setAuthenticationConfiguration(
        AuthenticationConfiguration authenticationConfiguration) {
    this.authenticationConfiguration = authenticationConfiguration;
}
```

实现 WebSecurityConfigurer 方法

```java
// 重点方法, 调用 getHttp() 获取一个配置好的 HttpSecurity, 并注册到WebSecurity中.
public void init(final WebSecurity web) throws Exception {
    final HttpSecurity http = getHttp();
    web.addSecurityFilterChainBuilder(http).postBuildAction(new Runnable() {
        public void run() {
            FilterSecurityInterceptor securityInterceptor = http
                    .getSharedObject(FilterSecurityInterceptor.class);
            web.securityInterceptor(securityInterceptor);
        }
    });
}

// 就是给我们覆盖的配置方法.
public void configure(WebSecurity web) throws Exception {
}
```

重点方法, 我们知道 new 一个HttpSecurity必须要有一个 AuthenticationManager.

```java
protected final HttpSecurity getHttp() throws Exception {
    if (http != null) {
        return http;
    }

    DefaultAuthenticationEventPublisher eventPublisher = objectPostProcessor
            .postProcess(new DefaultAuthenticationEventPublisher());
    localConfigureAuthenticationBldr.authenticationEventPublisher(eventPublisher);

    // 重点, 确定 AuthenticationConfiguration 和 localConfigureAuthenticationBldr
    // 哪一个的构造的AuthenticationManager成为parent
    AuthenticationManager authenticationManager = authenticationManager();

    authenticationBuilder.parentAuthenticationManager(authenticationManager);
    authenticationBuilder.authenticationEventPublisher(eventPublisher);

    // 重点二, 在创建HttpSecurity时还需要一个 sharedObjectMap, 这个方法负责
    Map<Class<? extends Object>, Object> sharedObjects = createSharedObjects();

    http = new HttpSecurity(objectPostProcessor, authenticationBuilder,
            sharedObjects);
    if (!disableDefaults) {
        // @formatter:off
        http
            .csrf().and()
            .addFilter(new WebAsyncManagerIntegrationFilter())
            .exceptionHandling().and()
            .headers().and()
            .sessionManagement().and()
            .securityContext().and()
            .requestCache().and()
            .anonymous().and()
            .servletApi().and()
            .apply(new DefaultLoginPageConfigurer<>()).and()
            .logout();
        // @formatter:on
        ClassLoader classLoader = this.context.getClassLoader();
        List<AbstractHttpConfigurer> defaultHttpConfigurers =
                SpringFactoriesLoader.loadFactories(AbstractHttpConfigurer.class, classLoader);

        for (AbstractHttpConfigurer configurer : defaultHttpConfigurers) {
            http.apply(configurer);
        }
    }
    configure(http);
    return http;
}

// 重点1

// 重点2
private Map<Class<? extends Object>, Object> createSharedObjects() {
    Map<Class<? extends Object>, Object> sharedObjects = new HashMap<Class<? extends Object>, Object>();

    sharedObjects.putAll(localConfigureAuthenticationBldr.getSharedObjects());
    //
    sharedObjects.put(UserDetailsService.class, userDetailsService());

    sharedObjects.put(ApplicationContext.class, context);
    sharedObjects.put(ContentNegotiationStrategy.class, contentNegotiationStrategy);
    sharedObjects.put(AuthenticationTrustResolver.class, trustResolver);
    return sharedObjects;
}
```
