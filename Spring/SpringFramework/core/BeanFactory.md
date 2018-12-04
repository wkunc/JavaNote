# BeanFactory
![](imgs/BeanFactory.PNG)
它是 Spring bean container 的 root interface 

它主要定义了一系列的 getBean() 方法
```java
public interface BeanFactory {
    //通过指定的beanName获取指定的 Bean
    Object getBean(String name) throws BeansException;

    //指定名字和类型来获取 Bean, requireType 可以是其父类或接口, 甚至可以是 null
    <T> T getBean(String name, Class<T> requireType) throws BeansException;

    //用指定的参数来构建指定的 Bean, 这些参数可以是构造器参数/工厂方法参数,用来覆盖默认的参数
    Object getBean(String name, Object... args) throws BeansException;

    //返回唯一匹配的对象类型, 不可以为 null.
    //这个方法在 ListableBeanFacroty 中通过类型查找
    <T> T getBean(Class<T> requiredType) throws BeansException;

    //和上面的一样,用新的参数去构造一个新的指定对象
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

    boolean containsBean(String name);
    boolean isSingleton(String name);
    boolean isProtorype(String name);

    //返回给定的beanName 和 类型是否匹配
    //ResolvableType 封装了 java.lang.reflect.Type 类
    boolean isTypeMatch(String name, ResolvableType typeToMatch)
    boolean isTypeMatch(String name, Class<?> typeToMatch)

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

它定义了设置类装载器, 属性编辑器, 容器初始化后置处理器 等方法

1. ParentBeanFactory BeanFactory
2. BeanClassLoader 加载Bean的ClassLoader
3. TempClassLoader ClassLoader
4. CacheBeanMetadata boolean (ps:是否缓存元数据)
5. BeanExpressionResolver(ps:好像和 EL表达式有关)
6. ConversionService(类型转换服务)
7. PropertyEditorRegistrar(属性编辑器)
8. TyepConverter(类型转换)
9. EmbeddedValueResolver
9. BeanPostProcessor
```java
public interface ConfigurableBeanFactory extends HierarchiaclBeanFactory, SingletonBeanRegistry {
	/**
	 * Scope identifier for the standard singleton scope: "singleton".
	 * Custom scopes can be added via {@code registerScope}.
	 * @see #registerScope
	 */
	String SCOPE_SINGLETON = "singleton";

	/**
	 * Scope identifier for the standard prototype scope: "prototype".
	 * Custom scopes can be added via {@code registerScope}.
	 * @see #registerScope
	 */
	String SCOPE_PROTOTYPE = "prototype";


	/**
     * 设置父容器, 这个父亲不能被改变
	 * <p>Note that the parent cannot be changed: It should only be set outside
	 * a constructor if it isn't available at the time of factory instantiation.
	 */
	void setParentBeanFactory(BeanFactory parentBeanFactory) throws IllegalStateException;

	/**
	 * 设置一个用来加载 Bean 的类加载器,默认使用当前线程的类加载器
	 * <p>Note that this class loader will only apply to bean definitions
	 * that do not carry a resolved bean class yet. This is the case as of
	 * Spring 2.0 by default: Bean definitions only carry bean class names,
	 * to be resolved once the factory processes the bean definition.
	 */
	void setBeanClassLoader(ClassLoader beanClassLoader);

	ClassLoader getBeanClassLoader();

	/**
	 * Specify a temporary ClassLoader to use for type matching purposes.
	 * Default is none, simply using the standard bean ClassLoader.
	 * <p>A temporary ClassLoader is usually just specified if
	 * <i>load-time weaving</i> is involved, to make sure that actual bean
	 * classes are loaded as lazily as possible. The temporary loader is
	 * then removed once the BeanFactory completes its bootstrap phase.
	 * @since 2.5
	 */
	void setTempClassLoader(ClassLoader tempClassLoader);

	ClassLoader getTempClassLoader();

	/**
	 * Set whether to cache bean metadata such as given bean definitions
	 * (in merged fashion) and resolved bean classes. Default is on.
	 * <p>Turn this flag off to enable hot-refreshing of bean definition objects
	 * and in particular bean classes. If this flag is off, any creation of a bean
	 * instance will re-query the bean class loader for newly resolved classes.
	 */
	void setCacheBeanMetadata(boolean cacheBeanMetadata);

	boolean isCacheBeanMetadata();

	/**
	 * Specify the resolution strategy for expressions in bean definition values.
	 * <p>There is no expression support active in a BeanFactory by default.
	 * An ApplicationContext will typically set a standard expression strategy
	 * here, supporting "#{...}" expressions in a Unified EL compatible style.
	 * @since 3.0
	 */
	void setBeanExpressionResolver(BeanExpressionResolver resolver);

	BeanExpressionResolver getBeanExpressionResolver();

