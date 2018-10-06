# BeanFactory
它是 Spring bean container 的 root interface 

它主要定义了一系列的 getBean() 方法

# ListableBeanFactory
定义了访问容器中 Bean 基本信息的若干方法

# HierarchicalBeanFactory
定义了父子级联的 Ioc 容器接口
```java
public interface HierarchicalBeanFactory extends BeanFactory{
    //返回父容器, 如果没有返回 null
    BeanFactory getParentBeanFactory();
    // 判断当前 BeanFactory 中是否有指定名字的 Bean ,忽略父容器中的
    boolean containsLocalBean(String name);
```

# ConfigurableBeanFactroy
这是一个重量级的接口, 它增强了 IoC 容器的可定制性(ps:可配置的BeanFactroy嘛)

它定义了设置类装载器, 属性编辑器, 容器初始化后置处理器 等方法

