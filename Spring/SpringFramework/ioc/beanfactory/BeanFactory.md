# BeanFactory
![](DefaultListableBeanFactory.PNG)
它是 Spring bean container 的 root interface 

它主要定义了一系列的 getBean() 方法
```java
public interface BeanFactory {
    //通过指定的beanName获取指定的 Bean
    //在 AbstractBeanFactory 中实现
    Object getBean(String name) throws BeansException;

    //指定名字和类型来获取 Bean, requireType 可以是其父类或接口, 甚至可以是 null
    //在 AbstractBeanFactory 中实现
    <T> T getBean(String name, Class<T> requireType) throws BeansException;

    //用指定的参数来构建指定的 Bean, 这些参数可以是构造器参数/工厂方法参数,用来覆盖默认的参数
    //在 AbstractBeanFactory 中实现
    // 2.5版本加入
    Object getBean(String name, Object... args) throws BeansException;

    //返回唯一匹配的对象类型, 不可以为 null.
    //在 DefaultListableBeanFactory 中实现
    // 3.0版本加入
    <T> T getBean(Class<T> requiredType) throws BeansException;

    //和上面的一样,用新的参数去构造一个新的指定对象
    //在 DefaultListableBeanFactory 中实现
    // 4.1版本加入
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

    // 5.1 新加的两个方法
	<T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);
	<T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

    // 在 AbstractBeanFactory 中实现
    boolean containsBean(String name);
    boolean isSingleton(String name);
    boolean isProtorype(String name);

    //返回给定的beanName 和 类型是否匹配. ResolvableType 封装了 java.lang.reflect.Type 类
    //在 AbstractBeanFactory 中实现
    boolean isTypeMatch(String name, ResolvableType typeToMatch)
    boolean isTypeMatch(String name, Class<?> typeToMatch)

    // 在 AbstractBeanFactory 中实现.
    Class<?> getType(String)
    String[] getAliases(String)
}
```

# ListableBeanFactory
定义了访问容器中 Bean 基本信息的若干方法
```java
//是否包含指定名字的 BeanDefinition
boolean containsBeanDefinition(String beanName);

//获取包含的 BeanDefinition 的个数
int getBeanDefinitiionCount();

//获取所有的 BeanNames
String[] getBeanDefinitionNames();

String[] getBeanNamesForType(ResolvableType type);
String[] getBeanNamesForType(Class<?> type);
String[] getBeanNamesForType(Class<?> type, boolean includeNonSingletions, boolean allowEagerInit);

<T> Map<String, T> getBeansOfType(Class<T> type) throws BeansException;
<T> Map<String, T> getBeansOfType(Class<T> type, boolean includeNonSingletions, boolean allowEagerInit);

String[] getBeanNamesForAnnotation(Class<? extends Annotation> annotationType);
Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType);
<A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType);
```

# HierarchicalBeanFactory
定义了父子级联的 Ioc 容器接口
仅仅包含了两个方法
```java
public interface HierarchicalBeanFactory extends BeanFactory{
    //返回父容器, 如果没有返回 null
    BeanFactory getParentBeanFactory();
    // 判断当前 BeanFactory 中是否有指定名字的 Bean ,忽略父容器中的
    boolean containsLocalBean(String name);
}
```

# ConfigurableBeanFactroy
这是一个重量级的接口, 它增强了 IoC 容器的可定制性(ps:可配置的BeanFactroy嘛)
这个 BeanFactory 接口并不适合用于普通的应用程序, 
这个扩展接口只是为了允许框架内部的即插即用和对 BeanFactory 配置方法的特殊访问.

它定义了设置类装载器, 属性编辑器, 容器初始化后置处理器 等方法