	/**
     * 设置一个转换服务, 用于转换属性值
	 * Specify a Spring 3.0 ConversionService to use for converting
	 * property values, as an alternative to JavaBeans PropertyEditors.
	 * @since 3.0
	 */
	void setConversionService(ConversionService conversionService);

	ConversionService getConversionService();

	/**
	 * Add a PropertyEditorRegistrar to be applied to all bean creation processes.
	 * <p>Such a registrar creates new PropertyEditor instances and registers them
	 * on the given registry, fresh for each bean creation attempt. This avoids
	 * the need for synchronization on custom editors; hence, it is generally
	 * preferable to use this method instead of {@link #registerCustomEditor}.
	 */
	void addPropertyEditorRegistrar(PropertyEditorRegistrar registrar);

	/**
	 * Register the given custom property editor for all properties of the
	 * given type. To be invoked during factory configuration.
	 * <p>Note that this method will register a shared custom editor instance;
	 * access to that instance will be synchronized for thread-safety. It is
	 * generally preferable to use {@link #addPropertyEditorRegistrar} instead
	 * of this method, to avoid for the need for synchronization on custom editors.
	 * @param requiredType type of the property
	 * @param propertyEditorClass the {@link PropertyEditor} class to register
	 */
	void registerCustomEditor(Class<?> requiredType, Class<? extends PropertyEditor> propertyEditorClass);

	/**
	 * Initialize the given PropertyEditorRegistry with the custom editors
	 * that have been registered with this BeanFactory.
	 * @param registry the PropertyEditorRegistry to initialize
	 */
	void copyRegisteredEditorsTo(PropertyEditorRegistry registry);

	/**
	 * Set a custom type converter that this BeanFactory should use for converting
	 * bean property values, constructor argument values, etc.
	 * <p>This will override the default PropertyEditor mechanism(机制) and hence(于是) make
	 * any custom editors or custom editor registrars irrelevant(不相关).
	 */
	void setTypeConverter(TypeConverter typeConverter);

	/**
	 * Obtain a type converter as used by this BeanFactory. This may be a fresh
	 * instance for each call, since TypeConverters are usually <i>not</i> thread-safe.
	 * <p>If the default PropertyEditor mechanism is active, the returned
	 * TypeConverter will be aware of all custom editors that have been registered.
	 */
	TypeConverter getTypeConverter();

	/**
	 * Add a String resolver for embedded values such as annotation attributes.
	 */
	void addEmbeddedValueResolver(StringValueResolver valueResolver);

	/**
	 * Determine whether an embedded value resolver has been registered with this
	 * bean factory, to be applied through {@link #resolveEmbeddedValue(String)}.
	 */
	boolean hasEmbeddedValueResolver();

	/**
	 * Resolve the given embedded value, e.g. an annotation attribute.
	 */
	String resolveEmbeddedValue(String value);

	/**
	 * Add a new BeanPostProcessor that will get applied to beans created
	 * by this factory. To be invoked during factory configuration.
	 * <p>Note: Post-processors submitted here will be applied in the order of
	 * registration; any ordering semantics expressed through implementing the
	 * {@link org.springframework.core.Ordered} interface will be ignored. Note
	 * that autodetected post-processors (e.g. as beans in an ApplicationContext)
	 * will always be applied after programmatically registered ones.
	 */
	void addBeanPostProcessor(BeanPostProcessor beanPostProcessor);

	int getBeanPostProcessorCount();

	/**
	 * Register the given scope, backed by the given Scope implementation.
	 * @param scopeName the scope identifier
	 * @param scope the backing Scope implementation
	 */
	void registerScope(String scopeName, Scope scope);

	/**
	 * Return the names of all currently registered scopes.
	 * <p>This will only return the names of explicitly registered scopes.
	 * Built-in scopes such as "singleton" and "prototype" won't be exposed.
	 */
	String[] getRegisteredScopeNames();

	/**
	 * Return the Scope implementation for the given scope name, if any.
	 * <p>This will only return explicitly registered scopes.
	 * Built-in scopes such as "singleton" and "prototype" won't be exposed.
	 */
	Scope getRegisteredScope(String scopeName);

	/**
	 * Provides a security access control context relevant to this factory.
	 * @return the applicable AccessControlContext (never {@code null})
	 * @since 3.0
	 */
	AccessControlContext getAccessControlContext();

	/**
	 * Copy all relevant configuration from the given other factory.
	 * <p>Should include all standard configuration settings as well as
	 * BeanPostProcessors, Scopes, and factory-specific internal settings.
	 * Should not include any metadata of actual bean definitions,
	 * such as BeanDefinition objects and bean name aliases.
	 * @param otherFactory the other BeanFactory to copy from
	 */
	void copyConfigurationFrom(ConfigurableBeanFactory otherFactory);

