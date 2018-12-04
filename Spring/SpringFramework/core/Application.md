# Application
![](imgs/Application.PNG)
# AbstractApplicationContext
有三个 abstract (抽象)方法
```java
//这两个是自己提供的
protected abstract void refreshBeanFactory();
protected abstract void closeBeanFactory();
// 这个是从 ConfigurableApplication 中遗留的
public abstract ConfigurableListableBeanFactory getBeanFactory();
```

这里使用了模板方法, 将方法调用过程放在父类中, 子类实现这些方法的细节
这个方法是核心方法, 第一次加载配置和以后的刷新配置都是经过这个方法
```java
    @Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
            // 预备刷新
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
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



# 子类
GenericApplicationContext(通用的applicationcontext) 和
AbstractRefreshableApplicationContext 继承并实现了 AbstractApplication

Spring 在这里搞了两套实现


AbstractRefreshableApplicationContext 实现了中的三个抽象方法, 
并且提供了一个加载配置的抽象方法
```java
protected abstract void loadBeanDefinitions(DefaultListableBeanFactory beanFactory);
```
