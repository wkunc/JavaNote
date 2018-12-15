# The IoC Container
org.springframework.beans 和 org.springframework.context 包是 Spring IoC 容器的基础

BeanFactory 接口提供了一种能够管理任何对象的高级配置机制

ApplicationContext 是 BeanFactory 的子接口 它补充了 "更容易集成 Spring'AOP, 
消息资源处理(用于国际化),事件发布, 特定于应用程序的上下文

简而言之, BeanFactory 提供了配置框架和基本功能而 ApplicationContext 添加了更多的特定于企业的功能

ApplicationContext 是 BeanFactory 的完整超集

# Container Overview(大纲)

## Configuration Metadata(元数据)
Configuration metadata 代表你如何告诉Spring 容器在应用程序中 instantiate(实例化), 
configure (配置) 和 assemble(组装对象)

Spring IoC container 本身完全与与实际编写此配置元数据的格式分离

# Bean Overview

## Instantiating Beans
A bean definition is essentially a recipe for creating one or more object

### Instantiation with a Constructor

### Instantiation with a Static Factory Method

```
<bean id = "clientService" class="examples.ClientService" factory-method="createInstance"/>
```
### Instantiation by Using an Instance Factory Method
和静态工厂方法类似, 使用实例工厂方法进行实例化会从容器调用现有 bean 的实例方法来创建新的 bean
```xml
<!-- the factory bean, which contains a method called createInstance() -->
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
    <!-- inject any dependencies required by this locator bean -->
</bean>

<!-- the bean to be created via the factory bean -->
<bean id="clientService"
    factory-bean="serviceLocator"
    factory-method="createClientServiceInstance"/>
```

# Dependencies
## Dependency Injection(DI)
### 构造器注入
```
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg type="int" value="7500000"/>
    <constructor-arg type="java.lang.String" value="42"/>
</bean>
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg index="0" value="7500000"/>
    <constructor-arg index="1" value="42"/>
</bean>
<!--有特殊要求的 -->
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg name="years" value="7500000"/>
    <constructor-arg name="ultimateAnswer" value="42"/>
</bean>
```
@ConstructorProperties
### 属性注入

### Dependency Resolution Process(依赖 解决 过程)

## Dependencies and Configuration in Detail
You can also configure a java.util.Properties instance, as follows:
```xml
<bean id="mappings"
    class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">

    <!-- typed as a java.util.Properties -->
    <property name="properties">
        <value>
            jdbc.driver.className=com.mysql.jdbc.Driver
            jdbc.url=jdbc:mysql://localhost:3306/mydb
        </value>
    </property>
</bean>
```
Spring容器通过使用JavaBeans PropertyEditor机制将\<value/>
元素内的文本转换为java.util.Properties实例。


# Customizing the Nature of a Bean (定制Bean)
spring 提供l许多可用于自定义bean特性的接口

* Lifecycle Ballbacks
* ApplicationContextAware and BeanNameAware
* Other Aware Interface
## Lifecycle Ballbacks
要与容器的bean生命周期管理进行交互, 可以实现Spring*InitializingBean*和*DisposableBean*接口.
容器为前者调用afterPropertiesSet()(ps:在属性设置之后), 为后者调用destroy(),
让bean在初始化和销毁bean时执行某些操作
> 使用JSR-250 @PostConstruce 和 @PreDestory 注解亦可以实现
> 而且不用和SpringApi耦合, 所以更好
> 也可以在xml中配置 init-method and destroy-method 属性

在 **AbstractAutowireCapableBeanFactory** 中实现
```java
	protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
			throws Throwable {

		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isDebugEnabled()) {
				logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
						@Override
						public Object run() throws Exception {
                            //重点代码
							((InitializingBean) bean).afterPropertiesSet();
							return null;
						}
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null) {
			String initMethodName = mbd.getInitMethodName();
			if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
```

###Startup and Shutdown Callbacks

org.springframework.context.Lifecyle
```java
public interface Lifecycle {
    void start();
    void stop();
    booklean isRuning();
}
public interface LifecycleProcessor extends Lifecycle {
    void onRefresh();
    void onClose();
}
```

## ApplicationContextAware and BeanNameAware

```java
public interface ApplicationContextAware {
    void setApplicationContext(ApplicationContext applicationContext) 
        throws BeansException;
}
```
Spring 在创建实现了 ApplicationContextAware 接口的类时,
会向它提供一个 ApplicationContext 的应用.

直接实现接口会导致我们的项目和 Spring 直接耦合, 自从 Spring 2.5 之后
自动配装是另一种获取 ApplicationContext 引用的另一种方法

-------
当 ApplicationContext 创建实现了 org.springframe.beans.factory.BeanNameAware 接口的类
时, 将为该类其关联对象定义中定义的名称的引用
```java
public interface BeanNameAware {
    void setBeanName(String name) throws BeansException;
}
```
它在属性填充之后但是在初始化回调(InitialzingBean,afterPropertiesSet等)之前调用这个接口

## Other Aware Interface
|Name|Injected Dependency|Explained in...
|ApplicationContextAware|Declaring ApplicationContext|ApplicationContextAware and BeanNameAware
|ApplicationEventPublisherAware|Event publisher of the enclosing ApplicationContext|Additional Capabilities of the ApplicationContext
|BeanClassLoaderAware|Class loader used to load the classes|Instantiating Beans
|BeanFactoryAware|Declaring BeanFactory|


# Container Extension Points(容器扩展点)

## Customizing Beans by Using BeanPostprocessor(使用 BeanPostProcessor自定义bean)
BeanPostProcessor 接口定义了一个 callback 方法,
你能实现它用来提供你自己的 instantiation logic(实例化逻辑),
dependency-resoluction logic(依赖解决逻辑)

如果你想实现一些自定义的逻辑在 Spring contaniner 结束 instantiating, configuring, and initalizing a bean
之后, 你可以插入一个或多个 BeanPostProcessor 实现类

你能配置多个 BeanPostProcessor 实例, 并且你也可以控制这些 BeanPostProcessor 实例的执行顺序,
通过设置 **order** 属性. 你可以设置这个属性只有在 BeanPostProcessor 实现类 **Oredered** 接口
```java
public interface BeanPostProcessor {
    Object postProcessBeforeInitialization(Object bean, String beanName);
    Object postProcessAfterInitialization(Object bean, String baenName);
}
```

# 基于注解配置
## 使用 CustomAutowireConfigurer
CustomAutowireConfigurer is a BeanFactoryPostProcessor

# Environment Abstraction
The Environment interface is an absetraction integerated in the conainer 
that models two key aspectes of the application environment : profiles and properties

环境接口被集成在容器中, 他提前了两个方面的关键抽象: profiles 和 properties

一个 profiles 是有名字的bean definitions 的logical(逻辑) group

