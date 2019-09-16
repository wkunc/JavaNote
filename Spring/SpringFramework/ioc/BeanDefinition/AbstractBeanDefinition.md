# AbstractBeanDefinition
```java
private volatile Object beanClass;
private String scope = SCOPE_DEFAULT;
private boolean abstractFlag = false;
private boolean lazyInit = false;
private String[] dependsOn;
private boolean autowireCandidate = true; 
private boolean primary = false;
private String factoryBeanName;
private String factoryMethodName;
private ConstructorArgumentValues constructorArgumentValues;
private MutablePropertyValues propertyValues;
private String initMethodName;
private String destroy MethoddName;
private int role = BeanDefinition.ROLE_APPLICTION;
private String description;
private Resource resource;
// 上面的是实现BeanDefinition接口必须定义的属性
// 还缺一个 parentName, 交给子类实现.


private int autowireMode = AUTOWIRE_NO;
private int dependencyCheck = DEPENDENCY_CHECK_NONE;
private final Map<String, AutowireCandidateQualifier> qualifiers = new LinkedHashMap<>();
private Supplier<?> instanceSupplier;
private boolean noPublicAccessAllowed = true;
private boolean lenientConstructorResolution = true;
private MethodOverrides methodOverrides;
private boolean enforrceInitMethod = true;
private boolean enforceDestroyMethod = true;
private boolean synthetic =false;
```

# 构造器
```java
protected AbstractBeanDefinition() {
    this(null, null);
}
protected AbstractBeanDefinition(ConstructorArgumentValues cargs, MutablePropertyValues pvs) {
    this.constructorArgumentValues = cargs;
    this.propertyValues = pvs;
}
protected AbstractBeanDefinition(BeanDefinition original) {
    setParentName(original.getParentName());
    setBeanClassName(original.getBeanClassName());
    setScope(original.getScope());
    setAbstract(original.isAbstract());
    setLazyInit(original.isLazyInit());
    setFactoryBeanName(original.getFactoryBeanName());
    setFactoryMethodName(original.getFactoryMethodName());
    setRole(original.getRole());
    setSource(original.getSource());
    // 
    copyAttributesFrom(original);

    if (original instanceof AbstractBeanDefinition) {
        AbstractBeanDefinition originalAbd = (AbstractBeanDefinition) original;
        if (originalAbd.hasBeanClass()) {
            setBeanClass(originalAbd.getBeanClass());
        }
        if (originalAbd.hasConstructorArgumentValues()) {
            setConstructorArgumentValues(new ConstructorArgumentValues(original.getConstructorArgumentValues()));
        }
        if (originalAbd.hasPropertyValues()) {
            setPropertyValues(new MutablePropertyValues(original.getPropertyValues()));
        }
        if (originalAbd.hasMethodOverrides()) {
            setMethodOverrides(new MethodOverrides(originalAbd.getMethodOverrides()));
        }
        setAutowireMode(originalAbd.getAutowireMode());
        setDependencyCheck(originalAbd.getDependencyCheck());
        setDependsOn(originalAbd.getDependsOn());
        setAutowireCandidate(originalAbd.isAutowireCandidate());
        setPrimary(originalAbd.isPrimary());
        //
        copyQualifiersFrom(originalAbd);
        setInstanceSupplier(originalAbd.getInstanceSupplier());
        setNonPublicAccessAllowed(originalAbd.isNonPublicAccessAllowed());
        setLenientConstructorResolution(originalAbd.isLenientConstructorResolution());
        setInitMethodName(originalAbd.getInitMethodName());
        setEnforceInitMethod(originalAbd.isEnforceInitMethod());
        setDestroyMethodName(originalAbd.getDestroyMethodName());
        setEnforceDestroyMethod(originalAbd.isEnforceDestroyMethod());
        setSynthetic(originalAbd.isSynthetic());
        setResource(originalAbd.getResource());
    }
    else {
        setConstructorArgumentValues(new ConstructorArgumentValues(original.getConstructorArgumentValues()));
        setPropertyValues(new MutablePropertyValues(original.getPropertyValues()));
        setResourceDescription(original.getResourceDescription());
    }
}
```

# 具体子类
## RootBeanDefinition
代表最该高层的BeanDefintion, 不能拥有 parent, 调用 setParentName(beanName) 会报错

## ChildBeanDefinition
代表儿子BeanDefintion, 一定有一个 parent, 因为它的每个构造器都有 parentName 这个参数.

## GenericBeanDefinition
可以有 parent 也可以没有, 最通用的 BeanDefintion, 如果需要手动注册 BeanDefintion 使用这个子类是不错的选择.
