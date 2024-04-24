# CacheManager

Spring 缓存抽象的核心(central cache manager SPI)
允许获取指定名字的 Cache

```java
public interface CacheManger {
    // 获取根据名字获取一个Cache对象. 就是我们缓存行为中的Key
    Cache getCache(String name);
    // 一个方便暴露所有已存在Cache的名字的方法
    Collection<String> getCacheNames();
}
```

![](CacheManger.png)

Spring 为这个 CacheManager 提供了很多不同的实现.

这个AbstractCacheManager实现了一些公共的方法, 为子类实现提供便利.
这个类很简单用一个Map\<String, Cache\>来存储Cache实例, 这样就可以根据name获取对应的Cache了.
还用一个Set\<String\> 来保存已经注册的Cache实例的名字, 这样就可以方便的实现getCacheNames()方法了

这个类还提供了额外的初始化逻辑 initalizingBean 接口.

这个类实现了getCache()方法, 逻辑也很简单(基于map实现, 还给子类行为做了定义).
如果没有获取到, 那么根据子类的设置可以进行创建也可以就返回null
(通常我们会允许创建, 在初始化时就配置所有的缓存是不可能的).

```java
public abstract class AbstractCacheManager implements CacheManager, InitializingBean {

	private final ConcurrentMap<String, Cache> cacheMap = new ConcurrentHashMap<>(16);

	private volatile Set<String> cacheNames = Collections.emptySet();

    // 初始化逻辑
	@Override
	public void afterPropertiesSet() {
		initializeCaches();
	}
	public void initializeCaches() {
        // 调用一个抽象方法, 用来保证 cacheMap 的初始化.
		Collection<? extends Cache> caches = loadCaches();

		synchronized (this.cacheMap) {
			this.cacheNames = Collections.emptySet();
			this.cacheMap.clear();
			Set<String> cacheNames = new LinkedHashSet<>(caches.size());
			for (Cache cache : caches) {
				String name = cache.getName();
				this.cacheMap.put(name, decorateCache(cache));
				cacheNames.add(name);
			}
			this.cacheNames = Collections.unmodifiableSet(cacheNames);
		}
	}

	/**
	 * The returned collection may be empty but must not be {@code null}.
	 */
	protected abstract Collection<? extends Cache> loadCaches();


    // 延迟了 Cache 对象的初始化, 为了线程安全好像还写了个双重检测锁
	@Override
	@Nullable
	public Cache getCache(String name) {
		Cache cache = this.cacheMap.get(name);
		if (cache != null) {
			return cache;
		}
		else {
			// Fully synchronize now for missing cache creation...
			synchronized (this.cacheMap) {
				cache = this.cacheMap.get(name);
				if (cache == null) {
                    // 获取不到的情况下调用 getMissingCache()方法默认返回null,
                    // 主要给子类扩展, 通常来说会创建一个新的缓存对象
					cache = getMissingCache(name);
					if (cache != null) {
                        // 如果创建了一个新的缓存, 就调用 decorateCache() 方法
                        // 目的是使用装饰器包装这个Cache对象实现一些功能.
                        // 默认实现是什么都不做, 有个关于事务的子类, 就是重写了这个方法, 用特定的事务Cache包装了新创建的Cache对象.
						cache = decorateCache(cache);
						this.cacheMap.put(name, cache);
                        // 更新CacheName集合
						updateCacheNames(name);
					}
				}
				return cache;
			}
		}
	}

	@Override
	public Collection<String> getCacheNames() {
		return this.cacheNames;
	}


	// Common cache initialization delegates for subclasses

	/**
     * 根据给定的缓存名获取Cache对象.
     * 和接口定义的 getCache(name) 方法比这个方法不会触发为找到的Cache的创建.
     * 是提供给子类访问 cacheMap 的手段. 就只有一个Redis实现的子类用了这个方法
	 */
	@Nullable
	protected final Cache lookupCache(String name) {
		return this.cacheMap.get(name);
	}

    // 被getMissingCache() 方法取代了
	@Deprecated
	protected final void addCache(Cache cache) {
		String name = cache.getName();
		synchronized (this.cacheMap) {
			if (this.cacheMap.put(name, decorateCache(cache)) == null) {
				updateCacheNames(name);
			}
		}
	}

	private void updateCacheNames(String name) {
		Set<String> cacheNames = new LinkedHashSet<>(this.cacheNames.size() + 1);
		cacheNames.addAll(this.cacheNames);
		cacheNames.add(name);
		this.cacheNames = Collections.unmodifiableSet(cacheNames);
	}


	// Overridable template methods for cache initialization

	protected Cache decorateCache(Cache cache) {
		return cache;
	}

	@Nullable
	protected Cache getMissingCache(String name) {
		return null;
	}

}
```

RedisCacheManager 我个人比较常用的具体实现.

```java
// 注意它不是直接继承上面说的AbstractCacheManager的, 它的父类就是之前提到负责为事务包装每个新Cache对象的实现
public class RedisCacheManager extends AbstractTransactionSupportingCacheManager {

    // 这些字段都是final的说明不可改变引用, 所以也就没有必要提供setter方法
    // 必须在调用构造器时就完成所有的配置工作. 这也是提供Builder的原因吧
	private final RedisCacheWriter cacheWriter;
	private final RedisCacheConfiguration defaultCacheConfig;
	private final Map<String, RedisCacheConfiguration> initialCacheConfiguration;
	private final boolean allowInFlightCacheCreation;

    // 一个私有的构造器, 下面还有多个公开的构造器
    // 还提供了Buidler类来更好创建这个对象.
    // 不过这些构造器都需要一个 RedisCacheWriter 使用起来不是很方便,
    // 然后Builder提供了从RedisConnectionFactory创建的操作
    // 还有提供了一些初始化配置的手段
	private RedisCacheManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration,
			boolean allowInFlightCacheCreation) {

		Assert.notNull(cacheWriter, "CacheWriter must not be null!");
		Assert.notNull(defaultCacheConfiguration, "DefaultCacheConfiguration must not be null!");

		this.cacheWriter = cacheWriter;
		this.defaultCacheConfig = defaultCacheConfiguration;
		this.initialCacheConfiguration = new LinkedHashMap<>();
		this.allowInFlightCacheCreation = allowInFlightCacheCreation;
	}

}
```

不过内部的Builder很简单, 也不重要. 重要的是下面两个方法

```java
// 实现AbstractCacheManager中的抽象方法, 会在Bean初始化后调用. 负责初始Cache的创建
// 主要调用 createRedisCache() 方法
@Override
protected Collection<RedisCache> loadCaches() {

    List<RedisCache> caches = new LinkedList<>();

    for (Map.Entry<String, RedisCacheConfiguration> entry : initialCacheConfiguration.entrySet()) {
        caches.add(createRedisCache(entry.getKey(), entry.getValue()));
    }

    return caches;
}

// 重写 getMissingCache 方法, 会根据一个布尔标志位判断是否要创建没有找到的Cache对象.
// 也一样调用了 createRedisCache() 方法, 不过注意传入的值有区别.
@Override
protected RedisCache getMissingCache(String name) {
    return allowInFlightCacheCreation ? createRedisCache(name, defaultCacheConfig) : null;
}

// 调用 RedisCache 的构造器, 如果传入的 RedisCacheConfiguration 是null, 就用默认的.
protected RedisCache createRedisCache(String name, @Nullable RedisCacheConfiguration cacheConfig) {
    return new RedisCache(name, cacheWriter, cacheConfig != null ? cacheConfig : defaultCacheConfig);
}
```
