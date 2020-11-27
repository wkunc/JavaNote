# Session and Message Expiry Intervals (会话和消息的 到期间隔)

## Session Expiry Interval

在 CONNECT packet 中, 一个正在连接的client 可以设置一个 session expiry interval (以秒为单位).
这个时间间隔定义了 MQTT broker 存储特定 Client 的Session信息的时间段.
当将 session expiry interval 设置为 0, 或者 CONNECT packet 不含有这个值时,
broker 会在client关闭连接后立刻删除session信息.
这个字段允许设置的最大值时 UNIT\_MAX (4,294,967,295) 大概是136年


## Message Expiry Interval
在 PUBLISH packet 中, client 可以设置每个消息的 expiry interval(以秒为单位).
这个时间间隔定义了 broker 需要保存这个消息多久(为了那些尚未连接的订阅者)
如果没有设置, 那么 broker 会永远保存.
当同时将 PUBLISH packet 中的 retained 设置为 true 时, 可以指示主题的保留消息的保存时间.


