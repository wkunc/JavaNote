# Publish

当一个 MQTT client 连接到 MQTT broker时, 它就可以 publish message (发布消息)

每个消息必须包含一个 topic , mqtt broker 将使用其作为依据 forward (导向) 消息到
对它们感兴趣的 client.

通常来说, 每个 message 都包含一个 payload 里面包含以字节传输的数据.
MQTT协议是 data-agnostic (数据无关的). client 确定发送的 payload 的结构.

publisher decides 是否想要发送 binary data, text data, or even XML or JSON.

MQTT-Packet
*PUBLISH*
|field | example
|------|--------|
|packetId | 4314
|topicName | "topic/1"
|qos | 1
|retainFlag | false
|payload | "temperature:32.5"
|dupFlag | false

* TopicName

> topic name 是一个有层次结构的, 通过 "/" 分隔的简单字符串
> 比如 "myhome/livingroom/temperature" or "Germany/Munich/Octoberfest/people"

* QoS

> 表示这个 message 的 Quality of Service Level(QoS, 服务质量等级).
> 有三个等级: 0, 1, 2
> QoS 确定message 对到达预期收件人( client or broker）的保证类型.
> ps: 有两个方向的, 一个是publisher发布消息时的qos, 这个代表消息到 broker的保证等级.
> subscriber 订阅topic指定的QoS代表, borker 将message 投递给sub时的保证等级.
> 如果同一消息在两个流程中指定的QoS等级不一致, 那么实际表现出来的整体行为, 按小的哪个来.

* RetainFlag

> 此标志定义消息是否由 broker 保存为指定 topic 的最后一个已知值.
> 当新client订阅主题时, 它们会收到该主题上保留的最后一条消息.

* Payload

> 这是消息的实际内容.
> MQTT与数据无关, 可以发送图像, 任何编码的文本, 加密数据以及几乎所有二进制数据.

* PacketId

> 当消息在客户端和代理之间流动时, 包标识符唯一地标识消息.
> 数据包标识符仅与大于零的QoS级别相关.

* DUP flag

> DUP标志该标志指示邮件是重复邮件.
> 并且已被重新发送, 因为目标收件人（客户或代理）未确认原始邮件, 这仅与大于0的QoS相关.
> 通常，resend/duplicate 机制由MQTT客户端库或代理作为实现细节来处理.

## Subscribe and Suback

## UnSubscribe and Unsuback


