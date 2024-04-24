# 建立连接

# MQTT Connection

MQTT 连接总是发生在一个 client 与 mqtt broker 之间. client 之间不会产生连接
(消息订阅模型都这样)

1. Client send CONNECT message to the broker.
2. broker 回复一个 CONNACK(连接确认) message 和 status code

一个连接建立后, mqtt broker 保持连接open, 直到 client send disconnect command(断开连接指令)
或者 connecttion break (连接中断, 在约定的心跳周期内mqtt broker 没有收到心跳包)

## Client initiates connection with the CONNECT message

为了初始化一个连接, 这个client 发送一个命令消息到 mqtt broker.
如果 CONNECT message 格式不正确(不符合 MQTT 规范) 或者在打开*socket连接*和发送*connect message*
之间花费了过多的时间, mqtt broker 会关闭连接.
这种行为可以阻止可以降低 mqtt broker 速度的恶意client.

*MQTT-Packet*
CONNECT

| field                      | example           |
|----------------------------|-------------------|
| cliend                     | "client-1"        |
| cleanSession               | true              |
| username (optional)        | "hans"            |
| password (optional)        | "letmein"         |
| lastWillTopic (optional)   | "/hans/will"      |
| lastWillQos (optional)     | 2                 |
| lastWillMessage (optional) | "unexpected exit"
| lastWillRetain (optional)  | false             |
| keepAlive                  | 60

* ClientId

> ClientId (client identifier) 用于每个连接到 MQTT Broker 的 MQTT Client.
> MQtt broker 使用 ClientId 标识每个 client 以及 client 的status.
> 所以 ClientId 应该是唯一的.
> 在 MQTT 3.1.1 中你可以发送一个 empty ClientId. 如果不需要broker保持任何状态.
> empty ClientId 导致没有任何状态的连接, 这种情况下必须将 cleanSession 标志位设置为 true, 否则broker 将拒绝连接.

* CleanSession

> CleanSession 标志位告诉 broker 这个客户端想要建立一个持久化的 session 或者不想.
> 在持久化 session (CleanSession = false) 中, mqtt broker 存储所有这个 client 订阅的消息.
> 以及所有的这个 client 订阅的 Qos 1 or Qos 2 错过的消息
> 反之 broker 不会存储任何内容, 并且会清除之前持久化session中的所有内容.

* Username/Password

> MQTT 可以发送 username password 进行 client 的认证授权.
> 然而, 如果这些信息没有经过加密或者hash处理, password 会明文传递(不安全).
> 所以推荐使用 username/password 时, 用一个secure transport (安全传输)

* WillMessage

> last will message is part of the Last Will and Testament (LWT) feature of MQTT
> 当client非正常断开连接时, 这个消息将会通知给其他client.
> 当client 连接时, 它可以向 broker 提供 last will message 通过包含在 CONNECT message 中.

* KeepAlive

> keepAlive 是在client在建立连接时指定与broker通信的时间间隔(以秒为单位).
> 这个间隔定义了代理和客户端在不发送消息的情况下所能承受的最长时间.
> 客户端承诺向代理发送 regular PING request message. 代理以 PING reponse进行响应.

## Broker response with a CONNACK message

当 mqtt broker 收到一个 CONNECT message时, 它有义务回复一个 CONNACK message(连接确认消息).

CONNACK message 包含两部分数据.

* session presend flag
* A connect return code


