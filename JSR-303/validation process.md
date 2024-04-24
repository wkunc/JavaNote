# Constraint declaration and validation process 约束声明和验证过程

Bean Validation 规范定义了一个框架, 用于声明对JavaBean类, 字段和属性的约束.
约束在类型上声明, 并根据实例

Bean Validation 规范还提供了一种声明构造函数和方法约束的方式,
其中参数和返回值是受约束的元素.

## Requirements on classes to be validated 被验证的类的要求

期望由Bean Validation提供程序验证的对象必须满足以下要求:

1. 要验证的属性必须遵循JavaBeans读取属性的方法签名约定(必须要有getter);
2. 验证中不包括静态字段和静态方法, 就是不能进行静态字段和方法的验证.
3. 约束可以应用于接口和超类. 定义在接口或超类的约束也会被验证

约束注解的目标可以是

* type 类上
* field or property 字段上或对应getter上
* constructor or method parameter, 构造器或方法的参数
* constructor or method cross-parameter
* container element

## Constraint declaration

约束声明只要通过注释放在类或接口.
约束注释可应用于 class, 任何类型的 field, properties 上

放在类上定义约束时, 要验证的类实例将被传递给对应的 ConstraintValidator.
在字段上定义约束时, 字段值将被传递给 ConstraintValidator.
在getter上定义约束时, getter 调用结果将传递给 ConstraintValidator.

*方法和构造器约束*

*容器约束*

## Inheritance

约束注解可以放在接口或超类中.

## Group and group sequence(重点)

group 定义约束的子集, 不是验证

