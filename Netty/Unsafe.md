#Netty
# AbstractUnsafe

```java
protected abstract class AbstractUnsafe implements Unsafe {

    // 用于缓存待出站消息的缓冲区
    // 在 close() 或 shutdownOutput() 方法中会设置为null
    private volatile ChannelOutboundBuffer outboundBuffer = new ChannelOutboundBuffer(AbstractChannel.this);
    private RecvByteBufAllocator.Handle recvHandle;
    private boolean inFlush0;
}
```


# Unsafe.connect()

AbstractNioChannel.AbstractNioUnsafe
```java

        @Override
        public final void connect(
                final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
            // 设置promise为不可取消, 并且确保channel没有close
            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }

            try {
                // 如果 AbstractNioChannel.connectPromise 不为空
                // 说明有connect任务正在执行了, 直接错误
                if (connectPromise != null) {
                    // Already a connect in process.
                    throw new ConnectionPendingException();
                }

                boolean wasActive = isActive();
                // 调用AbstractNioChannel.doConnect() 方法进行实际的连接操作
                // 主要也就是这个操作, 调用jdk的connect(), 方法如果没有直接连接成功注册SelectionKey.OP_CONNECT事件
                // connected = SocketUtils.connect(javaChannel(), remoteAddress);
                // if (!connected) {
                //     selectionKey().interestOps(SelectionKey.OP_CONNECT);
                // }
                // PS:
                // NIO的特性 connect() 方法不会阻塞线程.
                // 如果直接返回了true说明连接成功了(tcp 握手完成了)
                // 如果返回了 false 说明 tcp 握手流程没完成. 
                // 此时需要监听 SelectionKey.OP_CONNECT 事件
                // 并且需要手动调用 SocketChannel.finishConnect() 完成连接.
                if (doConnect(remoteAddress, localAddress)) {
                    fulfillConnectPromise(promise, wasActive);
                } else {
                    // 如果没有直接连接成功的话, 设置ConnectionPendingException, 表示当前正在进行的连接操作
                    // PS: 和上面的检查对起来了
                    connectPromise = promise;
                    requestedRemoteAddress = remoteAddress;

                    // Schedule connect timeout.
                    int connectTimeoutMillis = config().getConnectTimeoutMillis();
                    if (connectTimeoutMillis > 0) {
                        // 创建一个延时任务, 如果触发了说明连接未完成.
                        // 返回ConnectTimeoutException并关闭channel并
                        connectTimeoutFuture = eventLoop().schedule(new Runnable() {
                            @Override
                            public void run() {
                                ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                                if (connectPromise != null && !connectPromise.isDone()
                                        && connectPromise.tryFailure(new ConnectTimeoutException(
                                                "connection timed out: " + remoteAddress))) {
                                    close(voidPromise());
                                }
                            }
                        }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
                    }

                    // 如果connect动作被取消了, 那么timeOut延时任务也需要一并取消
                    // 那么就可以取消
                    promise.addListener(new ChannelFutureListener() {
                        @Override
                        public void operationComplete(ChannelFuture future) throws Exception {
                            if (future.isCancelled()) {
                                if (connectTimeoutFuture != null) {
                                    connectTimeoutFuture.cancel(false);
                                }
                                connectPromise = null;
                                close(voidPromise());
                            }
                        }
                    });
                }
            } catch (Throwable t) {
                promise.tryFailure(annotateConnectException(t, remoteAddress));
                closeIfClosed();
            }
        }

```
