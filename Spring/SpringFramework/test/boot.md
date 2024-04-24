#

因为 @RunWith() 注解, 所以JUnit确定 SpringJunit4ClassRunner 作为这个测试类的Runner.

然后调用其构造器.

```java
public SpringJUnit4ClassRunner(Class<?> clazz) throws InitializationError {
    super(clazz);
    if (logger.isDebugEnabled()) {
        logger.debug("SpringJUnit4ClassRunner constructor called with [" + clazz + "]");
    }
    ensureSpringRulesAreNotPresent(clazz);
    // 重点. 创建 TestContextManager.
    this.testContextManager = createTestContextManager(clazz);
}

protected TestContextManager createTestContextManager(Class<?> clazz) {
    return new TestContextManager(clazz);
}
```

调用 TestContextManager 的构造器.

TestContextBootstrapper 是spring test的用来自定义的接口, 默认实现可以满足大部分需求.
如果需要更改默认的 ContextLoader, 实现自定义的 TestContext 或 ContextCahce,
扩充 ContextCustomizerFactory 和 TestExecutionListener 实现的默认集.
可以实现这个接口以完成目的.

```java
// 首先为传入的 测试类 确定使用的 TestContextBootstrapper.
// 就是读取 @BootstrapWith 注解信息, 如果没有就使用默认的.
public TestContextManager(Class<?> testClass) {
    this(BootstrapUtils.resolveTestContextBootstrapper(BootstrapUtils.createBootstrapContext(testClass)));
}

// 最终的构造器.
// 用传入的 TestContextBootstrapper 实现类对象构建对应测试类的 TestContext.
// 并且注册对应的 TestExecutionListener 集.
public TestContextManager(TestContextBootstrapper testContextBootstrapper) {
    this.testContext = testContextBootstrapper.buildTestContext();
    registerTestExecutionListeners(testContextBootstrapper.getTestExecutionListeners());
}
```

# 默认的 TestContextBootstrapper 实现

在TestContextManager的构造过程中调用了这个方法.
主要调用 BuildMergedContextConfiguration() 方法生成对应的 Context 配置.

```java
public TestContext buildTestContext() {
    return new DefaultTestContext(getBootstrapContext().getTestClass(), buildMergedContextConfiguration(),
            getCacheAwareContextLoaderDelegate());
}
```

```java

public final MergedContextConfiguration buildMergedContextConfiguration() {
    Class<?> testClass = getBootstrapContext().getTestClass();
    CacheAwareContextLoaderDelegate cacheAwareContextLoaderDelegate = getCacheAwareContextLoaderDelegate();

    // 使用工具查找测试类上的 @ContextConfiguration 和 @ContextHierarchy 注解.
    // 如果没有找到就说明没有显式指定使用的配置文件,则使用默认配置.
    if (MetaAnnotationUtils.findAnnotationDescriptorForTypes(
            testClass, ContextConfiguration.class, ContextHierarchy.class) == null) {
        return buildDefaultMergedContextConfiguration(testClass, cacheAwareContextLoaderDelegate);
    }

    // 如果找到了 @ContextHierarchy 注解.
    if (AnnotationUtils.findAnnotation(testClass, ContextHierarchy.class) != null) {
        Map<String, List<ContextConfigurationAttributes>> hierarchyMap =
                ContextLoaderUtils.buildContextHierarchyMap(testClass);
        MergedContextConfiguration parentConfig = null;
        MergedContextConfiguration mergedConfig = null;

        for (List<ContextConfigurationAttributes> list : hierarchyMap.values()) {
            List<ContextConfigurationAttributes> reversedList = new ArrayList<>(list);
            Collections.reverse(reversedList);

            // Don't use the supplied testClass; instead ensure that we are
            // building the MCC for the actual test class that declared the
            // configuration for the current level in the context hierarchy.
            Assert.notEmpty(reversedList, "ContextConfigurationAttributes list must not be empty");
            Class<?> declaringClass = reversedList.get(0).getDeclaringClass();

            mergedConfig = buildMergedContextConfiguration(
                    declaringClass, reversedList, parentConfig, cacheAwareContextLoaderDelegate, true);
            parentConfig = mergedConfig;
        }

        // Return the last level in the context hierarchy
        Assert.state(mergedConfig != null, "No merged context configuration");
        return mergedConfig;
    }
    else {
        return buildMergedContextConfiguration(testClass,
                ContextLoaderUtils.resolveContextConfigurationAttributes(testClass),
                null, cacheAwareContextLoaderDelegate, true);
    }
}
```
