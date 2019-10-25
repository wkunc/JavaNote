# AuthenticationManagerBuilder
负责构建 AuthenticationManager, 相对独立.
因为不论是 web 环境下的认证, 还是非web环境下的认证, 都需要AuthenticationManager.
所以Spring Security 对它的配置类也是相当独立的由 @EnableGlobalAuthentication 注解添加.

无论是web-security还是method-security的注解内部都有这个@EnableGlobalAuthentication 注解.


通过之前对AuthenticManager接口及其Spring Security提供的默认实现 ProviderManager 的学习.
我们知道 ProviderManager 会将执行过程委托给内部 support 当前 token 类型的 AuthicationProvider 对象.
并且在自己无法支持认证的情况下, 会试着委托给自己的 parent (如果有的话).
所以需要一个 AuthenticationProvider 列表, 或者一个 parentAuthenticationManager. 

这两个东西必须拥有至少一个才能构建 ProviderManager 否则报错 *参考 isConfigured()方法*

```java
public class AuthenticationManagerBuilder
		extends
		AbstractConfiguredSecurityBuilder<AuthenticationManager, AuthenticationManagerBuilder>
		implements ProviderManagerBuilder<AuthenticationManagerBuilder> {
	private final Log logger = LogFactory.getLog(getClass());

    // 构建的认证管理器的父亲.
	private AuthenticationManager parentAuthenticationManager;

    // Spring 提供的 AuthenticationManager 的实现是基于 AuthenticationProvider 的, 所以需要一个List来保存.
	// 采用静态初始化的一个原因是, ProviderManager 的构造器要求传入的list, 不能是null, 不过可以是empty
	private List<AuthenticationProvider> authenticationProviders = new ArrayList<>();

    //
	private UserDetailsService defaultUserDetailsService;

    // 是否需要抹除密码, 用来设置 ProviderManager 的行为.
	private Boolean eraseCredentials;

    // SpringSecurity 事件发送器.
	private AuthenticationEventPublisher eventPublisher;

    // 调用父类构造器, 设置可以重复配置.
	public AuthenticationManagerBuilder(ObjectPostProcessor<Object> objectPostProcessor) {
		super(objectPostProcessor, true);
	}
}
```

```java
// 实现 ProviderManagerBuilder 接口, 主要的配置接口, 
// 通过这个方法为将要构建的ProviderManager提供接口实现.
// 对应的负责配置这个Builder的 SecurityConfiguerer 也是利用这个接口向其设置实现
public AuthenticationManagerBuilder authenticationProvider(
        AuthenticationProvider authenticationProvider) {
    this.authenticationProviders.add(authenticationProvider);
    return this;
}

// 次要的配置接口, 就是设置其他三个成员变量的值罢了.
public AuthenticationManagerBuilder parentAuthenticationManager(
        AuthenticationManager authenticationManager) {
    if (authenticationManager instanceof ProviderManager) {
        eraseCredentials(((ProviderManager) authenticationManager)
                .isEraseCredentialsAfterAuthentication());
    }
    this.parentAuthenticationManager = authenticationManager;
    return this;
}
public AuthenticationManagerBuilder authenticationEventPublisher(
        AuthenticationEventPublisher eventPublisher) {
    Assert.notNull(eventPublisher, "AuthenticationEventPublisher cannot be null");
    this.eventPublisher = eventPublisher;
    return this;
}
public AuthenticationManagerBuilder eraseCredentials(boolean eraseCredentials) {
    this.eraseCredentials = eraseCredentials;
    return this;
}

// 实现build逻辑.
@Override
protected ProviderManager performBuild() throws Exception {
    // 判断是否有 authenticationProvider 实例或者 parentAuthenticationManager,
    // 没有这些东西的话, 是不能创建出能使用的 ProviderManager 实例的
    if (!isConfigured()) {
        logger.debug("No authenticationProviders and no parentAuthenticationManager defined. Returning null.");
        return null;
    }
    // 调用构造器, 并设置. 
    ProviderManager providerManager = new ProviderManager(authenticationProviders,
            parentAuthenticationManager);
    if (eraseCredentials != null) {
        providerManager.setEraseCredentialsAfterAuthentication(eraseCredentials);
    }
    if (eventPublisher != null) {
        providerManager.setAuthenticationEventPublisher(eventPublisher);
    }

    // 让 ObjectProcessor 来处理生成的对象. Spring提供了一个 AutowiredObjectProcessor.
    // 会为传入的Object触发生命周期回调(init 方法, Aware 接口注入)
    // 因为Spring-Security 的配置文件是只注册各种组件 Builder.
    // 通过这些Builder获取定制完成的组件. 如: AuthenticationManager, AccesionManager.
    // 用这些组件, 去构建Filter链.
    // 所以这个 Builder.build() 创建的Bean, 不会注册到Spring IOC 容器中.
    // 也就享受不到容器提供的生命周期, 以及自动配置.
    // 为了这些组件也可以有生命周期, 提供了一个 ObjectProcessor 来触发这个事情.
    providerManager = postProcess(providerManager);

    return providerManager;
}
```

