# The IoC contatiner
org.springframework.beans 和 org.springframework.context 包是 Spring IoC 容器的基础

BeanFactory 接口提供了一种能够管理任何对象的高级配置机制

ApplicationContext 是 BeanFactory 的子接口 它补充了 "更容易集成 Spring'AOP, 消息资源处理(用于国际化),
事件发布, 特定于应用程序的上下文

简而言之, BeanFactory 提供了配置框架和基本功能而 ApplicationContext 添加了更多的特定于企业的功能

ApplicationContext 是 BeanFactory 的完整超集

# Container Overview(大纲)

## Configuration Metadata(元数据)
Configuration metadata 代表你如何告诉Spring 容器在应用程序中 instantiate(实例化), configure (配置)
和 assemble(组装对象)

Spring IoC contanier 本身完全与与实际编写此配置元数据的格式分离


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
```
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
Spring容器通过使用JavaBeans PropertyEditor机制将\<value/>元素内的文本转换为java.util.Properties实例。



# Customizing the Nature of a Bean (定制Bean)
spring 提供l许多可用于自定义bean特性的接口

* Lifecycle Ballbacks
* ApplicationContextAware and BeanNameAware
* Other Aware Interface
## Lifecycle Ballbacks
要与容器的bean生命周期管理进行交互, 可以实现Spring*InitializingBean*和*DisposableBean*接口.
容器为前者调用afterPropertiesSet(), 为后者调用destroy(), 让bean在初始化和销毁bean时执行某些操作


