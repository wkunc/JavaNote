# ChannelHandler 家族

## Channel 生命周期

Channel 定义了一组和 ChannelInBoundHandler API
密切相关的简单但功能强大的状态模型.

下面列出了Channel的这4个状态.

| 状态                | 描述                                                        |
|---------------------|-------------------------------------------------------------|
| ChannelUnregistered | Channel 已经被创建, 但还未注册到EventLoop                   |
| ChannelRegistered   | Channel 已经被注册到了 EventLoop                            |
| ChannelActive       | Channel 处于活动状态(已经连接到远程节点) 可以接收和发送数据 |
| ChannelInactive     | Channel 没有连接到远程节点                                  |

## ChannelHandler 的生命周期

interface ChannelHandler定义的生命周期操作,
在 ChannelHandler 被添加到 ChannelPipeline 中或者被从 ChannelPipeline 中移除
时会调用这些操作. 这些方法中的每一个都接受一个 ChannelHandlerContext 参数

| 类型            | 描述                                               |
|-----------------|----------------------------------------------------|
| handlerAdded    | 当把ChannelHandler添加到ChannelPipeline中时被调用  |
| handlerRemoved  | 当从ChannelPipeline中移除ChannelHandler时被调用    |
| execptionCaught | 当处理过程中在ChannelPipeline中有错误产生时被调用. |


## 内置常用的Handler

### IdleStateHandler
空闲监测: 支持 readIdle, writeIdle, allIdle(指定时间内未执行任何读写触发)

功能实现非常简单, 就是根据配置增加对应的延时任务. 监测在规定时间内是否进行过读写

#### 读监测
读监测其实很简单, 就是触发读事件的时候更新最新读时间.
任务执行时判断当前时间和上次read时间的差值, 如果大于规定时间就是读空闲
reading 的作用是减少System.currentNano()调用.
之前是在read事件中更新, 现在只会在readComplete中调用一次
(一次读就绪只会调用一次System.nano())
```java
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    // This method will be invoked only if this handler was added
    // before channelActive() event is fired.  If a user adds this handler
    // after the channelActive() event, initialize() will be called by beforeAdd().
    initialize(ctx);
    super.channelActive(ctx);
}

private void initialize(ChannelHandlerContext ctx) {
    // Avoid the case where destroy() is called before scheduling timeouts.
    // See: https://github.com/netty/netty/issues/143
    switch (state) {
    case 1:
    case 2:
        return;
    default:
         break;
    }

    state = ST_INITIALIZED;
    initOutputChanged(ctx);

    lastReadTime = lastWriteTime = ticksInNanos();
    if (readerIdleTimeNanos > 0) {
        readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                readerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (writerIdleTimeNanos > 0) {
        writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                writerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (allIdleTimeNanos > 0) {
        allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                allIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
}

@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (readerIdleTimeNanos > 0 || allIdleTimeNanos > 0) {
        reading = true;
        firstReaderIdleEvent = firstAllIdleEvent = true;
    }
    ctx.fireChannelRead(msg);
}

@Override
public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
    if ((readerIdleTimeNanos > 0 || allIdleTimeNanos > 0) && reading) {
        lastReadTime = ticksInNanos();
        reading = false;
    }
    ctx.fireChannelReadComplete();
}

```java
private final class ReaderIdleTimeoutTask extends AbstractIdleTask {

    ReaderIdleTimeoutTask(ChannelHandlerContext ctx) {
        super(ctx);
    }

