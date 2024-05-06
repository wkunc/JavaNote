# 循环依赖

Bean注册顺序或者说Bean初始化顺序引发的循环依赖问题.

## Bean注册顺序对循环依赖问题的影响

Spring 官网描述了循环依赖问题,
网络上也有很多分析Bean创建过程来分析Spring是如何解决部分循环依赖的.
大致就是允许获取Bean的早期引用来解决

<span class="spurious-link"
target="Spring文档描述Circular dependencies">*<https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html#beans-dependency-resolution>*</span>
描述了将构造器注入替换为Setter注入解决循环依赖的问题.

### 代码示例

如果 `ServiceA`, `ServiceB` 通过构造器互相依赖.
在运行时我们就会得到一个异常, SpringBoot 会给出一个友好的提示.

``` example
***************************
APPLICATION FAILED TO START
***************************

Description:

The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  serviceA defined in file [/Users/wkunc/IdeaProjects/DI-Cycle/target/classes/com/example/dicycle/ServiceA.class]
↑     ↓
|  serviceB defined in file [/Users/wkunc/IdeaProjects/DI-Cycle/target/classes/com/example/dicycle/ServiceB.class]
└─────┘
```

接下来我们将 `ServiceA` 修改为setter方法注入.

//@Lazy @Component public class ServiceA {

private ServiceB b;

@Autowired public void setB(ServiceB b) { this.b = b; }

@PreDestroy public void hello() { System.out.println("hello, A"); }

}

@Component public class ServiceB {

private ServiceA a;

@Autowired public ServiceB(ServiceA a) { this.a = a; }

// public void setA(ServiceA a) { // this.a = a; // }

@PreDestroy public void hello() { System.out.println("hello, B"); } }

@SpringBootApplication public class DiCycleApplication {

public static void main(String\[\] args) {
ConfigurableApplicationContext ctx =
SpringApplication.run(DiCycleApplication.class, args); }

}

运行代码发现确实成功运行, 没有循环依赖错误提示了.
但是事情真的完美解决了吗?

只需要将 `ServiceA` 标记为Lazy再次运行, 就又出现了循环依赖问题.

``` example
***************************
APPLICATION FAILED TO START
***************************

Description:

The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  serviceB defined in file [/Users/wkunc/IdeaProjects/DI-Cycle/target/classes/com/example/dicycle/ServiceB.class]
↑     ↓
|  serviceA
└─────┘
```

### 原因分析

创建Bean的流程可以简化为

1.  调用构造器
2.  注册早期引用, 这样及时这个Bean依赖的组件同时依赖了当前Bean.
    那也可以通过早期引用直接获取解决循环依赖
3.  填充属性, 也就是Setter注入依赖
4.  initializeBean 比如
    BeanPostProcessor.postProcessBeforeInitialization(), 各种 Aware
    接口, @PostConstruct 等指定的Bean init 方法

而构造器注入具体发生在调用构造器的步骤.

1.  确定使用的构造器
2.  解析构造器参数, 然后会识别给这些依赖的Bean调用 BeanFactory.getBean()
    方法

调用 `BeanFactory.getBean(ServiceB.class);` 流程

1.  确认ServiceB.class的构造器
2.  解析构造器参数调用 getBean() 方法
    1.  相当于BeanFactory.getBean(ServiceA.class);
    2.  调用ServiceA构造器(这里是默认构造器)
    3.  注册ServiceA的早期引用
    4.  填充A的属性, 发现需要ServiceB. 调用getBean(ServiceB.class)
    5.  而B正在创建中, 检测到循环依赖. 抛出异常

调用 `BeanFactory.getBean(ServiceA.class);` 流程

1.  确认ServiceA.class的构造器
2.  调用构造器(A 没有构造器注入,采用默认构造器调用)
3.  注册A的早期引用
4.  填充A的属性, 发现需要ServiceB. 调用getBean(ServiceB.class)
5.  确定B的构造器, 使用构造器注入A (由于A已经注册了早期引用, 所以B
    这里可以轻松的获取到A)
6.  返回B
7.  完成A的属性填充, 也就是Setter注入

到这里就可以发现Bean的创建顺序会影响是否会产生报错.
前面使用了@Lazy让ServiceB先被创建.

### Bean 的创建顺序如何确定

众所周知, BeanFactory 是一个Lazy的容器实现, 即在首次调用 getBean()
时才会触发对应Bean的创建 而 ApplcationContext 实现了及早创建Bean. 通过
ConfigurableApplicationContext.refresh() 方法 完成IOC容器的初始化工作.
最后会调用到 BeanFactory.preInstantiateSingletons() 方法上.

而BeanFactory.preInstantiateSingletons()就是简单的遍历
beanDefinitionNames 按顺序初始化. beanDefinitionNames
这个的顺序取决于BeanDefinition注册的顺序.

这时需要查看Spring配置类,xml解析的相关代码.

