# View
提供了MVC设计模式中的View抽象.

View 的实现可能有很大不同.
一个明显的实现是基于JSP的.其他实现可能是基于XSLT的, 或使用HTML生成库.
此接口旨在避免限制实现的范围.

```java
public interface View {
    defualt String getContentType() {
        return null;
    }

    void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throw Exception;
}
```

![](View.png)

# AbstractView

```java
public abstract class AbstractView extends WebApplicationObjectSupport implements View, BeanNameAware {
	/** Default content type. Overridable as bean property. */
	public static final String DEFAULT_CONTENT_TYPE = "text/html;charset=ISO-8859-1";

	/** Initial size for the temporary output byte array (if any). */
	private static final int OUTPUT_BYTE_ARRAY_INITIAL_SIZE = 4096;


	@Nullable
	private String contentType = DEFAULT_CONTENT_TYPE;

	@Nullable
	private String requestContextAttribute;

	private final Map<String, Object> staticAttributes = new LinkedHashMap<>();

	private boolean exposePathVariables = true;

	private boolean exposeContextBeansAsAttributes = false;

	@Nullable
	private Set<String> exposedContextBeanNames;

	@Nullable
	private String beanName;
}
```
