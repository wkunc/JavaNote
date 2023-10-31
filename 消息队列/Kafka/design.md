# Main Concepts and Terminology.

An event records the fact that "something happened" in the world or in your business.

It is also called record or message in the documentation.

When you read or write data to kafka, you do this in the form of events.

Conceptually an event has key, value, timestamp, and optional metadata headers

* Event key: "Alice"
* Event value: "made a payment of $200 to Bob"
* Event timestamp: "Jun. 25,2020 at 2:06 p.m"




# Topic and log
Kafka的核心概念: 提供一串流式的记录--topic.

Topic 就是数据主题, 是数据记录发布的地方, 可以用来区分业务系统.
kafka中的Topic总是多订阅者模式(), 一个Topic可以拥有一个或者多个消费者来订阅它的数据.

对与没一个Topic, kafka集群会维持一个分区日志, 如下所示:

![]()

每个分区都是有序且顺序不可变的记录集, 并且不断地追加到结构化的commit log 文件.
分区中的每一个记录都会分配一个id来表示顺序, 我们称之为 offset,
offset用来唯一表示分区中的每一条记录.


