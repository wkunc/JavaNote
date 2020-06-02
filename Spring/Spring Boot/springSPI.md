# SpringFactoriesLoader
Java 提供了一个SPI机制 ServerLoader, 本质上就是用ClassLoader加载
META-INF/service 文件中指定的类.

SpringBoot 提供了一个类似的机制来实现自动配置功能.
SpringFactoriesLoader 类是这个机制的入口.
和Java 提供的ServerLoader一样, 这个SpringFactoriesLoader会加载
META-INF/spring.factories文件.


只公开了两个方法, 一个是 loadFactories(), loadFactoryNames().
很显然, loadFactories() 会用到 loadFactoryNames().
首先加载所有注册的类名, 然后为每个类进行实例化.
```java
/*
* 框架内部通用的目标 factory 加载机制.
* "META-INF/spring.factories"
* 采用 Properties 格式, key 是 interface or abstract class 的全限定名.
* value 是 用`,`分割的具体实现类列表
*/
public final class SpringFactoriesLoader {

	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";


	private static final Log logger = LogFactory.getLog(SpringFactoriesLoader.class);

    // 缓存
	private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();


	private SpringFactoriesLoader() {
	}

    // 用给定的ClassLoader加载指定的 factory 类所有注册者的实例.
	public static <T> List<T> loadFactories(Class<T> factoryClass, @Nullable ClassLoader classLoader) {
		Assert.notNull(factoryClass, "'factoryClass' must not be null");
		ClassLoader classLoaderToUse = classLoader;
		if (classLoaderToUse == null) {
			classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
		}

        // 使用另一个公开方法 loadFactoryNames , 获取所有需要实例化的全类名
		List<String> factoryNames = loadFactoryNames(factoryClass, classLoaderToUse);
		if (logger.isTraceEnabled()) {
			logger.trace("Loaded [" + factoryClass.getName() + "] names: " + factoryNames);
		}
		List<T> result = new ArrayList<>(factoryNames.size());

        // 为每个加载到的 全类名调用 private instantiateFactory() 方法进行实例化.
        // 逻辑很简单, 就是Class.forName()加载类, 然后class.getDeclaredConstructor()获取默认构造器.
        // 然后 constructor.newInstance();
        // 也就是说目标类必须有默认构造器(空构造器)
		for (String factoryName : factoryNames) {
			result.add(instantiateFactory(factoryName, factoryClass, classLoaderToUse));
		}
        // 利用支持@Order的比较器对结果进行排序.
		AnnotationAwareOrderComparator.sort(result);
		return result;
	}

    // 用给定的ClassLoader加载指定的 factory 类所有注册者的名字.
	public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
	}

    /*
    * 首先这个类有缓存, 也就说基本上只会加载一次.
    * 
    * 利用缓存
    * 返回的内容相当与所有的 META-INF/spring.factories 文件的内存表示
    */
	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		try {
            // 获取所有的 META-INF/spring.factories 文件路径.
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					String factoryClassName = ((String) entry.getKey()).trim();
					for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
						result.add(factoryClassName, factoryName.trim());
					}
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}

	@SuppressWarnings("unchecked")
	private static <T> T instantiateFactory(String instanceClassName, Class<T> factoryClass, ClassLoader classLoader) {
		try {
			Class<?> instanceClass = ClassUtils.forName(instanceClassName, classLoader);
			if (!factoryClass.isAssignableFrom(instanceClass)) {
				throw new IllegalArgumentException(
						"Class [" + instanceClassName + "] is not assignable to [" + factoryClass.getName() + "]");
			}
			return (T) ReflectionUtils.accessibleConstructor(instanceClass).newInstance();
		}
		catch (Throwable ex) {
			throw new IllegalArgumentException("Unable to instantiate factory class: " + factoryClass.getName(), ex);
		}
	}

}
```