    @Override
    protected void run(ChannelHandlerContext ctx) {
        long nextDelay = readerIdleTimeNanos;
        if (!reading) {
            nextDelay -= ticksInNanos() - lastReadTime;
        }

        if (nextDelay <= 0) {
            // Reader is idle - set a new timeout and notify the callback.
            readerIdleTimeout = schedule(ctx, this, readerIdleTimeNanos, TimeUnit.NANOSECONDS);

            boolean first = firstReaderIdleEvent;
            firstReaderIdleEvent = false;

            try {
                IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);
                channelIdle(ctx, event);
            } catch (Throwable t) {
                ctx.fireExceptionCaught(t);
            }
        } else {
            // Read occurred before the timeout - set a new timeout with shorter delay.
            readerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
        }
    }
}
```

#### 写监测

写监测其实原理也一样, 每次写完一个msg时更新最新写时间.
PS: 这里特殊的是每次写完才会更新时间, 所以当写入一个很大的msg时由于没有及时写完,会导致触发写空闲(但此时其实是在写入中)
这时候需要监测写队列中的字节是否发生变化,作为是否真正写空闲的判断依据
所以IdleStateHandler增加一个observeOutput开关控制相关功能.
当遇到上述情况可以选择打开对应功能, 实现准确的写空闲监测

```java

    // Not create a new ChannelFutureListener per write operation to reduce GC pressure.
    private final ChannelFutureListener writeListener = new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
            lastWriteTime = ticksInNanos();
            firstWriterIdleEvent = firstAllIdleEvent = true;
        }
    };

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        // Allow writing with void promise if handler is only configured for read timeout events.
        if (writerIdleTimeNanos > 0 || allIdleTimeNanos > 0) {
            // 由于是通过write的完成后才更新的 lastWriteTime. 如果有一个很大的buffer在写入
            // 还没写完就触发了writeTask会到导致监测到writeIdle.
            ctx.write(msg, promise.unvoid()).addListener(writeListener);
        } else {
            ctx.write(msg, promise);
        }
    }

    private final class WriterIdleTimeoutTask extends AbstractIdleTask {

        WriterIdleTimeoutTask(ChannelHandlerContext ctx) {
            super(ctx);
        }

        @Override
        protected void run(ChannelHandlerContext ctx) {

            long lastWriteTime = IdleStateHandler.this.lastWriteTime;
            long nextDelay = writerIdleTimeNanos - (ticksInNanos() - lastWriteTime);
            if (nextDelay <= 0) {
                // Writer is idle - set a new timeout and notify the callback.
                writerIdleTimeout = schedule(ctx, this, writerIdleTimeNanos, TimeUnit.NANOSECONDS);

                boolean first = firstWriterIdleEvent;
                firstWriterIdleEvent = false;

                try {
                    if (hasOutputChanged(ctx, first)) {
                        return;
                    }

                    IdleStateEvent event = newIdleStateEvent(IdleState.WRITER_IDLE, first);
                    channelIdle(ctx, event);
                } catch (Throwable t) {
                    ctx.fireExceptionCaught(t);
                }
            } else {
                // Write occurred before the timeout - set a new timeout with shorter delay.
                writerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
            }
        }
    }

    // 从当前待写入的msg对象进行比较, 然后比较前后的写入字节数. 来检测通道在写入一个大对象的情况
    private boolean hasOutputChanged(ChannelHandlerContext ctx, boolean first) {
        if (observeOutput) {

            // We can take this shortcut if the ChannelPromises that got passed into write()
            // appear to complete. It indicates "change" on message level and we simply assume
            // that there's change happening on byte level. If the user doesn't observe channel
            // writability events then they'll eventually OOME and there's clearly a different
            // problem and idleness is least of their concerns.
            if (lastChangeCheckTimeStamp != lastWriteTime) {
                lastChangeCheckTimeStamp = lastWriteTime;

                // But this applies only if it's the non-first call.
                if (!first) {
                    return true;
                }
            }

            Channel channel = ctx.channel();
            Unsafe unsafe = channel.unsafe();
            ChannelOutboundBuffer buf = unsafe.outboundBuffer();

            if (buf != null) {
                int messageHashCode = System.identityHashCode(buf.current());
                long pendingWriteBytes = buf.totalPendingWriteBytes();
                // 监测当前写对象是否发生变化

                if (messageHashCode != lastMessageHashCode || pendingWriteBytes != lastPendingWriteBytes) {
                    lastMessageHashCode = messageHashCode;
                    lastPendingWriteBytes = pendingWriteBytes;

                    if (!first) {
                        return true;
                    }
                }

                // 如果写对象没有变化, 查看已写入的字节数是否发生变化
                long flushProgress = buf.currentProgress();
                if (flushProgress != lastFlushProgress) {
                    lastFlushProgress = flushProgress;

                    if (!first) {
                        return true;
                    }
                }
            }
        }

        return false;
    }

```

### FlushConsolidationHandler
用于减少 flush() 的调用次数, 因为flush()会触发syscall, 不论当前消息是多大. 减少flush可以提升性能
`Apache Cassandra` 项目里 `org.apache.cassandra.transport.Message#Dispatcher`也是一个类似的减少flush次数的优化.


### ChunkedWriteHandler
异步大数据读写


### ChannelTrafficShapingHandler
流量控制

### SSLHandler


### proxy Protocol
网络环境中通常会有nginx之类的前置代理服务
此时导致建立的tcp连接都是相同ip(由nginx转发了client的tcp).
HTTP 通常只需要添加(重写)相关的请求头就好了 (Host, X-Forwarded-For)
TCP 可以用nginx proxy_protocol 功能启用 Proxy Protocol协议
```
stream {
    server {
        listen              12345;
        proxy_pass          backend.example.com:8080;
        proxy_protocol      on;
    }

}
```
协议规定: 建立TCP连接后, 会首先发送一个proxy protocol包 (内容也很简单就是原始连接的IpAddress)
然后才是应用层协议数据.
所以Server端可以使用 `io.netty.handler.codec.haproxy.HAProxyMessageDecoder` 进行解码.

