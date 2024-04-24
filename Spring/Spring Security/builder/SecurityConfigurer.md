# SecurityConfigurer

用来配置对应的 SecurityBuilder 的配置类.
SecurityBuilder会在进行build之前用注册的配置类进行配置.

```java
// O 建造者要创建的对象类型
// B 负责构建O的 SecurityBuilder 实现类型.
public interface SecurityConfigurer<O, B extends SecurityBuilder<O>> {
	void init(B builder) throws Exception;

	void configure(B builder) throws Exception;
}
```

SecurityConfigurerAdapter 允许子类只实现部分方法.

# UserDetailsAwareConfigurer

```java
// 继承自 SercurityConfigurerAdapter, 填入的泛型为 AuthenticationManager, B.
// 代表这个配置类负责配置 SercurityBuilder<AuthenticationManager> 的构造器. 但是没有规定具体的Builder类型.

// B extends ProviderManagerBuilder<B> : 自己填入泛型要求必须是 ProviderManagerBuilder的子类.
// 还需要传入一个 UserDetailsService 的类型.
public abstract class UserDetailsAwareConfigurer<B extends ProviderManagerBuilder<B>, U extends UserDetailsService>
		extends SecurityConfigurerAdapter<AuthenticationManager, B> {

	/**
	 * Gets the {@link UserDetailsService} or null if it is not available
	 * @return the {@link UserDetailsService} or null if it is not available
	 */
	public abstract U getUserDetailsService();
}
```

# AbstractDaoAuthenticationConfigurer

Allows configuring a DaoAuthenticationProvider

// 暴露一些方法配置 DaoAuthenticationProvider, 主要是必须和 UserDetailsService 实例和一个可选的 PasswordEncoder.

```java
abstract class AbstractDaoAuthenticationConfigurer<B extends ProviderManagerBuilder<B>, C extends AbstractDaoAuthenticationConfigurer<B, C, U>, U extends UserDetailsService>
		extends UserDetailsAwareConfigurer<B, U> {

	private DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
	private final U userDetailsService;

	protected AbstractDaoAuthenticationConfigurer(U userDetailsService) {
		this.userDetailsService = userDetailsService;
		provider.setUserDetailsService(userDetailsService);
		if (userDetailsService instanceof UserDetailsPasswordService) {
			this.provider.setUserDetailsPasswordService((UserDetailsPasswordService) userDetailsService);
		}
	}

	public C withObjectPostProcessor(ObjectPostProcessor<?> objectPostProcessor) {
		addObjectPostProcessor(objectPostProcessor);
		return (C) this;
	}

	public C passwordEncoder(PasswordEncoder passwordEncoder) {
		provider.setPasswordEncoder(passwordEncoder);
		return (C) this;
	}

	public C userDetailsPasswordManager(UserDetailsPasswordService passwordManager) {
		provider.setUserDetailsPasswordService(passwordManager);
		return (C) this;
	}

    // 重点, 会给注册了这个配置类的 ProviderManagerBuilder 添加 DaoAuthenticationProvider.
	@Override
	public void configure(B builder) throws Exception {
		provider = postProcess(provider);
		builder.authenticationProvider(provider);
	}

	public U getUserDetailsService() {
		return userDetailsService;
	}
}
```