	/**
	 * Given a bean name, create an alias. We typically use this method to
	 * support names that are illegal within XML ids (used for bean names).
	 * <p>Typically invoked during factory configuration, but can also be
	 * used for runtime registration of aliases. Therefore, a factory
	 * implementation should synchronize alias access.
	 * @param beanName the canonical name of the target bean
	 * @param alias the alias to be registered for the bean
	 * @throws BeanDefinitionStoreException if the alias is already in use
	 */
	void registerAlias(String beanName, String alias) throws BeanDefinitionStoreException;

	/**
	 * Resolve all alias target names and aliases registered in this
	 * factory, applying the given StringValueResolver to them.
	 * <p>The value resolver may for example resolve placeholders
	 * in target bean names and even in alias names.
	 * @param valueResolver the StringValueResolver to apply
	 * @since 2.5
	 */
	void resolveAliases(StringValueResolver valueResolver);

	/**
	 * Return a merged BeanDefinition for the given bean name,
	 * merging a child bean definition with its parent if necessary.
	 * Considers bean definitions in ancestor factories as well.
	 * @param beanName the name of the bean to retrieve the merged definition for
	 * @return a (potentially merged) BeanDefinition for the given bean
	 * @throws NoSuchBeanDefinitionException if there is no bean definition with the given name
	 * @since 2.5
	 */
	BeanDefinition getMergedBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

	/**
	 * Determine whether the bean with the given name is a FactoryBean.
	 * @param name the name of the bean to check
	 * @return whether the bean is a FactoryBean
	 * ({@code false} means the bean exists but is not a FactoryBean)
	 * @throws NoSuchBeanDefinitionException if there is no bean with the given name
	 * @since 2.5
	 */
	boolean isFactoryBean(String name) throws NoSuchBeanDefinitionException;

	/**
	 * Explicitly control the current in-creation status of the specified bean.
	 * For container-internal use only.
	 * @param beanName the name of the bean
	 * @param inCreation whether the bean is currently in creation
	 * @since 3.1
	 */
	void setCurrentlyInCreation(String beanName, boolean inCreation);

	/**
	 * Determine whether the specified bean is currently in creation.
	 * @param beanName the name of the bean
	 * @return whether the bean is currently in creation
	 * @since 2.5
	 */
	boolean isCurrentlyInCreation(String beanName);

	/**
	 * Register a dependent bean for the given bean,
	 * to be destroyed before the given bean is destroyed.
	 * @param beanName the name of the bean
	 * @param dependentBeanName the name of the dependent bean
	 * @since 2.5
	 */
	void registerDependentBean(String beanName, String dependentBeanName);

	/**
	 * Return the names of all beans which depend on the specified bean, if any.
	 * @param beanName the name of the bean
	 * @return the array of dependent bean names, or an empty array if none
	 * @since 2.5
	 */
	String[] getDependentBeans(String beanName);

	/**
	 * Return the names of all beans that the specified bean depends on, if any.
	 * @param beanName the name of the bean
	 * @return the array of names of beans which the bean depends on,
	 * or an empty array if none
	 * @since 2.5
	 */
	String[] getDependenciesForBean(String beanName);

	/**
	 * Destroy the given bean instance (usually a prototype instance
	 * obtained from this factory) according to its bean definition.
	 * <p>Any exception that arises during destruction should be caught
	 * and logged instead of propagated to the caller of this method.
	 * @param beanName the name of the bean definition
	 * @param beanInstance the bean instance to destroy
	 */
	void destroyBean(String beanName, Object beanInstance);

	/**
	 * Destroy the specified scoped bean in the current target scope, if any.
	 * <p>Any exception that arises during destruction should be caught
	 * and logged instead of propagated to the caller of this method.
	 * @param beanName the name of the scoped bean
	 */
	void destroyScopedBean(String beanName);

	/**
	 * Destroy all singleton beans in this factory, including inner beans that have
	 * been registered as disposable. To be called on shutdown of a factory.
	 * <p>Any exception that arises during destruction should be caught
	 * and logged instead of propagated to the caller of this method.
	 */
	void destroySingletons();
}
```

# AutowireCapableBeanFactory
自动装配的 BeanFacroty

```java
<T> T createBean(Class<T> beanClass)
void autowireBean(Object existingBean)
Object configureBean(Object existingBean, String beanName)
Object creaetBean(Class<?> beanClass, int autowireMode, boolean dependencyCheck)
Object autowire(Class<?> beanClass, int autowireMode, boolean dependencyCheck)
void autowireBeanProperties(Object existingBean, int autowireMode, boolean dependencyCheck)
void applyBeanPropertyValues(Object existingBean, String beanName)
Object initializeBean(Object existingBean, String beanName)
Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
void destroyBean(Object existingBean)
<T> NamedBeanHolder<T> resovleNamedBean(Class<T> requiredType)
Object resolveDependency(DependencyDescriptory descriptor, String requresingBeanName)
Object resolveDependency(DependencyDescriptory descriptor, String requresingBeanName,
                        Set<String> autowiredBeanNames, TypeConverter typeConverter)
```
