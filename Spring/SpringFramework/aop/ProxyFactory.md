# ProxyFactory

# ProxyCreatorSupport

```java
public class ProxyCreatorSupport extends AdvisedSupport {

	private AopProxyFactory aopProxyFactory;

	private final List<AdvisedSupportListener> listeners = new LinkedList<>();

	/** Set to true when the first AOP proxy has been created. */
	private boolean active = false;

	public ProxyCreatorSupport() {
		this.aopProxyFactory = new DefaultAopProxyFactory();
	}

	public ProxyCreatorSupport(AopProxyFactory aopProxyFactory) {
		Assert.notNull(aopProxyFactory, "AopProxyFactory must not be null");
		this.aopProxyFactory = aopProxyFactory;
	}
}
```

## AopProxyFactory
实际上工作的接口, 根据 AdvisedSupport 对象(可以视为AOP的配置对象, 包含配置信息如: targetObject, Advisor,...)
生成一个AopProxy 接口的实现对象.
Spring 提供的 DefaultAopProxyFactory , 就是简单的根据配置信息确定调用哪一个 AopProxy 实现对象的构造器.

```java
public interface AopProxyFactory {
    // 根据配置信息创建 AopProxy 对象
	AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException;
}
```

```java
/*
* 创建JDK动态代理或者CGLIB代理.
* 如果给定的AdvisedSupport实例满足以下条件, 则创建CGLIB代理:
* 1. optimize flag is set (优化标志为 true)
* 2. the proxyTargetClass flag is set (代理目标类标志为true)
* 3. no proxy interfaces have been specified (没有指定代理接口)
*/
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}

	private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
		Class<?>[] ifcs = config.getProxiedInterfaces();
		return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
	}

}
```

## AdvisedSupprot

```java
public interface Advised extends TargetClassAware {

}


// 只有5个boolean标志和对应的getter setter
public class ProxyConfig implements Serializable {

	private static final long serialVersionUID = -8409359707199703185L;

    // 在@Enable...注解中可以设置
	private boolean proxyTargetClass = false;

	private boolean optimize = false;

	boolean opaque = false;

    // 在@Enable...注解中可以设置
	boolean exposeProxy = false;

	private boolean frozen = false;
    // getter setter...
}
```


```java
// TargetSource , List<Advisor> advisors, List<Class<?>>interfaces
public class AdvisedSupport extends ProxyConfig implements Advised {
	private static final long serialVersionUID = 2651364800145442165L;

	public static final TargetSource EMPTY_TARGET_SOURCE = EmptyTargetSource.INSTANCE;

	/** Package-protected to allow direct access for efficiency. */
	TargetSource targetSource = EMPTY_TARGET_SOURCE;

    // 
	private boolean preFiltered = false;

	/** The AdvisorChainFactory to use. */
	AdvisorChainFactory advisorChainFactory = new DefaultAdvisorChainFactory();

	/** Cache with Method as key and advisor chain List as value. */
	private transient Map<MethodCacheKey, List<Object>> methodCache;

    //
	private List<Class<?>> interfaces = new ArrayList<>();

	private List<Advisor> advisors = new ArrayList<>();

	private Advisor[] advisorArray = new Advisor[0];

}
```

提供一些方便配置方法, 可以直接添加 advice(内部会包装成合适的切面).
当然也可以添加 Advisor.
```java
// 
public void addAdvice(Advice advice) throws AopConfigException {
    int pos = this.advisors.size();
    addAdvice(pos, advice);
}

public void addAdvice(int pos, Advice advice) throws AopConfigException {
    Assert.notNull(advice, "Advice must not be null");
    if (advice instanceof IntroductionInfo) {
        // We don't need an IntroductionAdvisor for this kind of introduction:
        // It's fully self-describing.
        addAdvisor(pos, new DefaultIntroductionAdvisor(advice, (IntroductionInfo) advice));
    }
    else if (advice instanceof DynamicIntroductionAdvice) {
        // We need an IntroductionAdvisor for this kind of introduction.
        throw new AopConfigException("DynamicIntroductionAdvice may only be added as part of IntroductionAdvisor");
    }
    else {
        addAdvisor(pos, new DefaultPointcutAdvisor(advice));
    }
}
```

## AopProxy
```java
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {

	private static final long serialVersionUID = 5531744639992436476L;

	/** We use a static Log to avoid serialization issues. */
	private static final Log logger = LogFactory.getLog(JdkDynamicAopProxy.class);

	/** Config used to configure this proxy. */
	private final AdvisedSupport advised;

    // 是否要为生成的代理设置合适的equals, hashCode方法的标志.
	private boolean equalsDefined;
	private boolean hashCodeDefined;

    // 构造器
	public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
		Assert.notNull(config, "AdvisedSupport must not be null");
		if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
			throw new AopConfigException("No advisors and no TargetSource specified");
		}
		this.advised = config;
	}

    // 实现AopProxy接口, 使用JDK Proxy.newProxyInstance() 方法实现.
	@Override
	public Object getProxy() {
		return getProxy(ClassUtils.getDefaultClassLoader());
	}

	@Override
	public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isTraceEnabled()) {
			logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
		}
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
}
```


