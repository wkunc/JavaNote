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


