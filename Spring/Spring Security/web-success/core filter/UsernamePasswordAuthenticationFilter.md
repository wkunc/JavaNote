# AbstractAuthenticationProcessingFilter
负责登陆验证过程的 filter, 需要确保 authenticationManager 属性正确设置.
提供了 authentication (认证的基本过程):
1. 判断是否需要认证, 就是当前 request 访问的url 是否与 requiresAuthenticationRequestMatcher 匹配.
2. 如果匹配就进行调用 attemptAuthentication() 抽象方法执行验证.
3. 验证成功, 生成的Authentication对象放入当前线程的 SecuriteyContext 中.
4. 然后调用配置的 AuthenticationSucccessHandler, 在成功认证(登陆)后重定向到适当目标. (有一个默认实现的)
5. 

为子类实现提供方便.



```java
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean
		implements ApplicationEventPublisherAware, MessageSourceAware {

	protected ApplicationEventPublisher eventPublisher;
	protected AuthenticationDetailsSource<HttpServletRequest, ?> authenticationDetailsSource = new WebAuthenticationDetailsSource();
	private AuthenticationManager authenticationManager;
	protected MessageSourceAccessor messages = SpringSecurityMessageSource.getAccessor();
	private RememberMeServices rememberMeServices = new NullRememberMeServices();

	private RequestMatcher requiresAuthenticationRequestMatcher;

	private boolean continueChainBeforeSuccessfulAuthentication = false;

	private SessionAuthenticationStrategy sessionStrategy = new NullAuthenticatedSessionStrategy();

	private boolean allowSessionCreation = true;

	private AuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
	private AuthenticationFailureHandler failureHandler = new SimpleUrlAuthenticationFailureHandler();

	// ~ Constructors
	protected AbstractAuthenticationProcessingFilter(String defaultFilterProcessesUrl) {
		setFilterProcessesUrl(defaultFilterProcessesUrl);
	}

	protected AbstractAuthenticationProcessingFilter(
			RequestMatcher requiresAuthenticationRequestMatcher) {
		Assert.notNull(requiresAuthenticationRequestMatcher,
				"requiresAuthenticationRequestMatcher cannot be null");
		this.requiresAuthenticationRequestMatcher = requiresAuthenticationRequestMatcher;
	}

}
```


实现的Filter方法
```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
        throws IOException, ServletException {

    HttpServletRequest request = (HttpServletRequest) req;
    HttpServletResponse response = (HttpServletResponse) res;

    if (!requiresAuthentication(request, response)) {
        chain.doFilter(request, response);

        return;
    }

    if (logger.isDebugEnabled()) {
        logger.debug("Request is to process authentication");
    }

    Authentication authResult;

    try {
        authResult = attemptAuthentication(request, response);
        if (authResult == null) {
            // return immediately as subclass has indicated that it hasn't completed
            // authentication
            return;
        }
        sessionStrategy.onAuthentication(authResult, request, response);
    }
    catch (InternalAuthenticationServiceException failed) {
        logger.error(
                "An internal error occurred while trying to authenticate the user.",
                failed);
        unsuccessfulAuthentication(request, response, failed);

        return;
    }
    catch (AuthenticationException failed) {
        // Authentication failed
        unsuccessfulAuthentication(request, response, failed);

        return;
    }

    // Authentication success
    if (continueChainBeforeSuccessfulAuthentication) {
        chain.doFilter(request, response);
    }

    successfulAuthentication(request, response, chain, authResult);
}
```

# UsernamePasswordAuthenticationFilter

