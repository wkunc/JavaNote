# SingletonBeanRegistry


为共享 Bean 实例定义注册表接口.
注意这个接口提供了直接注册对象到IOC容器中的方法, 
也意味着IOC容器不会控制这个对象的生命周期.
IOC容器中的某些方法(查找判断之类的)不会理会这些注册进来的Bean
```java
public interface SingletonBeanRegistry {
    // 注册一个 bean
    void registerSingleton(String beanName, Object singletonObject);

    // 获得指定 name 的 bean
    Object getSingleton(String beanName);

    // 判断是否有给定 name 的bean
    boolean containsSingleton(String beanName);

    // 获得所有 bean 的注册名
    String[] getSingletonNames();

    // 获得注册表中 bean 的数量
    int getSingletOnCount();

    // return the singleton mutex used by this registry (for external collaborators).
    // 和线程安全相关, 返回一个用于互斥访问的同步对象.
    Object getSingletonMutex();
}
```

# DefaultSingletonBeanRegistry 
Spring提供了 DefaultSingletonBeanRegistry 实现了这个接口.

共享 bean 实例的通用注册表, 实现SingletonBeanRegistry.
允许通过 bean 名称注册应该为注册表的所有调用者共享的单例实例.
还支持在关闭注册表时销毁DisposableBean实例(可能对应于已注册的单例，也可能不对应于已注册的单例)

可以注册bean之间的依赖关系以强制执行适当的关闭顺序.(ps: )
该类主要用作org.springframework.beans.factory.BeanFactory实现的基类, 分解了单例bean实例的公共管理.

## 字段分析
```java

public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {

    // 存放已注册的的Bean实例, key beanName, value Object
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

    // 存放的用来初始化 SingleBean 的对象工厂实例,  key BeanName, value ObjectFactory
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

    // 用来存放已经存在但是未注册的 SingleBean 实例
	private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);

    // 存放已注册的 SingleBean 对象名字, 它是个 LinkedHashSet 所以还是保留了注册顺序信息
	private final Set<String> registeredSingletons = new LinkedHashSet<String>(256);

    //
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>(16));

    //
	private final Set<String> inCreationCheckExclusions =
			Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>(16));

	private Set<Exception> suppressedExceptions;

	/** Flag that indicates whether we're currently within destroySingletons */
	private boolean singletonsCurrentlyInDestruction = false;

    // key 是 name, value 是 DisposableBean 实例
	private final Map<String, Object> disposableBeans = new LinkedHashMap<String, Object>();

    // key 是 beanName, value 是对应的 bean 包含的 bean Name.
	private final Map<String, Set<String>> containedBeanMap = new ConcurrentHashMap<String, Set<String>>(16);

    //依赖于这个 bean 的所有 bean
	private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<String, Set<String>>(64);

    //当前 bean 依赖的所有 bean
	private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<String, Set<String>>(64);
}
```