特殊方法
```java
// 本质上就是和父类一样的思路, 添加一个 SecurityConfiguerer 对象, 到集合中.
// 不过要求这个 Configurer 对象是特定子接口的实现.
// 从传入的 Conigurer 对象中获取 UserDetailsService 
private <C extends UserDetailsAwareConfigurer<AuthenticationManagerBuilder, ? extends UserDetailsService>> C apply(
        C configurer) throws Exception {
    this.defaultUserDetailsService = configurer.getUserDetailsService();
    return (C) super.apply(configurer);
}

// 基于上面方法实现的配置方法, 我们在安全配置类中使用的方法就是这里.
// 除了LDAP auth是不同的以外,
// 其他三个都是 AbstractDaoAutheicationConfiguer 的子类, 
// 负责创建一个 DaoAuthenticationProvider. 
// 根据对 AuthenticationManager 接口的学习, 
// 我们知道 DaoAuthenticationProvider 是基于 UserDetailsService 实现的支持 UsernamePasswordToken 类型认证请求的提供者
// 所以这三个方法不同就是配置 UserDetailsService 方式.
// 如果想学习源码的话, 就看看 userDetailsService() 方法使用的 SecurityConfigurer 实现吧.
public InMemoryUserDetailsManagerConfigurer<AuthenticationManagerBuilder> inMemoryAuthentication()
        throws Exception {
    return apply(new InMemoryUserDetailsManagerConfigurer<>());
}
public JdbcUserDetailsManagerConfigurer<AuthenticationManagerBuilder> jdbcAuthentication()
        throws Exception {
    return apply(new JdbcUserDetailsManagerConfigurer<>());
}
public <T extends UserDetailsService> DaoAuthenticationConfigurer<AuthenticationManagerBuilder, T> userDetailsService(
        T userDetailsService) throws Exception {
    this.defaultUserDetailsService = userDetailsService;
    return apply(new DaoAuthenticationConfigurer<>(
            userDetailsService));
}
public LdapAuthenticationProviderConfigurer<AuthenticationManagerBuilder> ldapAuthentication()
        throws Exception {
    return apply(new LdapAuthenticationProviderConfigurer<>());
}
```


# @EnableGlobalAuthentication
Spring Security 提供了一个@EnableGlobalAuthentication 注解,
用来配置全局的 AuthenticationManagerBuilder.
会自己从IOC中获取 AuthenticationProvider 的实现, 
或者是 UserDetailsService 的Bean配置为DaoAuthenticationProvider.
也可以是显式的利用 @Autowired 获取配置的 AuthenticationManagerBuilder 实例, 调用上面的方法进行配置.
> 注意: 上面说的三种方式是互斥的, 只会有一种生效, 
> 比如说IOC中有一个 DaoAuthenticationProvider Bean, 我们也提供了一个UserDetailsService. 
> 通过Spring的Order机制确定 securityConfig 类生效的顺序,
> order 值越小优先级越高(排序算法通常是从小到大排序的, 所以越小越在前面)
> 设置是查找使用 AuthenticationProvider 实例, 然后才检测 UserDetailsService, 当然显式的配置是最大的.

@EnableWebSecurity, @EnableGlobalMethodSecurity 注解中都有它.

