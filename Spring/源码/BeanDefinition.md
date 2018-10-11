# BeanDefinition
这些是属性的 get/set 方法
ParentName
BeanClassName
Scope
LazyInit
DependsOn
FactoryBeanName
FactoryMethodName

这些是属性的 set/is方法
Primary
AutowireCandidate

这些是 get 方法
ConstructorArgumentValues
getPropertyValues
getRole
getDescription
getResourceDescription
getOriginatingBeanDefinition

# BeanPostProcessor
Factory hook 允许自定义的修改新的 bean 实例
比如: 检测标记接口 或者 用代理包装它
```java
Object postProcessBeforeInitialization (Object bean, String beanName)
Object postProcessAfterInitialization (Object bean, String beanName)
```
