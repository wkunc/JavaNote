# BeanDefinitionReader
![](imgs/BeanDefinitionReader.PNG)

Simple interface for bean definition readers.
这个接口负责向 BeanDefinitionRegistry 中注册 BeanDefinition
```java
public interface BeanDefinitionReader {
    //这几个方法其实反应了实现类中会有什么
    //BeanDefinitionReader 其实就是组合运用这几个类
    BeanDefinitionRegistry getRegistry();
    ResourceLoader getResourceLoader();
    ClassLoader getBeanClassLoader();
    BeanNameGenerator getBeanNameGenerator();
    // 核心方法
    int loadBeanDefinitions(Resource resource);
    int loadBeanDefinitions(Resource... resources);
    int loadBeanDefinitions(String location);
    int loadBeanDefinitions(String... locations);
}
```

## AbstractBeanDefinitionReader
实现了 BeanDefinitionReader 接口. 有一个方法没有实现
```java
public abstract class AbstractBeanDefinitionReader implements EnvironmentCapable, BeanDefinitionReader{
    private final BeanDefinitionRegistry registry;
    private ResourceLoader resourceLoader;
    private ClassLoader beanClassLoader;
    private Environment environment;
    private BeanNameGenerator beanNameGenerator = new DefaultBeanNameGenerator();
    
    /*
    * 利用BeanDefinitionRegistry进行初始化.
    * 因为在Spring中Application实现了各种接口,
    * 在Spring内部中每次调用这个方法传入的其实都是 BeanFactory 或 Application
    * */
	protected AbstractBeanDefinitionReader(BeanDefinitionRegistry registry) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		this.registry = registry;

		// Determine ResourceLoader to use.
		if (this.registry instanceof ResourceLoader) {
			this.resourceLoader = (ResourceLoader) this.registry;
		}
		else {
			this.resourceLoader = new PathMatchingResourcePatternResolver();
		}
		// Inherit Environment if possible
		if (this.registry instanceof EnvironmentCapable) {
			this.environment = ((EnvironmentCapable) this.registry).getEnvironment();
		}
		else {
			this.environment = new StandardEnvironment();
		}
	}
    
    public final BeanDefinitionRegistry getBeanFactory(){
        return this.registry;
    }

    public final BeanDefinitionRegistry getRegistry(){
        return this.registry;
    }

    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }

    
    public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionException {
        Assert.notNull(resource, "Resource array must not be null");
        int counter = 0;
        for (Resource resource : resources) {
            counter += loadBeanDefinitions(resource);
        }
        return counter;
    }

    public int loadBeanDefinitions(String location) throws BeanDefinitionException {
        return loadBeanDefinitions(location, null);
    }
    public int loadBeanDefinitions(String... locations) throws BeanDefinitionException {
        Assert.notNull(locations, "Resource array must not be null");
        int counter = 0;
        for (String location : locations) {
            counter += loadBeanDefinitions(location);
        }
        return counter;
    }

    public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionException {
        ResourceLoader resourceLoader = getResourceLoader();
        if (resourceLoader = null) {
            throw new BeanDefinitionStoreException(
                    "Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
        }

        if (resourceLoader instanceof ResourcePatternResolver) {
            try {
                Resource[] resources = ((ResourcePatternResolver)resourceLoader).getResources(location);
                int loadCount = loadBeanDefinitions(resources);
                if (actualResources != null) {
                    for (Resource resource : resources) {
                        actualResources.add(resource);
                    }
                }
                if (logger.isDebugEnable()) {
                    logger.debug();
                }
                return loadCount;
            }
            catch (IOException ex) {
                throw new BeanDefinitionStoreException(
                        "Could not resolve bean definition resource pattern [" + location + "]");
            }
        }
        else {
            Resource resource = resourceLoader.getResource(location);
            int loadCount = loadBeanDefinitions(resource);
            if (actualResources != null) {
                actualResources.add(resource);
            }
            if (logger.isDebugEnable()) {
                logger.debug();
            }
            return loadCount;
        }
    }
}
```
```java
public loadBeanDefinitions(Resource resource);
```
其他load方法底层都依赖于它, 然后把它交给子类实现 (ps:模板方法设计模式)

其他方法的实现都很简单, 只是简单的持有一个相应的类对象
然后一些 setter/getter 方法

# XmlBeanDefinitionReader
字段含义
```java
public static final int VALIDATION_NOTE = XmlValidationModeDetector.VALIDATION_NOTE;
public static final int VALIDATION_AUTO = XmlValidationModeDetector.VALIDATION_AUTO;
public static final int VALIDATION_DTD = XmlValidationModeDetector.VALIDATION_DTD;
public static final int VALIDATION_XSD = XmlValidationModeDetector.VALIDATION_XSD;
private static final Constants constants = new Constants(XmlBeanDefinitionReader.class);
private int validationMode = VALIDATION_AUTO;
private boolean namespaceAware = false;
private Class<?> documentReaderClass = DefaultBeanDefinitionDocumentReader.class;
private ProblemReporter problemReporter = new FailFastProblemReporter();
private ReaderEventListener eventListener = new EmptyReaderEventListener();
private SourceExtractor sourceExtractor = new NullSourceExtractor();
private NamespaceHandlerResolver namespaceHandlerResolver;
private DocumentLoader documentLoader = new DefaultDocumentLoader();
private EntityResolver entityResolver;
private ErrorHandler errorHandler = new SimpleSaxErrorHandler();
private final XmlValidationModeDetector validationModeDetector = new XmlValidationModeDetector();
private final ThreadLocal<Set<EncodedResource>> resourcesCurrentlyBeingLoaded = 
        new NamedThreadLocal<>("XML bean definition resources currently being loaded");
```
XmlBeanDefinitionReader 实现了这个抽象方法
```java
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}

        // resourceCurrentlyBeingLoaded 是一个 ThreadLocal 对象
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<EncodedResource>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
                // 加载 BeanDefinitions
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
```
