# BeanPostProcessor

工厂钩子, 允许自定义修改新的Bean实例, 例如检查标记接口或用代理包装它们.
ApplicationContext可以在其Bean定义中自动检测BeanPostProcessor bean.
而普通的 BeanFactory 允许进行手动编程式的注册 BeanPostProcessor.

```java
public interface BeanPostProcessor {

    /*
    * 应用这个 BeanPostProcessor 到每一个新的bean实例.
    * 在任何 initialization callbacks(比如 InitalizingBean接口, init-method属性) 之前
    * 这个 bean 总是已经填充了属性值的
    */
    Object postProcessBeforeInitizlization(Objcet bean, String beanName) throws BeanException;

    /*
    * 在任何 initialization callbacks 完成后调用
    *
    */
    Object postProcessAfterInitizlization(Object bean, String beanName) throws BeanException;
}
```

```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {
    /*
    * 在目标 bean 创建之前调用
    * 返回的 object 对象可以是代替 target bean 的 proxy 对象
    */
    Object postProcessBeforInstantiation(Class<?> beanClass, String beanName) throws BeansException;
    boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;

    PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException;

}
