# SessionManagermentFilter
SessionManagementFilter根据SecurityContextHolder的当前内容
检查 SecurityContextRepository 的内容,
以确定用户是否已在当前请求期间进行了身份验证,
通常是通过非交互式身份验证机制进行的,
例如 "pre-authentication" 或 "remember-me".
如果存储库包含安全上下文,则过滤器不执行任何操作.
如果不是,并且线程本地SecurityContext包含（非匿名）身份验证对象,
则过滤器将假定它们已由堆栈中的先前过滤器进行了身份验证.
然后它将调用配置的SessionAuthenticationStrategy.

如果用户当前未通过身份验证,
则过滤器将检查是否已请求了无效的会话ID(例如,由于超时),
并且将调用已配置的InvalidSessionStrategy(如果已设置).
最常见的行为就是重定向到固定URL,
并将其封装在标准实现SimpleRedirectInvalidSessionStrategy中.
如前所述,在通过名称空间配置无效的会话URL时,也会使用后者.

# SessionAuthenticationStrategy




# SessionManagementConfigurer

Spring Security 的 Session 管理功能的配置类, 主要负责构建 SessionManagerFilter.
学习SessionManagerFilter可以知道, Session管理功能是插件化的, 
通过 SessionAuthenticationStrategy(session 认证策略)实现.

```java

public final class SessionManagementConfigurer<H extends HttpSecurityBuilder<H>>
		extends AbstractHttpConfigurer<SessionManagementConfigurer<H>, H> {

	private final SessionAuthenticationStrategy DEFAULT_SESSION_FIXATION_STRATEGY = createDefaultSessionFixationProtectionStrategy();

    // 默认的Session策略, 是Session固定攻击防范策略.
	private SessionAuthenticationStrategy sessionFixationAuthenticationStrategy = this.DEFAULT_SESSION_FIXATION_STRATEGY;

	private SessionAuthenticationStrategy sessionAuthenticationStrategy;

    // sessionAuthenticationStrategy()方法尽量不用这个
    //用来保存用户(框架使用者)提供自定义的 SessionAuthenticationStrategy.
	private SessionAuthenticationStrategy providedSessionAuthenticationStrategy;

    // addSessionAuthenticationStrategy() 方法使用, 添加额外的自定义Session策略
	private List<SessionAuthenticationStrategy> sessionAuthenticationStrategies = new ArrayList<>();


    // 默认的 InvalidSessionStrategy 是执行重定向, 所以需要一个指定一个url
	private InvalidSessionStrategy invalidSessionStrategy;
	private String invalidSessionUrl;

    // 默认的失败策略 
	private String sessionAuthenticationErrorUrl;
	private AuthenticationFailureHandler sessionAuthenticationFailureHandler;


    // 最大Session数, 用于并发Session控制, 如果设置了这个值不为null, 就会开启Session并发控制策略
	private Integer maximumSessions;
    // 超过最大session数时是否允许登陆
	private boolean maxSessionsPreventsLogin;
    // 实现session并发控制需要的session注册表
	private SessionRegistry sessionRegistry;

    // 在并发控制时才生效的
	private String expiredUrl;
	private SessionInformationExpiredStrategy expiredSessionStrategy;

    // 影响init(), 决定使用的 SecurityContextRepository 和 RequestCache 对象类型.
	private SessionCreationPolicy sessionPolicy;

    //
	private boolean enableSessionUrlRewriting;


	public SessionManagementConfigurer() {
	}
}
```