1.  重复运行流水线, 打包出的镜像偶发循环依赖的原因

    `Idea` 运行时会取决于target/classes 目录里的 **文件顺序**
    也就是字母顺序 `Spring Boot jar` 会取决于jar包里的 **文件顺序**,
    也就取决于 `maven-jar-plugin` 实现的打包顺序 公司使用的
    `maven-jar-plugin` 插件版本为 `3.1.2` 此时打包时的文件顺序是随机的
    **(只看了现象未分分析3.1.2代码)** 每次运行 `mvn clean package`
    产生的jar包中都有不同的结果

### Maven 打包设置

查看 [maven-jar-plugin](https://github.com/apache/maven-jar-plugin)
源码. 可以找到解决方法

``` xml
<properties>
    <!--使用3.2.0以上版本的打包插件可以固定jar包中class的顺序>
    <!--配置outputTimestamp可以按照字母顺序排序-->
    <maven-jar-plugin.version>3.2.0</maven-jar-plugin.version>
    <project.build.outputTimestamp>${maven.build.timestamp}</project.build.outputTimestamp>
</properties>
```


## @Aync 等注解导致循环依赖报错问题

可复现的代码
```java
@Component
public class ServiceA {

    @Autowired
    private ServiceB b;

    @Async
    public void asyncMethod() {

    }

    @Transactional
    public void test() {

    }
}

@Component
public class ServiceB {

    @Autowired
    private ServiceA a;

    @Transactional
    public void test() {

    }
}

```

> org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'serviceA': Bean with name 'serviceA' has been injected into other beans [serviceB] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.
>     at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:622) ~[spring-beans-5.1.10.RELEASE.jar:5.1.10.RELEASE]
>     at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:515) ~[spring-beans-5.1.10.RELEASE.jar:5.1.10.RELEASE]
>     at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:320) ~[spring-beans-5.1.10.RELEASE.jar:5.1.10.RELEASE]
>     at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:222) ~[spring-beans-5.1.10.RELEASE.jar:5.1.10.RELEASE]
>     at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:318) ~[spring-beans-5.1.10.RELEASE.jar:5.1.10.RELEASE]

看错误提示很容易理解: 由于监测到serviceA 和 serviceB 存在循环依赖,
并且 serviceB 中注入了 serviceA 的早期引用不是最终的版本(未经过代理)所以报错提示.
并且可以通过 `allowRawInjectionDespiteWrapping = false`属性修改行为(允许注入原始版本)

疑问点: 看上去是由于`@Async`使用代理技术导致的循环依赖问题, 想到到`@Transactional`也用了代理技术为何不会导致相同的错误.
并且事务确实生效.

### TODO 代码分析


```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
            // 重点为了解析循环引用时, 注册的不是当前的bean, 而是一个lambda. 这样其他依赖的bean获取到的就是被提前包装过的
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
            // 填充属性, 也就是会触发依赖注入. 
            // 可以理解通过 beanFactory.getBean() 获取类并填充字段.
            // 此时如果存在循环依赖, 会getBean() 方法中获取到早期引用.
			populateBean(beanName, mbd, instanceWrapper);
            // 调用Bean声明的 init 方法
            // 以及 BeanPostProcessor.postProcessBeforeInitialization() 和 BeanPostProcessor.postProcessAfterInitialization()
            // @Async 等机制是通过 BeanPostProcessor.postProcessAfterInitialization() 返回一个代理对象实现.
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
            // 省略异常处理代理
		}

        // 这里就是监测,是不是存在早期引用和最终引用不一致的情况
		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}

```

### 结论
`@Transactional` 以及基于注解的`AOP`是通过 `AnnotationAwareAspectJAutoProxyCreator` 实现的.
实现了 `SmartInstantiationAwareBeanPostProcessor` 接口. 意味着事务以及AOP逻辑的代理逻辑会在获取早期引用时提前进行.
这样循环依赖中注入的早期引用就是代理过的版本. 后续调用 BeanPostProcessor.postProcessBeforeInitialization() 时 BeanPostProcessor.postProcessAfterInitialization()
不会重复代理
```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}

```

`@Async` 是通过 `AsyncAnnotationBeanPostProcessor` 没有实现`SmartInstantiationAwareBeanPostProcessor`
因此因为解决循环引用时导致的获取早期引用的流程中, 并不会提前生成代理. 
而是调用 BeanPostProcessor.postProcessAfterInitialization() 时完成代理逻辑.
此时就导致循环依赖路径上依赖该bean的获取到的原始版本. 从而被监测到然后报错

> 翻阅继承树, 可以知道方法级别的参数校验@Validated (不是Controller层的,mvc中的数据校验时独立的逻辑)
> 以及 `PersistenceExceptionTranslationPostProcessor` 如果Bean是用`@Repository` 标记的会代理方法实现. 
> 持久层异常的包装(所以把错误代码中的`@Async`注解去掉, 改用`@Repository`标记类也可以触发一样的问题)


所以使用`@Async`等机制时需要避免循环依赖(因为Spring无法解决)
> 特殊思考: 为何 `AsyncAnnotationBeanPostProcessor` 不去实现 `SmartInstantiationAwareBeanPostProcessor`
> 从而达到可以类似 `@Transactional` 一样的效果呢

### 解决方法
1. 新的spring版本修改了`@Async`的实现 [相关提交](https://github.com/spring-projects/spring-framework/issues/22370)
> `Spring 6.0`
