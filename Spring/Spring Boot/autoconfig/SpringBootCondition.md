# Condition

SpringBoot除去一个依赖管理外就还剩一个自动配置功能.
怎么加载自动配置类是使用之前的SPI机制.
如何判断默认配置是否生效其实是Spring4.0提供的条件配置功能实现的.
@Profile注解的机制也在Spring4.0之后后改写通过条件配置实现.

```java
/*
* 一个在 component 被注册之前必须匹配的条件.
*
* 在将要注册 BeanDefinition 之前执行检查 Condition
* 并且可以根据当时可以确定的任何标准自由的否决注册.
*
* Condition 遵循和 BeanFactoryPostProcessor 相同的限制.
* 并注意不要与 bean 实例进行交互.
* 要更精细的控制与 @Configuration bean 交互的条件, 请考虑
* ConfigurationCondition 借口
*/
public interface Condition {
	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

# SpringBootCondition

SpringBoot在我们将日志等级设置为debug, 或者在环境抽象中指定debug=true时,
会在控制台输出哪些自动配置生效, 已经不生效的原因.

这个SpringBoot提供的SpringBootCondition主要目的就是方便输出进行条件判断时的信息.

```java

```

