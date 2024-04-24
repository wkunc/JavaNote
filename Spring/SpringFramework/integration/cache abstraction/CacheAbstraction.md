# Cache Abstraction

从3.1版开始, Spring Framework提供了对现有Spring应用程序透明地添加缓存的支持.
与事务支持类似, 缓存抽象允许一致地使用各种缓存解决方案,
而对代码的影响最小.

从Spring 4.1开始, 通过JSR-107注释和更多自定义选项的支持,
缓存抽象得到了显着扩展.

# Understanding the Cache Abstraction

# Declarative Annotation-based Caching

* @Cacheable: 触发缓存填充
* @CacheEvict: 触发缓存删除
* @CachePut: 更新缓存而不会干扰方法执行
* @Caching: 重新组合方法上应用的多个缓存操作
* @CacheConfig: 在类级别共享一些常见的缓存相关设置

## @Cacheable

顾名思义 @Cacheable 指示Spring这个被标记方法的返回值是可以缓存的.
Spring 会将其放入缓存中, 以便在后续调用(具有相同参数时)返回缓存中的值.
这样就无需实际执行该方法.

这个注解必须声明使用的Cache的名字, 下面是最简单的申明方式

```java
@Cacheable("books")
public Book findBook(ISBN isbn) {...}
```

在前面的代码中, findBook() 方法与名为 books 的缓存相关联.
每次调用该方法时, 都会检查缓存调用是否已执行且不必重复.
*虽然在大多数情况下, 只声明了一个缓存, 但是注释允许指定多个名称.
以便使用多个缓存, 在这种情况下, 在执行方法前会检测每个缓存,
如果有一个缓存命中, 就不必实际执行方法*

```java
@Cacheable({"books","isbns"})
public Book findBook(ISBN isbn) {...}
```

### Default Key Generation

由于高速缓存(Cache对象)本质上是 key-value 存储.
因此需要将使用缓存的方法的每次调用转换为合适的Key, 这样才能取到正确的值.

缓存抽象使用基于下面算法的 SimpleKeyGenerator.

1. 如果方法没有参数, 那么使用SimpleKey.EMPTY
2. 如果方法只有一个参数, 则返回该实例.
3. 如果方法有多个参数, 则返回包含所有参数的SimpleKey实例.

这需要方法参数实现有效正确的 hashCode() 和 equals() 方法,
默认的Key 生成策略适用于大多数实例.
如果不能满足情况, 我们可以自己实现这个策略.

# @CachePut

当需要更新缓存而不干扰方法执行时, 可以使用@CachePut注解.
也就是说, 始终执行方法, 并将其结果放入缓存中.

