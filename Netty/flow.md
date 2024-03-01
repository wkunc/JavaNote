# 事件顺序

| 事件         | 描述                                                                                                                     |
|--------------|--------------------------------------------------------------------------------------------------------------------------|
| Registered   | Channel和对应的EventLoop进行绑定, 即调用 EventLoop.registr()                                                             |
| Active       | Channel.isActive() 返回true 时调用                                                                                       |
| Read         | Netty 默认开启autoRead, 所以 NIO 在连接后会注册read事件. EventLoop 发现channel准备read时, 调用对应的方法获取到字节时调用 |
| ReadComplete |                                                                                                                          |
| Inactive     |                                                                                                                          |
| unRegistered |                                                                                                                          |

1.  pipeline.fireChannelRegistered();

> 1.  channel 与 IO线程(EventLoop) 绑定,
> 2.  NIO执行 SocketChannel.register(selector, 0, this);
> 3.  通知所有的 ChannelHandler.handlerAdded(ctx)
> 4.  发送事件

2.  pipeline.fireChannelActive();

> 在 Registered 后, 判断 isActive(), firstRegistration 发送 channelActive 事件 
> isActive() 对于NIOSocket 来说, ch.isOpen() && ch.isConnected();
> 底层通道 open 并且已经建立连接时为激活状态. NIOServerSocket 则是 isOpen() && javaChannel().socket().isBound();
> 底层通道 open 并且已经监听本地端口时为激活状态. Active 事件从 Head 开始传播. HeadContext 重写了 fireChannelActive();
> 其他handler执行后, 会执行 readIfIsAutoRead() 方法(默认配置就是autoRead) 会调用 Channel.read() 方法.
> 而 AbstractChannel 实现了 read() 调用 pipeline.read(), 然后是从tail开始调用.最后会到head中的 read() 方法 HeadContext.read() 调用了 unsafe.beginRead() 方法.
> AbstractUnsafe.read() 调用抽象方法 doBeginRead() 处理异常 AbstractNioChannel 实现了 doBeginRead() 方法,
> 给关联的Channel 注册了 readInterestOp 字段的事件 (NioSocketChannel的构造器中初始化为 OPREAD) 此后NioEventLoop 的 run()的循环就可以获取到通道的 OPREAD 事件.
> 在NioEventLoop中的可以找到对应事件的处理方法, OPREAD事件会调用 unsafe.read() NioByteUnsafe.read() 实现了从 javaChannel 中读取字节到 ByteBuf 的过程

3.  pipeline.fireChannelRead(byteBuf);
4.  pipeline.fireChannelReadComplete();

> 在循环读取 javaChannel 中的可读字节到结束后(没有可读字节了) 结束循环, 发送readCompleted() 事件. 从 Head 开始 Head 向下传递事件, 并在其他handler结束后调用 readIfAutoRead().

# BootStrap

## AbstractBootstrap

``` java
// 设置ChannelFactory, 采用反射方式调用指定 Channel 的默认构造器
public B channel(Class<? extents C channelClass>) {
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass)
}

// 创建 Channel 并绑定到指定到本地端口
// 通常是 Server 调用, 因为Client有 Connect 方法
private ChannelFutre doBind(final SocketAddress loaclAddress) {

}
```

