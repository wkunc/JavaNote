# AbstractSecurityWebApplicationInitializer
这个类实现了 WebApplicationinitializer 接口
```java
public abstract class AbstractSecurityWebApplicationInitializer
		implements WebApplicationInitializer {

	private static final String SERVLET_CONTEXT_PREFIX = "org.springframework.web.servlet.FrameworkServlet.CONTEXT.";
	public static final String DEFAULT_FILTER_NAME = "springSecurityFilterChain";
	private final Class<?>[] configurationClasses;

	protected AbstractSecurityWebApplicationInitializer() {
		this.configurationClasses = null;
	}
	protected AbstractSecurityWebApplicationInitializer(
			Class<?>... configurationClasses) {
		this.configurationClasses = configurationClasses;
	}

    // 实现WebApplicationInitializer接口中的方法. 主要是注册了一个名叫
    // springSecurityFilterChain 的 DelegatingFilterProxy Filter对象.
    // 等效于我们用 web.xml 中配置
	public final void onStartup(ServletContext servletContext) throws ServletException {
        // 用来给子类扩展的空方法
		beforeSpringSecurityFilterChain(servletContext);
		if (this.configurationClasses != null) {
			AnnotationConfigWebApplicationContext rootAppContext = new AnnotationConfigWebApplicationContext();
			rootAppContext.register(this.configurationClasses);
			servletContext.addListener(new ContextLoaderListener(rootAppContext));
		}
		if (enableHttpSessionEventPublisher()) {
			servletContext.addListener(
					"org.springframework.security.web.session.HttpSessionEventPublisher");
		}
		servletContext.setSessionTrackingModes(getSessionTrackingModes());
        // 主要是这个方法的调用
		insertSpringSecurityFilterChain(servletContext);
        // 和 beforSpring... 方法一样是用来给我们的子类扩展用的
		afterSpringSecurityFilterChain(servletContext);
	}

	protected boolean enableHttpSessionEventPublisher() {
		return false;
	}

    // 根据类中定义的默认名字, 创建出一个 DelegatingFilterProxy (它将代理真正的Filter)
	private void insertSpringSecurityFilterChain(ServletContext servletContext) {
		String filterName = DEFAULT_FILTER_NAME;
		DelegatingFilterProxy springSecurityFilterChain = new DelegatingFilterProxy(
				filterName);
        // 这里的 contextAttribut 一定的是 null. 因为有一个空实现的方法
		String contextAttribute = getWebApplicationContextAttribute();
		if (contextAttribute != null) {
			springSecurityFilterChain.setContextAttribute(contextAttribute);
		}
        // 调用了这个类中的 private final 的方法, 这个方法就是一段向
        // ServletContext 注册 Filter 的模板代码.
		registerFilter(servletContext, true, filterName, springSecurityFilterChain);
	}

	protected final void insertFilters(ServletContext servletContext, Filter... filters) {
		registerFilters(servletContext, true, filters);
	}

	protected final void appendFilters(ServletContext servletContext, Filter... filters) {
		registerFilters(servletContext, false, filters);
	}

	private void registerFilters(ServletContext servletContext,
			boolean insertBeforeOtherFilters, Filter... filters) {
		Assert.notEmpty(filters, "filters cannot be null or empty");

		for (Filter filter : filters) {
			if (filter == null) {
				throw new IllegalArgumentException(
						"filters cannot contain null values. Got "
								+ Arrays.asList(filters));
			}
			String filterName = Conventions.getVariableName(filter);
			registerFilter(servletContext, insertBeforeOtherFilters, filterName, filter);
		}
	}

    // 这个类中唯一负责真正注册 Filter 的地方
    // 就是简单的调用 ServletContext.addFilter() 方法
	private final void registerFilter(ServletContext servletContext,
			boolean insertBeforeOtherFilters, String filterName, Filter filter) {
		Dynamic registration = servletContext.addFilter(filterName, filter);
		if (registration == null) {
			throw new IllegalStateException(
					"Duplicate Filter registration for '" + filterName
							+ "'. Check to ensure the Filter is only configured once.");
		}
		registration.setAsyncSupported(isAsyncSecuritySupported());
		EnumSet<DispatcherType> dispatcherTypes = getSecurityDispatcherTypes();
		registration.addMappingForUrlPatterns(dispatcherTypes, !insertBeforeOtherFilters,
				"/*");
	}

	private String getWebApplicationContextAttribute() {
		String dispatcherServletName = getDispatcherWebApplicationContextSuffix();
		if (dispatcherServletName == null) {
			return null;
		}
		return SERVLET_CONTEXT_PREFIX + dispatcherServletName;
	}

	protected Set<SessionTrackingMode> getSessionTrackingModes() {
		return EnumSet.of(SessionTrackingMode.COOKIE);
	}

	protected String getDispatcherWebApplicationContextSuffix() {
		return null;
	}

	protected void beforeSpringSecurityFilterChain(ServletContext servletContext) {

	}

	protected void afterSpringSecurityFilterChain(ServletContext servletContext) {

	}

	protected EnumSet<DispatcherType> getSecurityDispatcherTypes() {
		return EnumSet.of(DispatcherType.REQUEST, DispatcherType.ERROR,
				DispatcherType.ASYNC);
	}

	protected boolean isAsyncSecuritySupported() {
		return true;
	}
}
```
