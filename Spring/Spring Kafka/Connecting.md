# Connecting to Kafka

* KafkaAdmin
* ProducerFactory
* ConsumerFactory

Factory Listeners

从2.5 version开始, DefaultKafkaProducerFactory, DefaultKafkaConsumerFactory 都可以配置
一个 Lister 来接受 producter/consumer 被创建或者关闭事件.

```java
interface Listener<K, V> {
    default void producerAdded(String id, Producer<K, V> producer) {
    }

    default void producerRemoved(String id, Producer<K, V> producer) {
    }
}
```

```java
interface Listener<K, V> {
    default void consumberAdded(String id, Consumber<K, V> consumber) {
    }

    default void consumberRemoved(String id, Consumber<K, V> consumber) {
    }
}
```

# Configuring Topics

如果在 Application 中定义了 **KafkaAdmin**. 它允许自动创建Topic.
通过创建 **NewTopic** 类型的Bean在ApplicationContext中.
2.3version 之后新增了 TopicBuilder 来方便的创建
