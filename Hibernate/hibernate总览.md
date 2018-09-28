# 结构
![](/Hibernate/imgs/data_access_layers.svg)

![](/Hibernate/imgs/JPA_Hibernate.svg)

# SessionFactory

一个 **线程安全的** 应用领域对象 的表示

创建一个 SessionFactory 是非常昂贵的 ,所以应用中应该只有一个实例

SessionFactory 维护 Hibernate 所有的 Session 二级缓存, 连接池, 事务等

# Session

**线程安全**的 A single-threaded, short-lived object conceptually(概念) modeling a "Unit of Work" PoEAA. ,在 JPA 的概念中等同于 EntityManager

它还是一个 Transaction 实例工厂,  它维护应用程序域模型的通常“可重复读取”持久性上下文（第一级缓存）

# Transaction 
**线程安全**

# 2.Domain Model

## 2.1Mapping types
