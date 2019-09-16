# MultipartResolver

```java
public interface MultipartResolver {

	boolean isMultipart(HttpServletRequest request);

	MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException;

	void cleanupMultipart(MultipartHttpServletRequest request);
}
```
