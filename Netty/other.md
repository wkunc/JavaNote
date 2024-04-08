# 有趣的Issues
学习Netty代码时, 会看到不少有用的Issues注释以及应用场景.
在此记录

## 写错误是否自动关闭
[After the channel closes, client not always receives last part of data](https://github.com/netty/netty/issues/1952)

在此Issues之前 Netty 在写入时碰到`IOException`时会主动调用`AbstractUnsafe.close()`关闭连接
```java
try {
    doWrite(outboundBuffer);
} catch (Throwable t) {
    outboundBuffer.failFlushed(t);
    if (t instanceof IOException) {
        close(voidPromise());
    }
}
```
如果server向client发送 Disconnect 等包后,关闭连接. 数据到达client时, 正好处于写入流程中. 
由于自动关闭的逻辑, server 发送的最后数据包就会被丢弃. 调用 `AbstractUnsafe.close()` 意味着通道已经关闭
不会尝试读取系统缓冲区里剩余的数据了. (反过来也成立, client 发给server的数据也可能被丢弃)
所以后面增加了 `DefaultChannelConfig.autoClose` 字段默认为`true`.  `ChannelOption.AUTO_CLOSE` 设置选项.
用来把写错误时自动关闭的逻辑关闭掉. 
这样的话, `EventLoop`就会响应channel上的read事件, 读取到最后的数据包, 并且发现连接已经关闭(读到了EOF).
如此就不会错过最后的数据包

> PS: 感觉这是一个很容易触发的情况, 因为看到的大部分应用协议都是先发送一个`Disconnect`含义的包, 然后关闭连接.
> 默认又是写错误自动关闭的, 会导致client没有收到最后的 `Disconnect` 包
> 感觉大部分情况下都应该将 `autoClose` 设置为false.
 