来自`io.vertx.core.net.impl.NetServerImpl.NetServerWorker#initChannel(ChannelPipeline, boolean)`的例子
```java
  // 在启用代理协议解析的情况下执行
  if (HAProxyMessageCompletionHandler.canUseProxyProtocol(options.isUseProxyProtocol())) {
    IdleStateHandler idle;
    io.netty.util.concurrent.Promise<Channel> p = ch.eventLoop().newPromise();
    // 添加proxy protocol解析器
    ch.pipeline().addLast(new HAProxyMessageDecoder());
    // 如果配置了等待时长的话, 增加一个读空闲监测handler
    if (options.getProxyProtocolTimeout() > 0) {
      ch.pipeline().addLast("idle", idle = new IdleStateHandler(0, 0, options.getProxyProtocolTimeout(), options.getProxyProtocolTimeoutUnit()));
    } else {
      idle = null;
    }
    // 添加包处理器
    ch.pipeline().addLast(new HAProxyMessageCompletionHandler(p));
    // 在这个promies完成后(协议解析完成, 知道原始IpAddress了),
    // 移除之前添加idle. 并且执行 configurePiplien() 方法将其他业务handler添加到pipeline上
    // PS: 所以需要注意, 按这个流程的话, 业务Handler是建立连接后添加的, 所以收不到ChannelActive事件(所以通常会在HandlerAdded中执行初始化动作)
    p.addListener((GenericFutureListener<io.netty.util.concurrent.Future<Channel>>) future -> {
      if (future.isSuccess()) {
        if (idle != null) {
          ch.pipeline().remove(idle);
        }
        configurePipeline(future.getNow());
      } else {
        //No need to close the channel.HAProxyMessageDecoder already did
        handleException(future.cause());
      }
    });
  } else {
    configurePipeline(ch);
  }
```

逻辑非常简单, 如果协议不规范情况, 直接close. 正常流程就是从包中获取原始的IpAddress放到channel的属性中.
这样后续业务可以通过channel属性获取到真实的IpAddress, 而不是用`channel.remoteAddress()`获取到代理的地址
```java
public class HAProxyMessageCompletionHandler extends MessageToMessageDecoder<HAProxyMessage> {
  //Public because its used in tests
  public static final IOException UNSUPPORTED_PROTOCOL_EXCEPTION = new IOException("Unsupported HA PROXY transport protocol");

  private static final Logger log = LoggerFactory.getLogger(HAProxyMessageCompletionHandler.class);
  private static final boolean proxyProtocolCodecFound;

  private final Promise<Channel> promise;

  public HAProxyMessageCompletionHandler(Promise<Channel> promise) {
    this.promise = promise;
  }


  @Override
  protected void decode(ChannelHandlerContext ctx, HAProxyMessage msg, List<Object> out) {
    HAProxyProxiedProtocol protocol = msg.proxiedProtocol();

    //不支持的情况, 直接close
    //UDP over IPv4, UDP over IPv6 and UNIX datagram are not supported. Close the connection and fail the promise
    if (protocol.transportProtocol().equals(HAProxyProxiedProtocol.TransportProtocol.DGRAM)) {
      ctx.close();
      promise.tryFailure(UNSUPPORTED_PROTOCOL_EXCEPTION);
    } else {
      // 从包中获取原始的IpAddress, 添加到channel的属性中.
      // 后续如果需要用到client的真实Ip可以从channel的属性中获取
      /*
      UNKNOWN: the connection is forwarded for an unknown, unspecified
      or unsupported protocol. The sender should use this family when sending
      LOCAL commands or when dealing with unsupported protocol families. The
      receiver is free to accept the connection anyway and use the real endpoint
      addresses or to reject it
       */
      if (!protocol.equals(HAProxyProxiedProtocol.UNKNOWN)) {
        if (msg.sourceAddress() != null) {
          ctx.channel().attr(ConnectionBase.REMOTE_ADDRESS_OVERRIDE)
            .set(createAddress(protocol, msg.sourceAddress(), msg.sourcePort()));
        }

        if (msg.destinationAddress() != null) {
          ctx.channel().attr(ConnectionBase.LOCAL_ADDRESS_OVERRIDE)
            .set(createAddress(protocol, msg.destinationAddress(), msg.destinationPort()));
        }
      }
      ctx.pipeline().remove(this);
      promise.setSuccess(ctx.channel());
    }
  }


  @Override
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    promise.tryFailure(cause);
  }

  @Override
  public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
    if (evt instanceof IdleStateEvent && ((IdleStateEvent) evt).state() == IdleState.ALL_IDLE) {
      ctx.close();
    } else {
      ctx.fireUserEventTriggered(evt);
    }
  }

  private SocketAddress createAddress(HAProxyProxiedProtocol protocol, String sourceAddress, int port) {
    switch (protocol) {
      case TCP4:
      case TCP6:
        return SocketAddress.inetSocketAddress(port, sourceAddress);
      case UNIX_STREAM:
        return SocketAddress.domainSocketAddress(sourceAddress);
      default:
        throw new IllegalStateException("Should never happen");
    }
  }
}

```