``` java
private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
  // 1. 用 ChnnelFactory 创建 Channel
  // 2. 将指定到 ChannelHandler 放到 pipline 到末尾, 配置Channel选项 
  // 3. 将Channel注册到指定到 EventLoopGroup 
  final ChannelFuture regFuture = initAndRegister();
  final Channel channel = regFuture.channel();

  // 2. 保证在 regFuture 之后执行 doResolveAndConnect0(); 进行connect
  if (regFuture.isDone()) {
    if (!regFuture.isSuccess()) {
      return regFuture;
    }
    return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
  } else {
    // Registration future is almost always fulfilled already, but just in case it's not.
    final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
    regFuture.addListener(new ChannelFutureListener() {
      @Override
      public void operationComplete(ChannelFuture future) throws Exception {
        // Directly obtain the cause and do a null check so we only need one volatile read in case of a
        // failure.
        Throwable cause = future.cause();
        if (cause != null) {
          // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
          // IllegalStateException once we try to access the EventLoop of the Channel.
          promise.setFailure(cause);
        } else {
          // Registration was successful, so set the correct executor to use.
          // See https://github.com/netty/netty/issues/2586
          promise.registered();
          doResolveAndConnect0(channel, remoteAddress, localAddress, promise);
        }
      }
    });
    return promise;
  }
}


// AbstractBootstrap.java 307
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            // channel can be null if newChannel crashed (eg SocketException("too many open files"))
            // 注册失败时调用 closeForcibly() 方法关闭channel.
            //  这个方法不会触发任何事件(比如说unRegister, 毕竟register都没有成功就应该触发后续的事件)
            channel.unsafe().closeForcibly();
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }

    /*
     通过EventGroup的代码分析可知, 这里也是分配一个EventLoop.
     然后调用channel.unsafe().register() 实现不同的注册逻辑
     NioChannel 就会将底层scoketChannel注册到NioEventLoop上的 Selector 上
     javaChannel().register(eventLoop().unwrappedSelector(), 0, this); 注意 ops 传 0 代表都对所有事件都不感兴趣
    */
    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }

    // If we are here and the promise is not failed, it's one of the following cases:
    // 1) If we attempted registration from the event loop, the registration has been completed at this point.
    //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
    // 2) If we attempted registration from the other thread, the registration request has been successfully
    //    added to the event loop's task queue for later execution.
    //    i.e. It's safe to attempt bind() or connect() now:
    //         because bind() or connect() will be executed *after* the scheduled registration task is executed
    //         because register(), bind(), and connect() are all bound to the same thread.

    return regFuture;
}


// Bootstrap.java 188
private ChannelFuture doResolveAndConnect0(final Channel channel, SocketAddress remoteAddress,
                                           final SocketAddress localAddress, final ChannelPromise promise) {

    // 代码逻辑简化为2个步骤
    // 1. 获取地址解析器, 解析远程地址(DNS相关)
    AddressResolver<SocketAddress> resolver = this.resolver.getResolver(eventLoop);
    final Future<SocketAddress> resolveFuture = resolver.resolve(remoteAddress);

   // 2. 建立连接
    doConnect(resolveFuture.getNow(), localAddress, promise);

}

// Bootstrap.java 240
// AbstractChannel 将 connect() 方法委托给 pipeline 实现
// DefaultChannelPipeline 实现调用 tail.connect(), 会经过所有ChannelHandler 最后到达HeadContext. 调用 Unsafe.connect 方法
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

## Register 流程

Channel注册通用逻辑部分

``` java
protected abstract class AbstractUnsafe implements Unsafe {

