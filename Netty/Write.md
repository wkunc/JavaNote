## Write 流程

### `write(msg, promise)`

write 作为出站事件从经过的handler顺序是 tail -> head

```java
// io.netty.channel.AbstractChannel 定义的通用实现
@Override
public ChannelFuture write(Object msg, ChannelPromise promise) {
    return pipeline.write(msg, promise);
}
```

所以在`DefaultChannelPipeline.HeadContext`上实现了write方法, 转发给`unsafe.write()`

```java
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    unsafe.write(msg, promise);
}
```

`AbstractUnsafe.write()`实现了写逻辑的大致逻辑.
其实就是调用[[ChannelOutboundBuffer#`addMessage()`]]将 `Object msg` 添加待写入的链表中

```java
@Override
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();

    // 每个Unsafe在init阶段就拥有了outboundBuffer.
    // 只有调用 shutdownOutput() 或者 close() 时将 outboundBuffer 设置为null
    // 所以只要 outboundBuffer == null 就可以拒绝写入了(channel 已经关闭了输出流或者已经关闭了)
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        try {
            // release message now to prevent resource-leak
            ReferenceCountUtil.release(msg);
        } finally {
            // If the outboundBuffer is null we know the channel was closed and so
            // need to fail the future right away. If it is not null the handling of the rest
            // will be done in flush0()
            // See https://github.com/netty/netty/issues/2362
            safeSetFailure(promise,
                    newClosedChannelException(initialCloseCause, "write(Object, ChannelPromise)"));
        }
        return;
    }

    int size;
    try {
        // 子类实现消息的过滤, 比如:
        // NioByteChannel 支持 ByteBuf 或者 Fegion
        // NioDatagramChannel 支持 DatagramPacket  ByteBuf AddressedEnvelope
        // 而xxxServerChannel没有写入事件, 所以方法直接抛出 UnsupportedOperationException
        msg = filterOutboundMessage(msg);
        // 确定msg对象的大小,
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        try {
            ReferenceCountUtil.release(msg);
        } finally {
            safeSetFailure(promise, t);
        }
        return;
    }

    outboundBuffer.addMessage(msg, size, promise);
}
```

### `unsafe.flush()`

[[ChannelOutboundBuffer#`addFlush()`]]

```java
@Override
public final void flush() {
    assertEventLoop();

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }

    outboundBuffer.addFlush();
    flush0();
}
```

```java
@SuppressWarnings("deprecation")
protected void flush0() {
    if (inFlush0) {
        // Avoid re-entrance
        return;
    }

    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null || outboundBuffer.isEmpty()) {
        return;
    }

    inFlush0 = true;

    // Mark all pending write requests as failure if the channel is inactive.
    if (!isActive()) {
        try {
            // Check if we need to generate the exception at all.
            if (!outboundBuffer.isEmpty()) {
                if (isOpen()) {
                    outboundBuffer.failFlushed(new NotYetConnectedException(), true);
                } else {
                    // Do not trigger channelWritabilityChanged because the channel is closed already.
                    outboundBuffer.failFlushed(newClosedChannelException(initialCloseCause, "flush0()"), false);
                }
            }
        } finally {
            inFlush0 = false;
        }
        return;
    }

    try {
        doWrite(outboundBuffer);
    } catch (Throwable t) {
        handleWriteError(t);
    } finally {
        inFlush0 = false;
    }
}
```

### `doWrite()`

* AbstractNioByteChannel.java

``` java
@Override
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    // 写循环次数, 默认16次(ps: 读循环默认也是16次)
    int writeSpinCount = config().getWriteSpinCount();
    do {
        // 获取当前待写入entry.msg, 返回空说明已经写完了所有msg
        // 清除 OpWrite 可写事件监听
        Object msg = in.current();
        if (msg == null) {
            // Wrote all messages.
            clearOpWrite();
            // Directly return here so incompleteWrite(...) is not called.
            return;
        }
        writeSpinCount -= doWriteInternal(in, msg);
    } while (writeSpinCount > 0);

    incompleteWrite(writeSpinCount < 0);
}
```

根据entry包含的消息类型不同进行写入.
逻辑都是如果msg是个空的消息对象那么直接返回0.
如果msg完整的写入channel, 返回1
如果消息没有完整写入channel(ps: channel写满了), 返回 `ChannleUtils.WRITE_STATUS_SNDBUF_FULL` 也就是 `Integer.MAX_VALUE`
这样外面的循环条件 `writeSpinCount > 0` 就会被打破
并且 writeSpinCount 一定是个负数, 就会调用 `incomplateWrite(true)` 去设置 `NioChannel` 上的 `Opwrite`.
`NioEventLoop`在监听到write事件会最后会调用到上面到 `doWrite()` 再次开始写入, 后续逻辑一样直到将等待到消息都写入
channel.
此时 writeSpinCount > 0 成立. 调用 `incomplateWrite(false)`

``` java
// 如果写入完成返回1, 如果entry没有写完chnnel就返回了, 返回
private int doWriteInternal(ChannelOutboundBuffer in, Object msg) throws Exception {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        if (!buf.isReadable()) {
            in.remove();
            return 0;
        }

        final int localFlushedAmount = doWriteBytes(buf);
        if (localFlushedAmount > 0) {
            in.progress(localFlushedAmount);
            if (!buf.isReadable()) {
                in.remove();
            }
            return 1;
        }
    } else if (msg instanceof FileRegion) {
        FileRegion region = (FileRegion) msg;
        if (region.transferred() >= region.count()) {
            in.remove();
            return 0;
        }

        long localFlushedAmount = doWriteFileRegion(region);
        if (localFlushedAmount > 0) {
            in.progress(localFlushedAmount);
            if (region.transferred() >= region.count()) {
                in.remove();
            }
            return 1;
        }
    } else {
        // Should not reach here.
        throw new Error();
    }
    return WRITE_STATUS_SNDBUF_FULL;
}
```
