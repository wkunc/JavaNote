# BeanPostProcessor

Factory hook 允许自定义的修改新的 bean 实例
比如: 检测标记接口 或者 用代理包装它

BeanPostProcessor 实例的范围是每个容器的范围,
在一个容器中定义BeanPostProcessor, 它只对该容器中的bean进行后处理.

```java
Object postProcessBeforeInitialization (Object bean, String beanName)
Object postProcessAfterInitialization (Object bean, String beanName)
```

# BeanFactoryPostProcessor

```java
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

# ImportBeanDefinitionRegistrar

```java
public interface ImportBeanDefinitionRegistrar {
    public void registerBeanDefinitions(
            AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry);
}
```
