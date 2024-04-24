# GenericApplicationContext

通用ApplicationContext实现, 它包含单个内部 DefaultListableBeanFactory 实例.
并且不假定特定的 Bean 定义格式.(就是配置文件的形式没有约束)

实现BeanDefinitionRegistry接口, 以允许将任何 BeanDfinitionReader 应用于它.

典型用法是通过 BeanDefinitionRegistry 接口注册各种bean定义,
然后调用 refresh() 以使用applicationContext语义初始化这些bean.

与每次刷新创建新的内部BeanFactory实例的其他ApplicationContext相比,
此上下文内部的 BeanFactory 从一开始就是可用的, 以便能够在其上注册bean定义.
refresh() 方法只能调用一次.

典型用法如下:

```java
   GenericApplicationContext ctx = new GenericApplicationContext();

   XmlBeanDefinitionReader xmlReader = new XmlBeanDefinitionReader(ctx);
   xmlReader.loadBeanDefinitions(new ClassPathResource("applicationContext.xml"));

   PropertiesBeanDefinitionReader propReader = new PropertiesBeanDefinitionReader(ctx);
   propReader.loadBeanDefinitions(new ClassPathResource("otherBeans.properties"));

   ctx.refresh();

   MyBean myBean = (MyBean) ctx.getBean("myBean");
```

# 实现

```java
public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {
	private final DefaultListableBeanFactory beanFactory;

	@Nullable
	private ResourceLoader resourceLoader;

	private boolean customClassLoader = false;

	private final AtomicBoolean refreshed = new AtomicBoolean();

	public GenericApplicationContext() {
		this.beanFactory = new DefaultListableBeanFactory();
	}

	public GenericApplicationContext(DefaultListableBeanFactory beanFactory) {
		Assert.notNull(beanFactory, "BeanFactory must not be null");
		this.beanFactory = beanFactory;
	}

	public GenericApplicationContext(@Nullable ApplicationContext parent) {
		this();
		setParent(parent);
	}
}
```

# refresh

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 1. 设置上下文的 active 和 closed 标志, 记录启动时间. 给子类添加PropertySource的机会
        // 并且调用Environemnt抽象上的 validateRequiredProperties() 方法来验证必须属性.
        prepareRefresh();

        // 2. 抽象方法, 要求子类对内部的beanFactory进行刷新操作,如果支持的话.
        //    不能刷新的context会报错, 保证refresh()方法只能调用一次.
        //    能刷新的context, 会根据当前beanFactory的内容创建一个新的beanFactory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        //3. 挺重要的流程, 对刷新后的beanFactory进行基本设置工作.
        //设置的内容有:
        //            BeanExpressionResolver
        //            PropertyEditorRegistrar
        //            ApplicationContextAwareProcessor
        //            ApplicationListenerDetector
        //            注册单例:
        //            environment, systemProperties, systemEnvironment.
        prepareBeanFactory(beanFactory);

        try {
            // 在加载完所有的BeanDefintion, 然后必要的步骤(上面方法的行为)完成后. 允许子类扩展.
            // 像上面 prepareBeanFactory() 方法一样, 用来注册特殊的bean.
            postProcessBeanFactory(beanFactory);

            // 实例化并调用所有的 BeanFactroyPostProcessor Bean.
            // 这里其实有两步, 首先ApplicationContext支持编程式注册BeanFactoryProcessor实例.
            // addBeanFactoryPostProcessor().
            // 所以第一步是直接对内部的beanFactory使用直接注册的实例.
            // 第二步是扫描BeanFactory的beanDefintion来找到配置中的BeanFactoryProcessor,
            // 使用getBean()实例化后, 调用.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
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
