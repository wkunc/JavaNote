# BeanDefinition
![](imgs/BeanDefinition.PNG)
A BeanDefinition describes a bean instance, which has property values,
constructor argument values, and further information supplied by
concrete implementations.

一个 BeanDefinition 描述了一个 bean 实例, 它具有属性值, 构造参数值,以及更多的细节

这些是属性的 get/set 方法
```java
// 父定义如果有的话
void setParentName(String parentName);
String getParentName();

// 指定 bean 的 class 类名, 可以在 bean factory post-prcessing阶段被修改
void setBeanClassName(String beanClassName);
String getBeanClassName();

// override the target scope of this bean, specifying a new scope name
void setScope(String scope);
String getScope();

// 设置这个bean是否应该lazy加载
void setLazyInit(boolean lazyInit);
boolean isLazyInit();

//设置 bean 依赖的bean ,bean factory 会保证那些类会先被初始化
void setDependsOn(String... dependsOn);
String[] getDependsOn(String... dependsOn);

void setAutowireCandidate();
boolean isAutowireCandidate();

void setPrimary();
boolean isPrimary();

void setFactoryBeanName();
String getFactoryBeanName();

void setFactoryMethodName();
String getFactoryMethodName();

这些是 get 方法
ConstructorArgumentValues getConstructorArgumentValues();
MutablePropertyValues getPropertyValues();

// Read-only attributes 只读属性

boolean isSingleton();
boolean isPrototype();
boolean isAbstract();
int getRole();

String getDescription();
String getResourceDescription();
BeanDefinition getOriginatingBeanDefinition();
```

# BeanWrapper
```java
public interface PropertyEditor {
    void setValue(Object value);
    Object getValue();
    boolean isPaintable();
    void paintValue(java.awt.Graphics gfx, java.awt.Rectangle box);
    String getJavaInitializationString();
    String getAsText();
    void setAsText(String text) throws java.lang.IllegalArgumentException;
    String[] getTags();
    java.awt.Component getCustomEditor();
    boolean supportsCustomEditor();
    void addPropertyChangeListener(PropertyChangeListener listener);
    void removePropertyChangeListener(PropertyChangeListener listener);
}
```

# BeanDefinitionReader
![](imgs/BeanDefinitionReader)
Simple interface for bean definition readers.

```java
public interface BeanDefinitionReader {
    //
    BeanDefinitionRegistry getRegistry();
    ResourceLoader getResouceLoader();
    ClassLoader getBeanClassLoader();
    BeanNameGenerator getBeanNameGenerator();
    // 核心方法
    int loadBeanDefinitions(Resource resource);
    int loadBeanDefinitions(Resource... resources);
    int loadBeanDefinitions(String location);
    int loadBeanDefinitions(String... locations);
}
```

## AbstractBeanDefinitionReader
实现了 BeanDefinitionReader 接口. 有一个方法没有实现

```java
public loadBeanDefinitions(Resouce resourc);
```
其他load方法底层都依赖于它, 然后把它交给子类实现 (ps:模板方法设计模式)

其他方法的实现都很简单, 只是简单的持有一个相应的类对象
然后一些 setter/getter 方法


