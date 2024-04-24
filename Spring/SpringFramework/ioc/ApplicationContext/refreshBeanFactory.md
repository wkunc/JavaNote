# refreshBeanFactory()

AbstractApplicationContext 中的refresh()方法调用流程中的分支
除了 obtaionFreshBeanFactory(), 其他都是大家基本一致的.
而 obtaionFreshbeanFactory() 调用了
抽象的 refreshBeanFactory() 方法.
在子类中有两种实现, 可刷新和不可刷新的

AbstractRefreshableApplicationContext 和 GenericApplicationContext.

# AbstractRefreshableApplicationContext

这个类重点就是可以多次调用 refresh() 方法.
所以它在 refreshBeanFactory() 中会销毁原来的 BeanFactory,
创建新的BeanFactory.

```java
protected final void refreshBeanFactory() throws BeansException {
    // 销毁已有的 BeanFactory
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    // 创建新的 BeanFactory
    try {
        // createBeanFactory() 方法就是 new 一个 BeanFactory 实例.
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
```

# GenericApplicationContext

这个类和上面可刷新的ApplicationContext不同, 它是不可重复调用
refresh() 方法.

```java
// 这是一个final方法, 意味着子类无法重写这个方法,
// 保证了子类不会重写这个方法从而变成可刷新的
protected final void refreshBeanFactory() throws IllegalStateException {
    // 判断 refresh() 方法是否已经被调用.
    if (!this.refreshed.compareAndSet(false, true)) {
        throw new IllegalStateException(
                "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
    }
    this.beanFactory.setSerializationId(getId());
}
```

我们发现在这个实现中Context不可刷新意味着BeanFactory对象不会重新创建.
那么它要在哪里创建呢?
答案就是在构造器中就创建了一个永久和这个Context绑定的BeanFactory.

```java
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
```