```java
@Configuration
@Import(ObjectPostProcessorConfiguration.class) // 导入一个调用生命周期方法的 ObjectProcessor 在SecurityBuilder笔记中提到过.
public class AuthenticationConfiguration {

	private AtomicBoolean buildingAuthenticationManager = new AtomicBoolean();

	private ApplicationContext applicationContext;

	private AuthenticationManager authenticationManager;

	private boolean authenticationManagerInitialized;

	private List<GlobalAuthenticationConfigurerAdapter> globalAuthConfigurers = Collections
			.emptyList();

	private ObjectPostProcessor<Object> objectPostProcessor;

	@Bean
	public AuthenticationManagerBuilder authenticationManagerBuilder(
			ObjectPostProcessor<Object> objectPostProcessor, ApplicationContext context) {
		LazyPasswordEncoder defaultPasswordEncoder = new LazyPasswordEncoder(context);
		AuthenticationEventPublisher authenticationEventPublisher = getBeanOrNull(context, AuthenticationEventPublisher.class);

		DefaultPasswordEncoderAuthenticationManagerBuilder result = new DefaultPasswordEncoderAuthenticationManagerBuilder(objectPostProcessor, defaultPasswordEncoder);
		if (authenticationEventPublisher != null) {
			result.authenticationEventPublisher(authenticationEventPublisher);
		}
		return result;
	}

	@Bean
	public static GlobalAuthenticationConfigurerAdapter enableGlobalAuthenticationAutowiredConfigurer(
			ApplicationContext context) {
		return new EnableGlobalAuthenticationAutowiredConfigurer(context);
	}

	@Bean
	public static InitializeUserDetailsBeanManagerConfigurer initializeUserDetailsBeanManagerConfigurer(ApplicationContext context) {
		return new InitializeUserDetailsBeanManagerConfigurer(context);
	}

	@Bean
	public static InitializeAuthenticationProviderBeanManagerConfigurer initializeAuthenticationProviderBeanManagerConfigurer(ApplicationContext context) {
		return new InitializeAuthenticationProviderBeanManagerConfigurer(context);
	}

	/*
	* 希望给外界调用的方法接口
	*/
	public AuthenticationManager getAuthenticationManager() throws Exception {
		if (this.authenticationManagerInitialized) {
			return this.authenticationManager;
		}
		AuthenticationManagerBuilder authBuilder = authenticationManagerBuilder(
				this.objectPostProcessor, this.applicationContext);
		if (this.buildingAuthenticationManager.getAndSet(true)) {
			return new AuthenticationManagerDelegator(authBuilder);
		}

		for (GlobalAuthenticationConfigurerAdapter config : globalAuthConfigurers) {
			authBuilder.apply(config);
		}

		authenticationManager = authBuilder.build();

		if (authenticationManager == null) {
			authenticationManager = getAuthenticationManagerBean();
		}

		this.authenticationManagerInitialized = true;
		return authenticationManager;
	}

	/*
	* 获取IOC容器中注册的 GlobalAuthenticationConfigurerAdapter 类实现.
	* 通常情况下就是获取这个配置类中配置的3个Bean
	* 1. enableGlobalAuthenticationAutowiredConfigurer
	* 2. initializeUserDetailsBeanManagerConfigurer
	* 3. initializeAuthenticationProviderBeanManagerConfigurer
	* 并使用 Order 语义排序, 导致 3 比 2 早生效
	*/
	@Autowired(required = false)
	public void setGlobalAuthenticationConfigurers(
			List<GlobalAuthenticationConfigurerAdapter> configurers) throws Exception {
		Collections.sort(configurers, AnnotationAwareOrderComparator.INSTANCE);
		this.globalAuthConfigurers = configurers;
	}

	@Autowired
	public void setApplicationContext(ApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
	}

	@Autowired
	public void setObjectPostProcessor(ObjectPostProcessor<Object> objectPostProcessor) {
		this.objectPostProcessor = objectPostProcessor;
	}

	@SuppressWarnings("unchecked")
	private <T> T lazyBean(Class<T> interfaceName) {
		LazyInitTargetSource lazyTargetSource = new LazyInitTargetSource();
		String[] beanNamesForType = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
				applicationContext, interfaceName);
		if (beanNamesForType.length == 0) {
			return null;
		}
		Assert.isTrue(beanNamesForType.length == 1,
				() -> "Expecting to only find a single bean for type " + interfaceName
						+ ", but found " + Arrays.asList(beanNamesForType));
		lazyTargetSource.setTargetBeanName(beanNamesForType[0]);
		lazyTargetSource.setBeanFactory(applicationContext);
		ProxyFactoryBean proxyFactory = new ProxyFactoryBean();
		proxyFactory = objectPostProcessor.postProcess(proxyFactory);
		proxyFactory.setTargetSource(lazyTargetSource);
		return (T) proxyFactory.getObject();
	}

	private AuthenticationManager getAuthenticationManagerBean() {
		return lazyBean(AuthenticationManager.class);
	}

	private static <T> T getBeanOrNull(ApplicationContext applicationContext, Class<T> type) {
		try {
			return applicationContext.getBean(type);
		} catch(NoSuchBeanDefinitionException notFound) {
			return null;
		}
	}

	private static class EnableGlobalAuthenticationAutowiredConfigurer extends
			GlobalAuthenticationConfigurerAdapter {
		private final ApplicationContext context;
		private static final Log logger = LogFactory
				.getLog(EnableGlobalAuthenticationAutowiredConfigurer.class);

		public EnableGlobalAuthenticationAutowiredConfigurer(ApplicationContext context) {
			this.context = context;
		}

		@Override
		public void init(AuthenticationManagerBuilder auth) {
			Map<String, Object> beansWithAnnotation = context
					.getBeansWithAnnotation(EnableGlobalAuthentication.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly initializing " + beansWithAnnotation);
			}
		}
	}

	/*
	* 下面的几个静态内部类比较简单, 为了节省篇幅就不完全显示了.
	*/
	static final class AuthenticationManagerDelegator implements AuthenticationManager {
		//....
	}

	static class DefaultPasswordEncoderAuthenticationManagerBuilder extends AuthenticationManagerBuilder {
		private PasswordEncoder defaultPasswordEncoder;
		//....
	}

	static class LazyPasswordEncoder implements PasswordEncoder {
		//.....
	}
}
```
