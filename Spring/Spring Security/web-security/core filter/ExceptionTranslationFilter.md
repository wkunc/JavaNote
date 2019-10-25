# ExceptionTranslationFilter
负责检测抛出的任何 Spring Security 异常.
此类异常通常是由 AbstractSecurityInterceptor(授权服务) 抛出.

ExceptionTranslationFilter, 返回错误代码 403(经过认证, 但是没有足够的权限)


```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
        throws IOException, ServletException {
    HttpServletRequest request = (HttpServletRequest) req;
    HttpServletResponse response = (HttpServletResponse) res;

    // 继续执行, 主要是catch块中的内容.
    // 保证可以捕获所有异常.
    try {
        chain.doFilter(request, response);

        logger.debug("Chain processed normally");
    }
    catch (IOException ex) {
        throw ex;
    }
    catch (Exception ex) {
        // 捕获Spring SecurityException.
        // Try to extract a SpringSecurityException from the stacktrace
        Throwable[] causeChain = throwableAnalyzer.determineCauseChain(ex);
        RuntimeException ase = (AuthenticationException) throwableAnalyzer
                .getFirstThrowableOfType(AuthenticationException.class, causeChain);

        if (ase == null) {
            ase = (AccessDeniedException) throwableAnalyzer.getFirstThrowableOfType(
                    AccessDeniedException.class, causeChain);
        }

        if (ase != null) {
            if (response.isCommitted()) {
                throw new ServletException("Unable to handle the Spring Security Exception because the response is already committed.", ex);
            }
            // 主要逻辑
            handleSpringSecurityException(request, response, chain, ase);
        }
        // 原样抛出其他异常
        else {
            // Rethrow ServletExceptions and RuntimeExceptions as-is
            if (ex instanceof ServletException) {
                throw (ServletException) ex;
            }
            else if (ex instanceof RuntimeException) {
                throw (RuntimeException) ex;
            }

            // Wrap other Exceptions. This shouldn't actually happen
            // as we've already covered all the possibilities for doFilter
            throw new RuntimeException(ex);
        }
    }
}


private void handleSpringSecurityException(HttpServletRequest request,
        HttpServletResponse response, FilterChain chain, RuntimeException exception)
        throws IOException, ServletException {
    if (exception instanceof AuthenticationException) {
        logger.debug(
                "Authentication exception occurred; redirecting to authentication entry point",
                exception);

        // 如果是没有认证异常, 调用 AuthenticationEntryPoint.
        sendStartAuthentication(request, response, chain,
                (AuthenticationException) exception);
    }
    else if (exception instanceof AccessDeniedException) {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

        // 虽然是访问拒绝异常, 但是不代表用户真正登录了. 可能是匿名用户或记住我用户.
        // 此时需要启用 AuthenticationEntryPoint
        aif (authenticationTrustResolver.isAnonymous(authentication) || authenticationTrustResolver.isRememberMe(authentication)) {
            logger.debug(
                    "Access is denied (user is " + (authenticationTrustResolver.isAnonymous(authentication) ? "anonymous" : "not fully authenticated") + "); redirecting to authentication entry point",
                    exception);

            sendStartAuthentication(
                    request,
                    response,
                    chain,
                    new InsufficientAuthenticationException(
                        messages.getMessage(
                            "ExceptionTranslationFilter.insufficientAuthentication",
                            "Full authentication is required to access this resource")));
        }
        else {
            logger.debug(
                    "Access is denied (user is not anonymous); delegating to AccessDeniedHandler",
                    exception);

            // 调用 accessDeniedHandler
            accessDeniedHandler.handle(request, response,
                    (AccessDeniedException) exception);
        }
    }
}
```


## AuthenticationEntryPoint

被 ExceptionTranslationFilter 使用, commence(启动) an authentication scheme
有多个实现, 分别用于启动不同的认证模式.
1. BasicAuthenticationEntryPoint
2. DigestAuthenticationEntryPoint
3. Http403ForbiddenEntryPoint
4. LoginUrlAuthenticationEntryPoint (基于表单的登陆会使用这个 entryPoint, 它是跳转的原因)
5. HttpStatusEntryPoint
6. DelegatingAuthenticationEntryPoint

