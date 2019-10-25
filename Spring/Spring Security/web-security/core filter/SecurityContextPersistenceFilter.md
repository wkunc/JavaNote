# SecurityContextPersistenceFilter

根据应用程序的类型,可能需要制定一种策略来存储用户操作之间的安全上下文.
在典型的Web应用程序中,用户登录一次, 然后通过其会话ID进行标识.  服务器缓存持续时间会话的主体信息.  
在Spring Security中,
在请求之间存储SecurityContext的责任落在SecurityContextPersistenceFilter上,
它默认将上下文存储为HTTP请求之间的HttpSession属性.
它将每个请求的上下文还原到SecurityContextHolder,
并且至关重要的是,在请求完成时清除SecurityContextHolder.
为了安全起见,您不应直接与HttpSession进行交互.
这样做根本没有道理-始终使用SecurityContextHolder代替.

许多其他类型的应用程序(例如,无状态RESTful Web服务)不使用HTTP会话,
并且将在每个请求上重新进行身份验证. 
但是,将SecurityContextPersistenceFilter包含在链中以确保在每次请求后清除SecurityContextHolder仍然很重要.

> 自己的理解: 在没有使用安全框架时, 传统的Web应用下,
> 我会选择将用户信息存储在Session中, 鉴权(访问控制)功能会直接和Session交互,
> Session中获取信息, 如果没有获取到就是(未登陆), 获取到后判断权限.
> 而SpringSecurity设计的使用环境, 不完全是只考虑Web, 它还支持别的环境下的认证和访问控制.
> 所以SpringSecurity使用了设计了 SecurityContextHolder, 用来存储SecurityContext(凭证信息).
> 默认的方式是使用 ThreadLocal 保存每一个线程的 SecurityContext.
> 鉴权过程变成了和 SecurityContextHolder 交互获取认证身份信息, 判断有无权限.(就不依赖运行环境, 可以在任何环境中使用)
> 
> 然后再Web环境下, 用户进行登陆后, 后续请求就不再需要进行认证.
> (在没有使用安全框架前, 直接和HttpSession进行交互可以直接实现)
> 我们还是会再Session中保存认证信息, 但是为了框架的适应性, 它被已经设计为不予任何环境API(Session)交互.
> 所以我们需要在每次请求过程到访问控制系统(FilterSecurityInterceptor)前,
> 正确的从Session中获取认证信息, 重新填充 SecurityContextHolder.
> 使得如果一个用户经过了认证, 那么后续的请求就不再需要认证.
> 
> 然后由于SecurityContextHolder是与Thread相关的,
> 在Servlet下, 每次请求由一个Thread处理, Tomcat管理一个线程池.
> 线程是重复使用的, 当一个用户请求进入分配一个空闲线程处理, 响应完成后回收.
> 如果一个线程被分配给一个用户进行登陆逻辑后, 
> SecurityContextHodler 使用的ThreadLocal中就会保存这个线程和其当前用户登陆的认证信息.
> 如果我们在每次响应完成后不清除认证信息,
> 下一次这个 Thread, 被Servlet容器用来处理另一个用户的请求时, 
> SecurityContextHolder 就会有一个错误的认证信息(上一个没有被清理的用户信息)
> 从而导致错误. (明明是B用户的请求, 但是系统中的认证信息却是A用户).
> 所以在请求完成后, 清理 SecurityContextHolder 是必要的.

从上面的说明我们可以看出, 这个Filter的主要目的就是在同一个用户的请求之间传播SecurityContext,
而默认行为是将其存储在HttpSession中, 在每个请求进入时尝试获取Session, 并从中获取SecurityContext
如果能获取到就说明这个请求代表的用户是经过认证的.


```java
public class SecurityContextPersistenceFilter extends GenericFilterBean {

	static final String FILTER_APPLIED = "__spring_security_scpf_applied";

    // 主要核心, 一个策略接口有两个实现
	private SecurityContextRepository repo;

	private boolean forceEagerSessionCreation = false;

	public SecurityContextPersistenceFilter() {
		this(new HttpSessionSecurityContextRepository());
	}

	public SecurityContextPersistenceFilter(SecurityContextRepository repo) {
		this.repo = repo;
	}

	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		if (request.getAttribute(FILTER_APPLIED) != null) {
			// ensure that filter is only applied once per request
			chain.doFilter(request, response);
			return;
		}

		final boolean debug = logger.isDebugEnabled();

		request.setAttribute(FILTER_APPLIED, Boolean.TRUE);

		if (forceEagerSessionCreation) {
			HttpSession session = request.getSession();

			if (debug && session.isNew()) {
				logger.debug("Eagerly created session: " + session.getId());
			}
		}

		HttpRequestResponseHolder holder = new HttpRequestResponseHolder(request,
				response);
		SecurityContext contextBeforeChainExecution = repo.loadContext(holder);

		try {
			SecurityContextHolder.setContext(contextBeforeChainExecution);

			chain.doFilter(holder.getRequest(), holder.getResponse());

		}
		finally {
			SecurityContext contextAfterChainExecution = SecurityContextHolder
					.getContext();
			// Crucial removal of SecurityContextHolder contents - do this before anything
			// else.
			SecurityContextHolder.clearContext();
			repo.saveContext(contextAfterChainExecution, holder.getRequest(),
					holder.getResponse());
			request.removeAttribute(FILTER_APPLIED);

			if (debug) {
				logger.debug("SecurityContextHolder now cleared, as request processing completed");
			}
		}
	}

	public void setForceEagerSessionCreation(boolean forceEagerSessionCreation) {
		this.forceEagerSessionCreation = forceEagerSessionCreation;
	}
}
```

# SecurityContextRepository
```java
public interface SecurityContextRepository {

	SecurityContext loadContext(HttpRequestResponseHolder requestResponseHolder);

	void saveContext(SecurityContext context, HttpServletRequest request,
			HttpServletResponse response);

	boolean containsContext(HttpServletRequest request);
}
```

