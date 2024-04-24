# AnnotationConfigApplciationContext

```java
// 重点就是这个 register() 方法, refresh() 就是Spring最标准的启动方法.
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    this();
    register(annotatedClasses);
    refresh();
}

public void register(Class<?>... annotatedClasses) {
    this.reader.register(annotatedClasses);
}
```

## AnnotatedBeanDefinitionReader

```java
public void register(Class<?>... annotatedClasses) {
    for (Class<?> annotatedClass : annotatedClasses) {
        registerBean(annotatedClass);
    }
}
public void registerBean(Class<?> annotatedClass) {
    doRegisterBean(annotatedClass, null, null, null);
}

// 所有的注册最终都会到这个方法.
// annotatedClass : 传入的配置类, Supplier : 用来生成这个实例的工厂 可以为空,
// name : 显式指定的 beanName, 如果没有会调用 BeanNameGenerator 来生成一个默认名字.
// definitionCustomizers: 一个自定义处理生成 BeanDefinition 的方法数组. 用来修改生成的 BeanDefinition, 可以为空
<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
        @Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {

    // 这里直接创建一个 BeanDefinition. 下一步通过 conditionEvaluator 来验证 @Condition 注解, 这就是Spring提供的条件Bean功能的实现.
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    // 设置 BeanDefinition, 这个 Supplier 本来就在BeanDefinition接口中定义了.
    abd.setInstanceSupplier(instanceSupplier);
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

    // 一个方便的工具来读取 AnnotatedGenericBeanDefinition 中的 AnnotatedTypeMetadata.
    // 主要读取注解信息中的, @Lazy, @Primary, @DependsOn, @Role, @Description
    // 读取注解信息, 并将注解上的信息放到 BeanDefinition 中对应的属性.
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    if (qualifiers != null) {
        for (Class<? extends Annotation> qualifier : qualifiers) {
            if (Primary.class == qualifier) {
                abd.setPrimary(true);
            }
            else if (Lazy.class == qualifier) {
                abd.setLazyInit(true);
            }
            else {
                abd.addQualifier(new AutowireCandidateQualifier(qualifier));
            }
        }
    }
    // 到上面循环结束为止, Spring 标志的必须的BeanDefinition就生成了
    // 下面是应用传入的 BeanDefinitionCustomizer 修改生成的 BeanDefinition.
    for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
        customizer.customize(abd);
    }

    // 最后注册BeanDefinition到 Registry 中.
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

