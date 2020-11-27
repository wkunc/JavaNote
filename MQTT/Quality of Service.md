# Quality of Service

Quality of Service (QoS) level 是消息发送者与接受者之间的协议, 
定义了指定message 的传递保证.
在MQTT中有3个QoS leve:

* At most one (0)
* At least one (1)
* Exactly one (2)

publisher 在发送 message 时指定消息的 QoS level.
代理使用 subscriber 在订阅主题是指定的QoS level 传递消息.

如果subscriber指定较低的 QoS level, 那么broker会以较低的QoS提供服务.

# Why is Quality of Service important?
QoS 是MQTT protocol 的关键 feature.
QoS 给与 client 选择与自身 网络状况, 应用逻辑选择服务等级的能力.
因为 MQTT 管理消息的重新传输并且保证传递(即使在底层传输不可靠的情况下),
QoS 使得不可靠网络中通信更加容易.

## QoS 0 - at most one
最小的 QoS level 是 0. 这个服务等级保证尽力而为. 没有任何关于交货的保证.
收件人不用确认收到消息, 发送者不会存储并尝试重发消息.
QoS level 0 通常被叫做 "fire and forget"(发送即丢弃) 与底层的 TCP 协议提供一样的保证.