    @Override
    public final void register(EventLoop eventLoop, final ChannelPromise promise) {
        ObjectUtil.checkNotNull(eventLoop, "eventLoop");
        if (isRegistered()) {
            promise.setFailure(new IllegalStateException("registered to an event loop already"));
            return;
        }
        if (!isCompatible(eventLoop)) {
            promise.setFailure(
                    new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
            return;
        }

        AbstractChannel.this.eventLoop = eventLoop;

        if (eventLoop.inEventLoop()) {
            register0(promise);
        } else {
            try {
                eventLoop.execute(new Runnable() {
                    @Override
                    public void run() {
                        register0(promise);
                    }
                });
            } catch (Throwable t) {
                logger.warn(
                        "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                        AbstractChannel.this, t);
                closeForcibly();
                closeFuture.setClosed();
                safeSetFailure(promise, t);
            }
        }
    }

    private void register0(ChannelPromise promise) {
        try {
            // check if the channel is still open as it could be closed in the mean time when the register
            // call was outside of the eventLoop
            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }
            boolean firstRegistration = neverRegistered;
            doRegister();
            neverRegistered = false;
            registered = true;

            // 确保在通知 promise 之前先调用 handlerAdded() 方法通知handler Add Event事件
            // (因为有些Handler会利用Add Event事件完成延迟初始化之类的行为才能正常工作)
            // 而这个 promise 上存在用户添加的ChannelFutureListener, 里可能有 fire event 动作. 需要保障所有handler都准备好了.
            pipeline.invokeHandlerAddedIfNeeded();

            // 设置promise的成功结果 
            safeSetSuccess(promise);
            // 触发 registered 事件
            pipeline.fireChannelRegistered();
            // Only fire a channelActive if the channel has never been registered. This prevents firing
            // multiple channel actives if the channel is deregistered and re-registered.
            if (isActive()) {
                if (firstRegistration) {
                    pipeline.fireChannelActive();
                } else if (config().isAutoRead()) {
                    // This channel was registered before and autoRead() is set. This means we need to begin read
                    // again so that we process inbound data.
                    //
                    // See https://github.com/netty/netty/issues/4805
                    beginRead();
                }
            }
        } catch (Throwable t) {
            // Close the channel directly to avoid FD leak.
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}
```

## Connect 流程

## Read 流程

默认情况下 autoRead 为true, active 事件中会调用beginRead() 方法 Nio模式下就会去注册对read事件的监听. NioEventLoop 循环中获取到channel准备好read时就会触发NioUnsafe.read()方法

``` java
// AbstractNioByteChannel.NioByteUnsafe
@Override
public final void read() {
  final ChannelConfig config = config();
  if (shouldBreakReadReady(config)) {
      clearReadPending();
      return;
  }
  final ChannelPipeline pipeline = pipeline();
  final ByteBufAllocator allocator = config.getAllocator();
  // RecvByteBufAllocator字面意思, 负责接受方的 byteBuf 的分配.
  // 有多种实现不同的策略来达到每次分配合理的byteBuf大小
  // 减少分配次数(多次分配自然会影响性能)且不浪费内存(不会分配过大的byteBuf)
  // NIO默认情况下使用 AdaptiveRecvByteBufAllocator (默认值在DefaultChannelConfig构造器中)
  final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
  allocHandle.reset(config);

  ByteBuf byteBuf = null;
  boolean close = false;
  try {
      do {
          // 先用分配器分配一个 byteBuf
          byteBuf = allocHandle.allocate(allocator);
          // doReadBytes(byteBuf) 就是调用底层方法读取字节到byteBuf中,并返回读取到字节数
          // 调用 allocHandle.lastBytesRead(long) 方法记录本次读取字节数量
          allocHandle.lastBytesRead(doReadBytes(byteBuf));
          // 如果本次读取字节 <=0 说明没有读取到任何内容, 释放byteBuf.
          // 如果 < 0 说明是EOF, 读已关闭
          if (allocHandle.lastBytesRead() <= 0) {
              // nothing was read. release the buffer.
              byteBuf.release();
              byteBuf = null;
              close = allocHandle.lastBytesRead() < 0;
              if (close) {
                  // There is nothing left to read as we received an EOF.
                  readPending = false;
              }
              break;
          }

          // 增长读取次数
          allocHandle.incMessagesRead(1);
          readPending = false;
          pipeline.fireChannelRead(byteBuf);
          byteBuf = null;
      // continueReading() 是否要继续读取
      } while (allocHandle.continueReading());

      allocHandle.readComplete();
      pipeline.fireChannelReadComplete();

      if (close) {
          closeOnRead(pipeline);
      }
  } catch (Throwable t) {
      handleReadException(pipeline, byteBuf, t, close, allocHandle);
  } finally {
      // Check if there is a readPending which was not processed yet.
      // This could be for two reasons:
      // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
      // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
      //
      // See https://github.com/netty/netty/issues/2254
      if (!readPending && !config.isAutoRead()) {
          removeReadOp();
      }
  }
}
```
AbstractUnsafe.java
``` java 
@Override
public RecvByteBufAllocator.Handle recvBufAllocHandle() {
  if (recvHandle == null) {
      recvHandle = config().getRecvByteBufAllocator().newHandle();
  }
  return recvHandle;
}
```

`DefaultChannelConfig` 的构造器中指定了默认的采用  [[RecvByteBuffAllocator#AdaptiveRecvByteBufAllocator|AdaptiveRecvByteBufAllocator]]

``` java
public DefaultChannelConfig(Channel channel) {
    this(channel, new AdaptiveRecvByteBufAllocator());
}

protected DefaultChannelConfig(Channel channel, RecvByteBufAllocator allocator) {
    setRecvByteBufAllocator(allocator, channel.metadata());
    this.channel = channel;
}

private void setRecvByteBufAllocator(RecvByteBufAllocator allocator, ChannelMetadata metadata) {
    checkNotNull(allocator, "allocator");
    checkNotNull(metadata, "metadata");
    // 大部分时候 maxMessagesPerRead = 16
    if (allocator instanceof MaxMessagesRecvByteBufAllocator) {
        ((MaxMessagesRecvByteBufAllocator) allocator).maxMessagesPerRead(metadata.defaultMaxMessagesPerRead());
    }
    setRecvByteBufAllocator(allocator);
}

```

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

### `dowWrite()`

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
`NioEventLoop`在监听到write事件会最后会调用到上面到 `doWrite()` 再次开始写入, 后续逻辑一样直到将等待到消息都写入 channel.
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

## Close 流程

目前发现只有调用方会触发close事件 调用 channel.close() 或者 channelContext.close() 触发

channel.closeFuture()上添加的listener会在调用doClose()方法时被调用
实际上在调用 javachannel().close()方法之前 然后调用fireChannelInactiveAndDeregister(). 
而fireChannelInactiveAndDeregister()方法先调 doDeregister() 方法完成了实际上的取消注册动作然后 fireChannelInactive(), fireChannelUnregistered()

在HeadContenxt.channelUnregistered()里最后会将handler按照顺序删除,并通知handler.

主动调用 `channel.close()` `channelContext.close()` 方法, 执行流程和上面一样.但是有close()事件. 并且也不是在 `read` 过程中发现并调用close()
