# Constraint Definition 约束定义
Constraints are defined by the combination of a constraint annotation
and a list of constraint validation implementations. 

约束由约束注解和约束验证实现列表共同定义.
约束注解应用于 type, fields, methods, constructors, parameters, container elements,
or 其他约束注解

## Constraint annotation


Generic constraint annotations can target any of the following ElementTypes:
通用约束注解的目标包括以下:
* FIELD 属性
* METHOD 方法返回值
* CONSTRUCTOR 构造器返回值 
* PARAMETER 方法或构造器参数
* TYPE 
* ANNOTATION_TYPE
* TYPE_USE

Cross-parameter constraint annotations can target any of the following ElementTypes:
交叉参数约束 不懂了?

* METHOD
* CONSTRUCTOR
* ANNOTATION_TYPE 

### Constraint Definition properties
上面描述了一个约束注解可以使用的位置, 即 ElementTypes.集.
和必须被 @Constraint 注解标记.

这一章节描述一个约束注解必须包含的属性.
#### message
每一个约束注解中必须定义一个String类型的 message 元素
```java
String message() default "{com.acme.constrain.MyConstraint.message}"
```
message 元素的值被用来创建 error message 错误信息.
建议将 message 的默认值设置为 Resource Bundle 的key.
用来启用消息国际化.

内建的Bean验证约束遵从此约定.
#### groups
每一个约束注解必须定义一个Class<?>[]类型的 groups 元素,
用来指定与约束声明关联的处理组
```java
Class<?>[] groups() default {};
```
默认值必须是空数组.

#### payload
每个约束注解必须定义一个 payload 元素

#### validationAppliesTo


## Applying multiple constraints of the same type (在同一个地方应用多个约束注解)


## Constraint validation implementation (约束定义验证的实现)
约束验证实现执行给定类型的给定约束注解的验证.
实现类由装饰约束定义的 @Constraint 的 validatedBy 元素指定.
约束验证必须实现 ConstraintValidator 接口.
```java
// 给定类型T验证约束A的逻辑, 必须遵从以下规定:
// 1. T 必须解析为非参数化类型
// 2. 或者T 的通用参数必须是无界通配符.
// A 负责验证的约束, T 约束所注解的类型, 结合起来就是描述了,
// 这个验证器负责验证什么类上的什么约束.如:负责验证 Integer上的@Max约束
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


