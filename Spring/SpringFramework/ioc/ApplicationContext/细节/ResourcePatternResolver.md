# ResoucePatternResolver

```java
public interface ResourcePatternResolver extends ResourceLoader {

	String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

	Resource[] getResources(String locationPattern) throws IOException;
}
```

# PathMatchingResourcePatternPatternResolver

# ServletContextResourcePatternResolver
