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
以编程的方式使用*DefaultListableBeanFactory*时,
首先定义一个*Resource*来定位容器使用的*BeanDefinition*.
我们使用了ClassPathResource, 这意味着Spring会在类路径中去寻找以文件形式存在的BeanDefinition信息
ClassPathResource res = new ClassPathResource("beans.xml");

这里定义的 Resource 并不能由 DefaultListableFactory 直接使用,
Spring 通过 BeanDefinitionReader 来对这些信息进行处理.

这里也可以看出 Application 的好处,在ApplicationContext中Spring为我们提供了一系列加载不同Resource的读取器实现.
而 DefaultListableBeanFactory 只是单纯的IoC容器实现, 需要为它配置特定的读取器才能完成这些功能.


回到我们经常使用的 ApplicationContext 上来, 例如: FileSystemXmlApplicationContext, ClassPathXmlApplicationContext, XmlWebApplicationContext.
简单的从类名上就看以看出它们提供哪些不同的Resource读入功能.


下面我们分析常用的 ClassPathXmlApplicationContext 的实现, 看看它是怎样完成加载过程的.
从类图很明显看出 ApplicationContext 的基类是 DefaultResourceLoader(是一个 ResourceLoader的实现)
所以 ClassPathXmlApplicationContext 通过继承就拥有了 ResourcLoader 读入Resourc定义的BeanDefinition的能力
```java
public class FileSystemXmlApplicationContext extends AbstractXmlApplicationContext {
	/**
	 * Create a new FileSystemXmlApplicationContext for bean-style configuration.
     */
	public FileSystemXmlApplicationContext() {
	}
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
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
        customizeBeanFactory(beanFactory);
        // 启动对 BeanDefinition 的载入
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMoitor) {
            this.beanFactory = beanFactory;
        }
    } catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```
