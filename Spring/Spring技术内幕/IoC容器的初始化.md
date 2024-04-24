# IoC容器的初始化过程

BeanDefinition 的Resource定位, 载入,和注册.

Spring 把这三个过程分开, 并使用不同的模块来完成,
如使用 ResourceLoader, BeanDefinitionReader 等模块.
通过这样的设计方式, 可以让用户更加灵活地对三个过程进行剪裁或扩展,
定义出最适合自己的 IoC 容器的初始化过程

第一个过程是 Resource 定位过程. 这个Resource定位指的是BeanDefinition的资源
定位, 它由 ResourceLoader 通过统一的Resource接口完成.
这个Resource接口对各种形式的BeanDefinition的使用都提供了统一的接口

第二个过程是 BeanDefinition 的载入. 这个

第三个过程是向 IoC 容器注册这些 BeanDefinition 的过程.

## BeanDefinition的Resource定位

```java
ClassPathResource res = new ClassPathResource("bean.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(res);
```

以编程的方式使用*DefaultListableBeanFactory*时,
首先定义一个*Resource*来定位容器使用的*BeanDefinition*.
我们使用了ClassPathResource, 这意味着Spring会在类路径中去寻找以文件形式存在的BeanDefinition信息
ClassPathResource res = new ClassPathResource("beans.xml");

这里定义的 Resource 并不能由 DefaultListableFactory 直接使用,
Spring 通过 BeanDefinitionReader 来对这些信息进行处理.

这里也可以看出 Application 的好处,
在ApplicationContext中Spring为我们提供了一系列加载不同Resource的读取器实现.
而 DefaultListableBeanFactory 只是单纯的IoC容器实现, 需要为它配置特定的读取器才能完成这些功能.

回到我们经常使用的 ApplicationContext 上来, 例如:
FileSystemXmlApplicationContext, ClassPathXmlApplicationContext, XmlWebApplicationContext.
简单的从类名上就看以看出它们提供哪些不同的Resource读入功能.

下面我们分析常用的 ClassPathXmlApplicationContext 的实现, 看看它是怎样完成加载过程的.

从类图很明显看出 ApplicationContext 的基类是 DefaultResourceLoader(是一个 ResourceLoader的实现)
[](ClassPathXMLApplicationContext.PNG)
所以 ClassPathXmlApplicationContext 通过继承就拥有了 ResourcLoader 读入Resourc定义的BeanDefinition的能力

```java
public class FileSystemXmlApplicationContext extends AbstractXmlApplicationContext {
	public FileSystemXmlApplicationContext() { }
	public FileSystemXmlApplicationContext(ApplicationContext parent) {
		super(parent);
	}
    //这个构造器的参数是包含 BeanDefinition 所在文件的路径
	public FileSystemXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}
    //SE5出的变参, 允许传入多个文件路径
	public FileSystemXmlApplicationContext(String... configLocations) throws BeansException {
		this(configLocations, true, null);
	}
    // 允许传入多个文件路径的同时, 还允许指定自己的父IoC容器
	public FileSystemXmlApplicationContext(String[] configLocations, ApplicationContext parent) throws BeansException {
		this(configLocations, true, parent);
	}
	public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh) throws BeansException {
		this(configLocations, refresh, null);
	}
    // 在对象初始化过程中, 调用refresh()载入 BeanDefinition, 这个 refresh() 启动了
    // BeanDefinion 的载入过程
	public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {
		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
    /*
    * 这是应用于文件系统中 Resourc 的实现,通过构造一个 FileSystemResource来得到一个在文件系统中定位的BeanDefinion
    * 这个 getResourceByPath()是在 BeanDefinitionReader 的 loadBeanDefintion 中调用的
    * loadBeanDefintion 采用了模板设计模式, 具体的实现其实由子类负责
    */
	@Override
	protected Resource getResourceByPath(String path) {
		if (path != null && path.startsWith("/")) {
			path = path.substring(1);
		}
		return new FileSystemResource(path);
	}

}
```

----
AbstractRefreshableApplicationContext 对容器的初始化

```java
protected final void refreshBeanFactory() throws BeansException {
	// 如果, 已经有建立了 BeanFactory, 则销毁并关闭该 BeanFactory
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	// 创建并设置持有的 DefaultListableBeanFactory
	// loadBeanDefinitions() 再载入 BeanDefinition 的信息, (ps: loadBeanDefinitions() 是一个abstract方法,具体交给子类实现)
	try {
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		customizeBeanFactory(beanFactory);
		loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}
// 这里有时在Context中创建 DefaultListBeanFactory的地方, 而 getInternalParentBeanFactory()的具体实现可以
// 参看 AbstractApplicationContext 中实现, 会根据容器已有的 双亲IOC容器来生成 D
protected DefaultListableBeanFactory createBeanFactory() {
	return new DefaultListableBeanFactory(getInternalParentBeanFactory());
}

protected abstract void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException;
```