1. ParentBeanFactory BeanFactory // 父接口中来的
2. BeanClassLoader 加载Bean的ClassLoader
3. TempClassLoader ClassLoader
4. CacheBeanMetadata boolean (ps:是否缓存元数据)
5. BeanExpressionResolver(ps:好像和 EL表达式有关)
6. ConversionService(类型转换服务)
7. PropertyEditorRegistrar(属性编辑器)
8. TyepConverter(类型转换)
9. EmbeddedValueResolver
10. BeanPostProcessor
```java
public interface ConfigurableBeanFactory extends HierarchiaclBeanFactory, SingletonBeanRegistry {

	String SCOPE_SINGLETON = "singleton";

	String SCOPE_PROTOTYPE = "prototype";

	/**
     * 设置父容器, 这个父亲不能被改变, getParentBeanFactory() 方法定义在父接口 HierarchiaclBeanFactory中
	 */
	void setParentBeanFactory(BeanFactory parentBeanFactory) throws IllegalStateException;

	/**
     * ClassLoader 的getter setter
	 * 设置一个用来加载 Bean 的类加载器,默认使用当前线程的类加载器
	 */
	void setBeanClassLoader(ClassLoader beanClassLoader);
	ClassLoader getBeanClassLoader();

	/**
     * 指定一个临时类加载器, 默认是none.
     * 通常只有发生 load-time weaving 时才特殊指定 
     * 临时类加载器会在 BeanFactory 完全启动后被移除
	 */
	void setTempClassLoader(ClassLoader tempClassLoader);
	ClassLoader getTempClassLoader();

	/**
     * 设置是否缓存 Bean 的元数据, 默认开启.
     * 关闭该设置以开启 Bean definition 的热刷新.
     * 如果这个 flag 为 off, 每次创建Bean实例都会重新询问BeanClassLoader
	 */
	void setCacheBeanMetadata(boolean cacheBeanMetadata);

	boolean isCacheBeanMetadata();

	/**
     * 设置Bean定义中表达式的解析策略, BeanFactory 中没有默认的
     * ApplicationContext 一般会设置 standard expression
	 */
	void setBeanExpressionResolver(BeanExpressionResolver resolver);

	BeanExpressionResolver getBeanExpressionResolver();

	/**
     * 设置一个转换服务, 用于转换属性值.
     * 从Spring3.0开始用于替代 JavaBeans PropertyEditor
	 */
	void setConversionService(ConversionService conversionService);

	ConversionService getConversionService();

	/**
     * 属性编辑器注册员, 利用这个对象可以应用于所有bean创建过程, 每次使用对应属性编辑器都是新的实例
     * 避免了在自定义编辑器上进行同步的需要.
     * 通常最好使用此方法而不是registerCustormEditor.
	 */
	void addPropertyEditorRegistrar(PropertyEditorRegistrar registrar);

    // 为给定类型注册自己的自定义属性编辑器. 为了线程安全自定义的属性编辑器需要线程同步.
	void registerCustomEditor(Class<?> requiredType, Class<? extends PropertyEditor> propertyEditorClass);

	/**
     * 用已在此BeanFactory中注册的自定义编辑器初始化给定的 PropertyEditorRegistry.
	 */
	void copyRegisteredEditorsTo(PropertyEditorRegistry registry);

	/**
     * 设置应用于转换bean属性值, 构造器参数值等的自定义类型转换器.
     * 这将覆盖默认的 PropertyEditor 机制. 因此上面注册的自定义属性编辑器都变得无关紧要.
	 */
	void setTypeConverter(TypeConverter typeConverter);

	TypeConverter getTypeConverter();

	/**
	 * 添加一个 String 解析器, 用于解析 embedded 值, 如注解属性.
	 */
	void addEmbeddedValueResolver(StringValueResolver valueResolver);

    // 确定是否有 EmdeddedValueResolver 当前BeanFactory.
    // 用于在解决嵌入字符串时先判断是否有对应的工具
	boolean hasEmbeddedValueResolver();

	/**
     * 解析对应的emdedded value
	 */
	String resolveEmbeddedValue(String value);

	/**
     * 添加一个 BeanPostProcessor 它将应用于此工厂创建的bean.
     * 这个方法应该在工厂配置期间调用.
     * 注意: 通过这个方法注册的 BeanPostProcessor 将按注册顺序执行.
     * 通过 Ordered 接口表示的排序语义都将被忽略.
     * 所有自动检测的 post-processor 都将在编程注册的 post-processor 调用之后才调用.
	 */
	void addBeanPostProcessor(BeanPostProcessor beanPostProcessor);

	int getBeanPostProcessorCount();


    // 和Spring的自定义 Socpe 功能有关, 显式的注册自定义的 Socpe.
	void registerScope(String scopeName, Scope scope);
    // 获取注册的自定义的Socpe, 内建的 singleton 和 prototype 不会被获取到.
	String[] getRegisteredScopeNames();
	Scope getRegisteredScope(String scopeName);

    // 或者访问控制上下文, 永远不会为null.
	AccessControlContext getAccessControlContext();

	/**
     * 从给定的工厂中拷贝相关设置, 应该包括所有标准设置以及 BeanPostProcessors, Scopes 和工厂内部设置
     * 不包括实际 BeanDefintion对象和 bean aliase
	 */
	void copyConfigurationFrom(ConfigurableBeanFactory otherFactory);

	/**
     * 注册别名, 被 SimpleAliasRegistry 这个支持类实现了
	 */
	void registerAlias(String beanName, String alias) throws BeanDefinitionStoreException;

	/**
     * 被 SimpleAliasRegistry 这个支持类实现了
	 */
	void resolveAliases(StringValueResolver valueResolver);

	/**
     * 返回给定 bean name 对应的 MergedBeanDefintion, 如果有需要, 将 child beanDefinition 和 它的父定义合并.
     * 也会考虑 parentBeanFactory 中的定义.
	 */
	BeanDefinition getMergedBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

	boolean isFactoryBean(String name) throws NoSuchBeanDefinitionException;
	void setCurrentlyInCreation(String beanName, boolean inCreation);
	boolean isCurrentlyInCreation(String beanName);

    // 定义在这个接口中, 却被 SingletonBeanRegistry 中被基本实现了.
	void registerDependentBean(String beanName, String dependentBeanName);
	String[] getDependentBeans(String beanName);
	String[] getDependenciesForBean(String beanName);

	void destroyBean(String beanName, Object beanInstance);
	void destroyScopedBean(String beanName);
	void destroySingletons();
}
```

# AutowireCapableBeanFactory
自动装配的 BeanFacroty

```java
// 创建并填充外部bean的典型方法

<T> T createBean(Class<T> beanClass)
void autowireBean(Object existingBean)
Object configureBean(Object existingBean, String beanName)


// 用于细粒度控制bean生命周期的专用方法
Object creaetBean(Class<?> beanClass, int autowireMode, boolean dependencyCheck)
Object autowire(Class<?> beanClass, int autowireMode, boolean dependencyCheck)
void autowireBeanProperties(Object existingBean, int autowireMode, boolean dependencyCheck)
void applyBeanPropertyValues(Object existingBean, String beanName)
Object initializeBean(Object existingBean, String beanName)
Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
void destroyBean(Object existingBean)

// 委派解决注入点的方法
<T> NamedBeanHolder<T> resovleNamedBean(Class<T> requiredType)
Object resolveDependency(DependencyDescriptory descriptor, String requresingBeanName)
Object resolveDependency(DependencyDescriptory descriptor, String requresingBeanName,
                        Set<String> autowiredBeanNames, TypeConverter typeConverter)
```
