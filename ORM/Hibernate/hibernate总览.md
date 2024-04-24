# 结构

![](/Hibernate/imgs/data_access_layers.svg)

![](/Hibernate/imgs/JPA_Hibernate.svg)

# SessionFactory

一个 **线程安全的** 应用领域对象 的表示

创建一个 SessionFactory 是非常昂贵的 ,所以应用中应该只有一个实例

SessionFactory 维护 Hibernate 所有的 Session 二级缓存, 连接池, 事务等

# Session

**非线程安全**的 A single-threaded, short-lived object conceptually(概念) modeling
a "Unit of Work" PoEAA., 在 JPA 的概念中等同于 EntityManager

它还是一个 Transaction 实例工厂, 它维护应用程序域模型的通常“可重复读取”持久性上下文（第一级缓存）

# Transaction

**线程安全**

# 2.Domain Model

## 2.1Mapping types

org.hibernate.type.Type 接口
这个接口描述了 java 类型的各种行为方面, 例如:如何检查相等性,如果克隆值等等

* Value types

1. Basic types
2. Embeddable(嵌入的) types
3. Collection types

* Entity types

重点搞清楚 值类型和实体类型的区别

## 2.2Naming strategies

1. stage1:
   第一阶段是从域模型映射中确定正确的逻辑名称. 逻辑名可以由用户显式指定(例如:使用@Column或@Table),
   也可以由Hibernate通过Implicit(含蓄)NamingStrategy契约隐式确定

2. stage2:
   其次是将此逻辑名称解析为Physical(物理)NamingStrategy合同定义的物理名称。

## 2.3 Basic Types

### 2.3.2 @Basic 注解

```java
public @interface Basic {
    FetchType fetch() default EAGER;

    /*
    * 设置是否可以为 null ,和非空约束有关, 默认是可以为null
    */
    boolean optional() default true;
}
```

### 2.3.4 BasicTypeRegistry

org.hibernate.type.BasicTypeRegistry 是 Hibernate 知道如果映射基本属性的关键

应用程序还可以在引导期间使用MetadataBuilder#applyBasicType方法之一或
MetadataBuilder＃applyTypes方法扩展(添加新的BasicType注册)或覆盖(替换现有的BasicType注册)

### 2.3.5 Explicit(显式) BasicTypes

```java
@Entity(name = "Product")
public class Product {
	@Id
	private Integer id;

	private String sku;

	@org.hibernate.annotations.Type( type = "nstring" )
	private String name;

	@org.hibernate.annotations.Type( type = "materialized_nclob" )
	private String description;
}
```

type属性的值可以是以下之一:

* 任何org.hibernate.type.Type实现的完全限定名称
* 在BasicTypeRegistry中注册的任何密钥
* 任何已知类型定义的名称

### 2.3.6 Custom BaiscTypes(自定义基本类型映射)

开发人员可以很容易的将 java.util.BigInteger 映射为 VARCHAR columns
There are two approaches(途径) to developing a custom type:

1. implementing a BasicType and registering it

2. implementing a UserType which doesn’t require type registration

详情看官网