AbstractXmlApplicationContext 中实现了loadBeanDefinitions() 方法

```java
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	// 创建用于解析 xml 资源的解析器
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

	// 配置这个资源解析器, 第一是环境抽象, 第二是一个 ResourceLoader(它拥有加载资源的功能)
	beanDefinitionReader.setEnvironment(this.getEnvironment());
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	// 这个方法允许子类这个解析器提供自定义配置
	initBeanDefinitionReader(beanDefinitionReader);
	// 这个方法开始实际加载resource
	loadBeanDefinitions(beanDefinitionReader);
}
protected void initBeanDefinitionReader(XmlBeanDefinitionReader reader) {
	reader.setValidating(this.validating);
}
// 这个方法中有两个方法 getConfigResources() 和 getConfigLocations()
// getConfigResources() 就定义在下面, 但是 getConfigLoactions() 方法却在其父类 AbstractRefreshableConfigApplicationContext中定义
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
	Resource[] configResources = getConfigResources();
	if (configResources != null) {
		reader.loadBeanDefinitions(configResources);
	}
	String[] configLocations = getConfigLocations();
	if (configLocations != null) {
		reader.loadBeanDefinitions(configLocations);
	}
}
@Nullable
protected Resource[] getConfigResources() {
	return null;
}
@Nullable
protected String[] getConfigLocations() {
	return (this.configLocations != null ? this.configLocations : getDefaultConfigLocations());
}
```

# BeanDefinitionReader

AbstractBeanDefinitionReader中实现了基本过程, 所有的解析方法最终都会调用: 如下定义的方法, 而不同的类型的Resource解析有不同的子类实现.
我们下面用 XmlBeanDefinitionReader 作为示范

```java
	int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException;
```

Application只提供资源而不负责加载, 而且有两种情况.
reader.loadBeanDefinitions(configLoactions);

```java
// 遍历这个可变参数, 足以加载StringPath的Resource
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
	Assert.notNull(locations, "Location array must not be null");
	int count = 0;
	for (String location : locations) {
		count += loadBeanDefinitions(location);
	}
	return count;
}
public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(location, null);
}
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
		// 获取 ResourceLoader对象, 还记得在创建这个 XmlBeanDefinitionReader的时候我们对其进行了一系列的配置
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				// 用我们配置的 ResourcePatternResolver 进行加载资源
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				int count = loadBeanDefinitions(resources);
				if (actualResources != null) {
					Collections.addAll(actualResources, resources);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
				}
				return count;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
			int count = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
			}
			return count;
		}
	}
```

这里是xmlBeanDefintionReader中的具体实现

```java
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
	}
	/**
	 * Load bean definitions from the specified XML file.
	 * @param encodedResource the resource descriptor for the XML file,
	 * allowing to specify an encoding to use for parsing the file
	 * @return the number of bean definitions found
	 * @throws BeanDefinitionStoreException in case of loading or parsing errors
	 */
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
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
	protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			Document doc = doLoadDocument(inputSource, resource);
			int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
		}// 省略捕获SAX异常部分
	}
	// 创建一个 DocumentReader 解析这个 Document
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

下面是 BeanDefinitionDocumentReader 解析过程的部分

```java
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		doRegisterBeanDefinitions(doc.getDocumentElement());
	}
	/**
	 * Register each bean definition within the given root {@code <beans/>} element.
	 */
	@SuppressWarnings("deprecation")  // for Environment.acceptsProfiles(String...)
	protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				// We cannot use Profiles.of(...) since profile expressions are not supported
				// in XML config. See SPR-12458 for details.
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
	}
```

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		prepareRefresh();

		// 这里是在子类中启动 refreshBeanFactory() 的地方(ps: 就是上面分析的方法)
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		prepareBeanFactory(beanFactory);

		try {
			// 设置BeanFactory的后置处理
			postProcessBeanFactory(beanFactory);
			// 调用BeanFactory的后处理器, 这些后置处理器是在Bean定义中向容器注册的
			invokeBeanFactoryPostProcessors(beanFactory);
			// 注册Bean的后处理器, 在Bean的创建过程中调用
			registerBeanPostProcessors(beanFactory);
			// 对Context中的消息源进行初始化
			initMessageSource();
			// 初始化Context中的事件机制
			initApplicationEventMulticaster();
			// 初始化其他特殊的Bean
			onRefresh();
			// 检测ListernerBean并将这些bean向容器注册
			registerListeners();
			// 实例化所有的非(non-lazy-init)的Bean
			finishBeanFactoryInitialization(beanFactory);
			// 发布容器事件, 结束Refresh过程
			finishRefresh();
		}
		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```
