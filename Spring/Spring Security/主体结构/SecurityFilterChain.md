# SecuritryFilterChain


```java
public interface SecurityFilterChain {

	boolean matches(HttpServletRequest request);

	List<Filter> getFilters();
}
```



```java
public final class DefaultSecurityFilterChain implements SecurityFilterChain {
	private static final Log logger = LogFactory.getLog(DefaultSecurityFilterChain.class);
	private final RequestMatcher requestMatcher;
	private final List<Filter> filters;

	public DefaultSecurityFilterChain(RequestMatcher requestMatcher, Filter... filters) {
		this(requestMatcher, Arrays.asList(filters));
	}

	public DefaultSecurityFilterChain(RequestMatcher requestMatcher, List<Filter> filters) {
		logger.info("Creating filter chain: " + requestMatcher + ", " + filters);
		this.requestMatcher = requestMatcher;
		this.filters = new ArrayList<>(filters);
	}

	public RequestMatcher getRequestMatcher() {
		return requestMatcher;
	}

	public List<Filter> getFilters() {
		return filters;
	}

	public boolean matches(HttpServletRequest request) {
		return requestMatcher.matches(request);
	}

	@Override
	public String toString() {
		return "[ " + requestMatcher + ", " + filters + "]";
	}
}
```

