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
 

## 1175

该方法可能在 channelRegistered event 发送前调用, 
所以为了保证register事件在connect之前(这样用户才有机会在连接之前初始化好pipeline)
使用 channel.eventLoop().execute() 执行连接逻辑

> [!TIP]
> 为啥能在channelRegistered event 发送前调用呢
> 因为 register 返回的future, 没有立刻完成时, 会向其添加Listener(方便注册成功后开始执行连接).
> 而register流程中注册成功时先 promies.trySuccess(), 此时就会直接通知Listener
> 然后在调用 fireRegisterEvent() 发送事件

```java
private static void doConnect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise connectPromise) {
    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    final Channel channel = connectPromise.channel();
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (localAddress == null) {
                channel.connect(remoteAddress, connectPromise);
            } else {
                channel.connect(remoteAddress, localAddress, connectPromise);
            }
            connectPromise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
        }
    });
}
```

## 13452
Netty 对 `Selector` 的优化存在缺陷. `SelectedSelectionKeySet` 未实现 `contains()` 方法.
在`MacOS`, `Windows` 上会丢失Event.

> 根据Issues中的描述, 可以知道在`MacOS`中如果一个通道同时有多个事件 read event (1) and write event(4)
> linux 会合并返回 5, 而MacOS会分开返回
> 由于之前的`contions`实现固定返回`false`所以导致, 统一调用`channel.translateAndSetReadyOps`.
> 这个调用会覆盖上次的readyOps, 一次循环设置了 read ops, 下一次循环设置了 write 
> 导致read event无了
> 正确实现`contains`后同一个通道第二个事件处理调用`channel.translateAndUpdateReadyOps`会进行拼接
> 从而不丢失事件

`KQueueSelectorImpl.updateSelectedKeys()`
```java
if (selectedKeys.contains(ski)) {
    // first time this file descriptor has been encountered on this
    // update?
    if (me.updateCount != updateCount) {
        if (ski.channel.translateAndSetReadyOps(rOps, ski)) {
            numKeysUpdated++;
            me.updateCount = updateCount;
        }
    } else {
        // ready ops have already been set on this update
        ski.channel.translateAndUpdateReadyOps(rOps, ski);
    }
} else {
    ski.channel.translateAndSetReadyOps(rOps, ski);
    if ((ski.nioReadyOps() & ski.nioInterestOps()) != 0) {
        selectedKeys.add(ski);
        numKeysUpdated++;
        me.updateCount = updateCount;
    }
}
```

## 13849

## 12610

## 11729

## 11708