## 注册方法分析
```java
// 主要执行注册前的判断, 如果给定的 beanName 已经有对应的 Bean 了就报异常. 没有被注册的情况下执行注册.
@Override
public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
    // 省略参数非null判断的Assert语句...
    synchronized (this.singletonObjects) {
        Object oldObject = this.singletonObjects.get(beanName);
        if (oldObject != null) {
            throw new IllegalStateException("Could not register object [" + singletonObject +
                    "] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
        }
        addSingleton(beanName, singletonObject);
    }
}
protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        // 注册bean
        this.singletonObjects.put(beanName, singletonObject);

        this.singletonFactories.remove(beanName);
        this.earlySingletonObjects.remove(beanName);
        // 添加注册名
        this.registeredSingletons.add(beanName);
    }
}
```
## 获取方法
```java
// 实现 SingletonRegistry 接口方法
@Override
@Nullable
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

// 按给定的 name 返回 raw 的被注册的单例对象, 检查已经实例化的单例和允许一个 早期引用,
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 首先从缓存的对象中获取, 如果获取不到. 
    // 判断当前想要获取的单例 Bean 是否正在被创建(within the entire factory)
    Object singletonObject = this.singletonObjects.get(beanName);

    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {

        synchronized (this.singletonObjects) {
            // 从 earlySingletonObjects 中获取
            singletonObject = this.earlySingletonObjects.get(beanName);

            // 如果没有获取到, 并且接受返回早期引用 
            if (singletonObject == null && allowEarlyReference) {
                // 从 singletonnFactories 中获取对应的工厂, 从工厂获取,  并从移除对应工厂
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    // earlySingletonObjects 这个map的唯一调用put方法的地方.
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
// 返回给定beanName对应的 bean 是否正在被实例化.
public boolean isSingletonCurrentlyInCreation(String beanName) {
    return this.singletonsCurrentlyInCreation.contains(beanName);
}

// 这是类中补的方法, 给定name返回底层已经注册的bean,
//如果name没有注册bean, 就创建并注册一个
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    synchronized (this.singletonObjects) {
        // 尝试获取目标bean
        Object singletonObject = this.singletonObjects.get(beanName);
        // 没有获取到就开始创建并注册
        if (singletonObject == null) {
            // 如果当前 beanFactory 真正销毁单例对象, 则不允许创建, 并警告不要在 BeanFactoy 销毁方法时,请求一个 bean.
            if (this.singletonsCurrentlyInDestruction) {
                throw new BeanCreationNotAllowedException(beanName,
                        "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                        "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
            }
            // 以下开始创建 bean.
            if (logger.isDebugEnabled()) {
                logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
            }
            // 在创建bean之前的回调方法. 默认实现是执行向 Set<String> singltonsCurrentlyInCreation 添加beanName.
            beforeSingletonCreation(beanName);

            boolean newSingleton = false;
            // 判断是否需要new 一个 set 来保存异常. 在finally 会将其置为 null, 每次调用都会创建.
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
            if (recordSuppressedExceptions) {
                this.suppressedExceptions = new LinkedHashSet<>();
            }
            try {
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            catch (IllegalStateException ex) {
                // Has the singleton object implicitly appeared in the meantime ->
                // if yes, proceed with it since the exception indicates that state.
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    throw ex;
                }
            }
            catch (BeanCreationException ex) {
                // 记录异常
                if (recordSuppressedExceptions) {
                    for (Exception suppressedException : this.suppressedExceptions) {
                        ex.addRelatedCause(suppressedException);
                    }
                }
                throw ex;
            }
            finally {
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = null;
                }
                // 创建bean完成后的回调, 默认实现是标记bean不是正在创建中
                // (就是将 beanName 从 singltonsCurrentlyInCreation 中移除.)
                afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}

protected void beforeSingletonCreation(String beanName) {
    if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
        throw new BeanCurrentlyInCreationException(beanName);
    }
}
protected void afterSingletonCreation(String beanName) {
    if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
        throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
    }
}
```

# 注册销毁依赖
之前提到这个SingletonBeanRegistry支持按给定顺序销毁 bean 对象.

