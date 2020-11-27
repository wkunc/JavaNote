# MQTT 5 的优点
MQTT3 提出时还没有IoT的概念, 所以对于 cloud 等新事物的支持有限.因此对协议进行了升级.
MQTT5 在2019年3月份通过成为标志规范


# Better Error Handling for More Robust Systems (更好的错误处理)

1. session and message expiry feature (会话以及消息的过期机制)
> 允许为message或者session设置一个时间限制
> 如果在时间限制内, 消息没有被发送, 那么消息会被删除
> example, 发送了MQTT message 控制打开车间的机器, 
> 如果message在限制时间段内没有发送, 则会被删除.
> 确保只有在安全的时间段内启动机器, 并且不会因为网络延迟或中断而延迟发送.
>
> ps: 感觉用于下发那些有时效性的指令时很有用.


2. negative acknowledgements (否定确认 NACk)
MQTT broker 可以在特定情况下发送否定确认.
比如: 不支持的特性, 或者broker处于不可靠状态.
有助于MQTT broker防御 DOS 攻击

# More Scalability for Cloud Native Computing (伸缩性 云原生)

1. Shared subscription
> MQTT 5 标准化了 shared subscriptions (共享订阅)
> 共享订阅允许多个MQTT client实例, 在broker上共享相同的订阅.
> 该特性使部署在云集群上的MQTT客户机能够实现负载平衡.
> 当您使用MQTT Client 将 MQTT message存储并转发到后端企业系统
> (如数据库或企业服务总线(ESB))时, 这是非常有用的.

2. Topic aliases
主题别名允许允许用 integer 替换 topic string.
对于大型系统, topic name 结构复杂的情况下
(也就是 topic name 太长了, 而每次push message时, 都需要指定topic name.)
可以优化网络使用情况


# Greater Flexibility and Easier Integration(更好的扩展性, 更易于集成)

1. User Properties(用户属性)
> 允许添加 key-value 形式的属性到 MQTT message 的 message header 中.
> These properties allow you to add application-specific information to each message that can be used in processing the message.
> 这些属性允许你添加 处理message应用程序的特定信息到每个消息中.
> 比如添加设备的固件版本

2. Payload format indicator(负载格式标识)
> 为了使接收方更容易处理消息, 有效负载格式指示器
> 包括MIME样式的内容类型, 已经在 MQTT 5 中被添加


