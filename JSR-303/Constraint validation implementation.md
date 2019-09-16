上文我们研究了如何定义一个约束, 和如何组合已有约束.
下面我了解如何是实现约束的验证.

约束验证实现执行的给定类型的给定约束注解的验证.
实现类由装饰约束的@Constraint注释的 validatedBy 元素指定.

验证器实现 Constraintvalidator 接口.
这个接口的定义意味着一个 约束注解 应该会有多个验证器来处理不同的类型上的验证逻辑.
```java
// 给定类型T验证约束A的逻辑, 必须遵从以下规定:
// 1. T 必须解析为非参数化类型
// 2. 或者T 的通用参数必须是无界通配符.
public interface ConstraintValidator<A extends Annotation, T> {

    // 在初始化验证器以准备 isValid()方法调用前调用.
    // 会传递对应的约束注解的实例, 可用来获取约束的MetaData元数据
    // 默认为空实现.
    default void initialize(A constraintAnnotation) {
    }

    // 实现验证逻辑, 不可以改变 value 的状态.(比如把 value 指向null, 或者指向别的东西)
    // 这个方法可能会被多线程同时访问, 必须通过实现确保线程安全.
    boolean isValid(T value, ConstraintValidatorContext context);
}
```
A 代表这个验证器支持什么约束.
T 代表这个验证器支持验证什么类型.
```java
@Size(min = 1, max = 10)
List<String> a;

这个验证器支持验证Collection 上的 Size 约束的逻辑.
public class SizeValidatorForCollection implements ConstraintValidator<Size, Collection> {

}
```
## ConstraintValidatorContext
传递给 isValid() 方法的 ConstraintValidatorContext 对象携带在在约束被验证的上下文中可用的信息和操作.
```java
```

# ConstraintValidatorFactory
```java
public interface ConstraintValidatorFactory {

    <T extends ConstraintValidator<?, ?>> T getInstance(Class<T> key);

    void releaseInstance(ConstraintValidator<?, ?> instance);
}
```
