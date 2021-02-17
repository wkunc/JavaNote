# Receiving Messages (接收消息)
You can receive message by configuring a MessageListenerContainer and providing a
message listener or by using the @KafkaListener annotation.

## Message Listeners

when you use a message listener container, you must provide a listener to receive data.
当你使用 message listener container 时, 必须提供一个 listener 来接收数据.

下面时当前支持的8种 message Listeners 接口类型
```java
public interface MessageListener<K, V> {
    void onMessage(ConsumerRecord<K, V> data);
}

public interface AcknowledgingMessageListerner<K, V> {
    void onMessage(ConsumerRecord<K, V> data, Acknowledgment acknowledgment);
}

public interface ConsumerAwareMessageListener<K, V> extends MessageListener<K, V> {
    void onMessage(ConsumerRecord<K, V> data, Consumer<?, ?> consumer);
}

public interface AcknowledgingConsumerAwareMessageListener<K, V> extends Message<K, V> {
    void onMessage(ConsumerRecord<K, V> data, Acknowledgment acknowledgment, Consumer<?, ?> consumer);
}

```

## Message Listener Containers

* KafkaMessageListenerContainer
* ConcurrentMessageListenerContainer

KafkaMessageListenerContainer 接收所有 topic or partition 的消息在单个线程中.
ConcurrentMessageListenerContainer 委托到一个或多个KafkaMessageListenerContainer 实例, 以提供多线程使用.

since version 2.2.7 可以添加 RecordInterceptor 到 listener container.
拦截器将会在调用 listener 之前调用, (可以拦截或者修改 record), 如果 listener 返回null, 那么listener将不会执行.
NOTE: 这个拦截器机制不会在 batch listener 流程中触发.


since version 2.3 可以应用 CompositeRecordInterceptor 来组合应用多个拦截器

默认情况下, 在使用事务时, 拦截器会在事务开始之后调用.
since version 2.3.4 可以通过 listener container 的 *interceptBeforeTx* 属性控制拦截器在事务之前触发.


### Using KafkaMessageListenerContainer

```java
public class KafkaMessageListenerContainer<K, V> // NOSONAR line count
		extends AbstractMessageListenerContainer<K, V> {

    public KafkaMessageListenerContainer(ConsumerFactory<K, V> consumerFactory,
                        ContainerProperties containerProperties);

}
```

构造器接收一个 *ConsumerFactory* 和一个表示 topic and partition 等配置的 ContainerProperties 对象.

### Committing Offsets

* RECORD: 
* BATCH:
* TIME:
* COUNT:
* COUNT_TIME:
* MANUAL:
* MANUAL_TIMEDATE:

### @KafkaListener Annotation
@KafkaListener annotation 被用来将一个 bean method 作为一个 listener 给 listener contationer.
这个 bean 将被包装成 *MessagingMessageListenerAdapter*

可以在注解上shiyong SPEL 或者属性占位符来设置属性.


### Record Listeners

@KafkaListener 注解提供了一种 Simple POJO listeners 的机制.
```java
public class Listener {

    @KafkaListener(id = "foo", topics="myTopic", clientIdPrefix = "myClientId")
    public void listen(String data) {
        //....
    }

}
```

这个机制 requires 一个 @EnableKafka 注解在配置类上. 并且需要一个
用来配置底层 ConcurrentMessageListenerContainer 的 Listener Container Factory.
默认情况下, 需要一个 beanName 为 *KafkaListenerContainerFactory*.


