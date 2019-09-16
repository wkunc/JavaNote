---
传统的API
# ConnectionFactory 
客户端使用这个对象创建到提供者的链接

# Connection
客户端到JMS提供者之间的活动链接

# Session
发送和接受消息的单线程上下文

# MessageProducer
由Session创建的对象, 用于发送消息到目的地(Queue或Topic)
# MessageConsumer
由Session创建的对象, 用于接收目的地(Queue或Topic)中的消息

# Destination 目的地
JMS定义了两种模型, 点对点和订阅发布模型.
它们都是发送的目的地, 分别对应 Queue, Topic 两个子接口.


# Message
---
简化的API, 
在简化API中JMSContext对象封装了传统API中Connection和Session两个对象的行为.
但是JMSContext底层并没由使用这两个接口.
# ConnectionFactory
和上面一样

# JMSContext
客户端到JMS提供者之间的活动连接, 以及发送接收消息的一个单线程上下文.

# JMSProducer
由JMSContext创建的对象, 用于发送消息到Queue或Topic

# JMSConsumer
由JMSContext创建的对象, 用于接收Queue或Topic中的消息.


---
遗留的API

# QueueConnectionfactory
# QueueConnection
# QueueSession
# QueueSender
# QueueReceiver

# TopicConnectionfactory
# TopicConnection
# TopicSession
# TopicSession
# TopicReceiver


---
# 3.消息模型
企业消息产品把消息当做一个轻量级的实体, 由一个消息头和一个消息体组成.
消息头包括一些用于消息标识和消息路由的字段;
消息体包含被发送的应用数据.

基于这个基础的格式, 不同的消息产品在消息具体的定义上是显著不同的.
最主要的区别是消息头部的语义和消息内容.

对于JMS来说, 包括所有的这些内容提供一个统一的消息模型,无疑是非常困难的.

## 目标
* 提供一个统一的消息API
* 提供一组适合已有非JMS应用创建其格式的消息的API
* 支持跨操作系统、机器架构和编程语言的复杂应用的开发
* 支持包含Java对象的消息
* 支持包含可扩展标记语言（Extensible Markup Language，XML）的消息

## JMS 消息
JMS 消息由如下几个部分组成:
* 消息头(Header): 所有的消息都支持同样的一组消息头字段. 它包含了client和 provider 用于标识和路由消息的值
* 消息属性(properties): 除了标准的消息头字段, 还可以为消息提供多个可选的消息头字段.
    1. 应用的相关属性
    2. 标准属性
    3. 提供者相关属性
* 消息体(body): JMS 定义了几种消息类型的消息体, 这几种消息体覆盖了当前使用种的大部分消息风格.
### header
10个头部字段
|name|Description
|---|---|
|JMSDestination(目的地)|包含消息发送到目标.
|JMSDeliveryMode(输送模式)|包含发送消息指定的传递模式
|JMSMessageID|包含唯一标识提供程序发送的每条消息的值, 它是一个String值, 它应该作为用于标识存储库中的消息的唯一键. 所有JMSMessageID值必须以"ID:"开头.
|JMSTimestamp|包含将消息传递给提供程序的时间戳.这不是消息实际传输的时间, 因为实际发送可能由于事务或其他客户端消息排队而稍后发生.
|JMSCorrelationID|
|JMSReplyTo(答复)|包含发送消息时提供的目标, 它是应该发送消息的目的地.
|JMSRedelivered(交还)|和消息确认有关.
|JMSType|包含客户端发送消息时提供的消息类型标识符.
|JMSExpiration(期限,满期)|发送消息时, JMS提供程序通过发送方法上指定的生存时间添加到发送消息的时间来计算其到期时间. 它是与1970.1.1的时间插值单位毫秒值.
|JMSPriority|包含消息的优先级, JMS定义了0~9一共10级, 0为最低, 9为最高. 0~4视为普通优先级, 5~9视为高级优先级. JMS不要求提供者严格实现消息的优先级排序.但它应该尽力先发送加急消息.
|JMSDeliveryTime|

注意这些header字段在Message接口中提供了setter方法. 但是有些属性自己设置是被忽略的, 应该它的语义应该交给 JMS Provider 来设置.
1. JMSDestination
2. JMSDeliveryMode
3. JMSExpiration
4. JMSDeliveryTime
5. JMSPriority
6. JMSMessageID
7. JMSTimestamp

还剩下4个是由我们这些使用者设置的.

### properties
属性也是Header的一部分, 只不过是可选的.
它相当于自定义的Header字段. 上面将的是强制的Header字段.

### body

## 3.8消息选择
许多的消息应用需要对它们生成的消息进行过滤和分类.

在订阅发布模型中, 当消息被广播到很多client时, 将条件放入消息头中使JMS provider 可以看到是非常有用.
这个模型中一条消息会发布给所有订阅者, 但是不是所有的订阅者对这个消息感兴趣, 这时就涉及到了消息的选择.
最简单的方法就是不感兴趣的client把这个消息抛弃, 感兴趣的client读取消息. 这样消息的选择在client实现.
我们也可以让JMS provider 来完成大部分的过滤和路由工作.

JMS 提供了一种允许client将消息选择委派给提供者的工具.
这样会简化客户端工作, 并且也会节省带宽(不需要全部广播了).

生产者使用特定于应用程序的消息选择标准附加到消息的属性里,
消费者使用 JMS message selector expressions 来指定消息选择标准.

### 消息选择器

----
语法
消息选择器是一个String, 其语法基于SQL92条件表达式语法的子集.

一个选择器可以包含:
1. Literals(文字)
2. Identifiers:
3. Whitespace:
4. Expressions:
5. Standard bracketing
6. Logical operators
7. Comparison operators
8. Arithmetic operators



# 访问发送的消息(生产者)
发送消息后, 客户端可以保留和修改, 而不会影响到已发送的邮件.
相同的消息对象可以多次发送.

但是在它的sender方法执行时, 不能修改消息, 如果被修改,发送的结果是不确定的.

# 修改接收的消息(消费者)


# JMS message body
JMS 定义了5中Message类型: 表示为 Message 接口的子接口.

# Message Domain
1. PTP
2. publish and subscribe
## PTP
点对点系统工作和消息队列一起, 它是点对点的, 因为 clien 发送 message 到特定的Queue.

但是一些PTP消息系统模糊了PTP和pub/sub之间的区别提供自动分发消息的系统客户端.
客户端通常将消息传给同一个队列. 和通用的邮箱一样, 队列可以包含多种消息.
这样设计的原因是: 创建和维护每个队列是昂贵的.

JMS PTP 模型定义客户端如何使用队列: 
1. 它如何找到它们
2. 它如何向它们发送消息
3. 它如何从中接收消息

换句话说, JMS 不规定PTP模型的实现方式, 无论是维护不同的队列.
还是维护一个通用队列, 用标记来代表每条消息的所属队列.
它只关系如何找到它和使用它.
ID:851c1c0c-94f9-11e9-8794-d8cb8af51d35
 
ID:895d6e46-94f9-11e9-8685-d8cb8af51d35
