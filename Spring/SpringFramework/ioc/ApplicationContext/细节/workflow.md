# AnnotationApplication workflow

Creating shared instance of singleton bean '
org.springframework.context.annotation.internalConfigurationAnnotationProcessor'

# AbstractApplicatoinContext

AbstractApplicationContext 的 refresh() 方法定义了 Spring 容器启动时所执行的各项操作

```java
@Override
public void refresh() throws BeanException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        // 初始化 BeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            // 调用工厂后置处理器
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            // 注册事件监听器
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            // 初始化消息源
            initMessageSource();

            // Initialize event multicaster for this context.
            // 初始化事件广播器
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            // 初始化其他特殊的 Bean, 具体由子类实现
            onRefresh();

            // Check for listener beans and register them.
            // 注册事件监听器
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            // 实例化所有单例的 Bean, 使用 lazy 加载的除外
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            // 完成刷新并发布容器刷新事件
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

步骤

1.

> 初始化 **BeanFactory** :根据配置文件实例化 BeanFactory, 在 obtainFreshBeanFactory()方法中
> 首先调用 refreshBeanFactory()方法刷新 BeanFactory, 然后调用 getBeanFactory() 方法获取 BeanFactory,
> 这两个方法都是由具体子类实现的. 在这一步里, Spring 将配置文件的信息装入容器的 Bean 定义注册表(BeanDefinitionRegistry)中
> 但此时 Bean 还未初始化

2.

> 调用工厂后处理器: 根据反射机制从 BeanDefinitionRegistry 中找出所有实现了 BeanFactoryPostProcessor 接口的 Bean.
> 并且调用 postProcessBeanFactory()接口方法

3.

> 注册 Bean 后处理器: 根据反射机制从 BeanDefinitionRegistry 中找出所有实现 BeanPostProcessor 接口的Bean,
> 并将它们注册到容器 Bean 后处理器的注册表中.

4.

> 初始化消息源: 初始化容器的国际化消息资源

5.

> 初始化应用上下文事件

# Bean 的生命周期

1. InstantiationAwareBeanPostProcessor

2. 通过构造器或工厂方法产生bean实例

1. BeanNameAware
2. BeanFactoryAware
3. BeanPostProcessor
4. InitializingBean 接口
5. init-method 或 @PostConstruct 指定的初始化方法