```java
// 注册 DisposableBean 的对象
public void registerDisposableBean(String beanName, DisposableBean bean) {
    synchronized (this.disposableBeans) {
        this.disposableBeans.put(beanName, bean);
    }
}

// 注册两个bean之间的包含关系, 还会根据包含关系注册销毁顺序,
// 内部的bean 应该在外部bean之前销毁, 也就是说, 外部bean销毁依赖内部bean.
// 应该对应 xml中的 inner bean
public void registerContainedBean(String containedBeanName, String containingBeanName) {
    synchronized (this.containedBeanMap) {
        Set<String> containedBeans =
                this.containedBeanMap.computeIfAbsent(containingBeanName, k -> new LinkedHashSet<>(8));
        if (!containedBeans.add(containedBeanName)) {
            return;
        }
    }
    registerDependentBean(containedBeanName, containingBeanName);
}

// 注册显式的销毁顺序, 
public void registerDependentBean(String beanName, String dependentBeanName) {
    String canonicalName = canonicalName(beanName);

    synchronized (this.dependentBeanMap) {
        Set<String> dependentBeans =
                this.dependentBeanMap.computeIfAbsent(canonicalName, k -> new LinkedHashSet<>(8));
        if (!dependentBeans.add(dependentBeanName)) {
            return;
        }
    }

    synchronized (this.dependenciesForBeanMap) {
        Set<String> dependenciesForBean =
                this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, k -> new LinkedHashSet<>(8));
        dependenciesForBean.add(canonicalName);
    }
}

// 判断是否是依赖关系.
protected boolean isDependent(String beanName, String dependentBeanName) {
    synchronized (this.dependentBeanMap) {
        return isDependent(beanName, dependentBeanName, null);
    }
}

private boolean isDependent(String beanName, String dependentBeanName, @Nullable Set<String> alreadySeen) {
    if (alreadySeen != null && alreadySeen.contains(beanName)) {
        return false;
    }
    String canonicalName = canonicalName(beanName);
    Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
    if (dependentBeans == null) {
        return false;
    }
    if (dependentBeans.contains(dependentBeanName)) {
        return true;
    }
    for (String transitiveDependency : dependentBeans) {
        if (alreadySeen == null) {
            alreadySeen = new HashSet<>();
        }
        alreadySeen.add(beanName);
        if (isDependent(transitiveDependency, dependentBeanName, alreadySeen)) {
            return true;
        }
    }
    return false;
}

```
# FactoryBeanRegistrySupport
在支持注册单例bean的基础上添加了对 interface- FactoryBean<?> 的支持
```java
public abstract class FactoryBeanRegistrySupport extends DefaultSingletonBeanRegistry {

	private final Map<String, Object> factoryBeanObjectCache = new ConcurrentHashMap<>(16);

	@Nullable
	protected Class<?> getTypeForFactoryBean(final FactoryBean<?> factoryBean) {
		try {
			if (System.getSecurityManager() != null) {
				return AccessController.doPrivileged((PrivilegedAction<Class<?>>)
						factoryBean::getObjectType, getAccessControlContext());
			}
			else {
				return factoryBean.getObjectType();
			}
		}
		catch (Throwable ex) {
			// Thrown from the FactoryBean's getObjectType implementation.
			logger.info("FactoryBean threw exception from getObjectType, despite the contract saying " +
					"that it should return null if the type of its object cannot be determined yet", ex);
			return null;
		}
	}

	@Nullable
	protected Object getCachedObjectForFactoryBean(String beanName) {
		return this.factoryBeanObjectCache.get(beanName);
	}

	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
		if (factory.isSingleton() && containsSingleton(beanName)) {
			synchronized (getSingletonMutex()) {
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
					object = doGetObjectFromFactoryBean(factory, beanName);
					// Only post-process and store if not put there already during getObject() call above
					// (e.g. because of circular reference processing triggered by custom getBean calls)
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					if (alreadyThere != null) {
						object = alreadyThere;
					}
					else {
						if (shouldPostProcess) {
							if (isSingletonCurrentlyInCreation(beanName)) {
								// Temporarily return non-post-processed object, not storing it yet..
								return object;
							}
							beforeSingletonCreation(beanName);
							try {
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
										"Post-processing of FactoryBean's singleton object failed", ex);
							}
							finally {
								afterSingletonCreation(beanName);
							}
						}
						if (containsSingleton(beanName)) {
							this.factoryBeanObjectCache.put(beanName, object);
						}
					}
				}
				return object;
			}
		}
		else {
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			if (shouldPostProcess) {
				try {
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			return object;
		}
	}

	private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
			throws BeanCreationException {

		Object object;
		try {
			if (System.getSecurityManager() != null) {
				AccessControlContext acc = getAccessControlContext();
				try {
					object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				object = factory.getObject();
			}
		}
		catch (FactoryBeanNotInitializedException ex) {
			throw new BeanCurrentlyInCreationException(beanName, ex.toString());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
		}

		// Do not accept a null value for a FactoryBean that's not fully
		// initialized yet: Many FactoryBeans just return null then.
		if (object == null) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(
						beanName, "FactoryBean which is currently in creation returned null from getObject");
			}
			object = new NullBean();
		}
		return object;
	}

	protected Object postProcessObjectFromFactoryBean(Object object, String beanName) throws BeansException {
		return object;
	}

	protected FactoryBean<?> getFactoryBean(String beanName, Object beanInstance) throws BeansException {
		if (!(beanInstance instanceof FactoryBean)) {
			throw new BeanCreationException(beanName,
					"Bean instance of type [" + beanInstance.getClass() + "] is not a FactoryBean");
		}
		return (FactoryBean<?>) beanInstance;
	}

	@Override
	protected void removeSingleton(String beanName) {
		synchronized (getSingletonMutex()) {
			super.removeSingleton(beanName);
			this.factoryBeanObjectCache.remove(beanName);
		}
	}

	@Override
	protected void clearSingletonCache() {
		synchronized (getSingletonMutex()) {
			super.clearSingletonCache();
			this.factoryBeanObjectCache.clear();
		}
	}

	protected AccessControlContext getAccessControlContext() {
		return AccessController.getContext();
	}

}
```