```java
public interface AuthenticationEntryPoint {
	void commence(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException authException) throws IOException, ServletException;
}
```

## AccessDeniedHandler
如果用户登录了, 但是访问了一个没有权限的资源(权限不足).
会抛出 AccesssDenidedException 访问拒绝异常.
此时就会触发第二个策略 AccessDeniedHandler.

> 如果用户是匿名用户或记住我用户的话, 不会使用AccessDeniedHandler.
> 会和之前一样使用 AuthenticationEntryPoint, 导向认证点.
> 因为这两种用户不算严格意义上的登录

```java
public interface AccessDeniedHandler {
	void handle(HttpServletRequest request, HttpServletResponse response,
			AccessDeniedException accessDeniedException) throws IOException,
			ServletException;
}
```

# ExceptionHandingConfigurer
从默认配置看出, 默认情况下是使用 Http403ForbiddenEntryPoint(在抛出没有认证异常后, 发送403响应).
而在使用中我们发现在没有触发没有认证异常后, 会重定向到loginPage.
这是因为在启用FormLogin后, 其配置会影响使用的 AuthenticationEntryPoint.
使其变成 LoginUrlAuthenticationEntryPoint (重定向到指定Url).

* Shared Objects Created
nothing
* Shared Objects Used
1. RequestCache
2. AuthenticationEntryPoint

```java
public final class ExceptionHandlingConfigurer<H extends HttpSecurityBuilder<H>> extends
		AbstractHttpConfigurer<ExceptionHandlingConfigurer<H>, H> {

	private AuthenticationEntryPoint authenticationEntryPoint;

	private AccessDeniedHandler accessDeniedHandler;

	private LinkedHashMap<RequestMatcher, AuthenticationEntryPoint> defaultEntryPointMappings = new LinkedHashMap<>();

	private LinkedHashMap<RequestMatcher, AccessDeniedHandler> defaultDeniedHandlerMappings = new LinkedHashMap<>();

	public ExceptionHandlingConfigurer() {
	}

    // 指定使用的 authenticationEntryPoint (有合理的默认值基本可以进行配置)
    // 1. 如果没有调用这个方法进行指定, 就使用下面的这个 Map.
    // 2. 如果这个map为空(也没有进行配置, 就使用 Http403ForbiddenEntryPoint)
    // 3. 如果这个map只有一个, 那么就使用那一个
    // 4. 如果提供了多个, 就使用 DelegatingAuthenticationEntryPoint 将多个实例进行包装.
    // 内部逻辑是, 如果 requset 匹配 RequestMatcher, 然后调用对应的 AuthenticationEntryPoint.
	public ExceptionHandlingConfigurer<H> authenticationEntryPoint(
			AuthenticationEntryPoint authenticationEntryPoint) {
		this.authenticationEntryPoint = authenticationEntryPoint;
		return this;
	}
	public ExceptionHandlingConfigurer<H> defaultAuthenticationEntryPointFor(
			AuthenticationEntryPoint entryPoint, RequestMatcher preferredMatcher) {
		this.defaultEntryPointMappings.put(preferredMatcher, entryPoint);
		return this;
	}

    //
	public ExceptionHandlingConfigurer<H> accessDeniedPage(String accessDeniedUrl) {
		AccessDeniedHandlerImpl accessDeniedHandler = new AccessDeniedHandlerImpl();
		accessDeniedHandler.setErrorPage(accessDeniedUrl);
		return accessDeniedHandler(accessDeniedHandler);
	}
	public ExceptionHandlingConfigurer<H> accessDeniedHandler(
			AccessDeniedHandler accessDeniedHandler) {
		this.accessDeniedHandler = accessDeniedHandler;
		return this;
	}
	public ExceptionHandlingConfigurer<H> defaultAccessDeniedHandlerFor(
			AccessDeniedHandler deniedHandler, RequestMatcher preferredMatcher) {
		this.defaultDeniedHandlerMappings.put(preferredMatcher, deniedHandler);
		return this;
	}

}
```
