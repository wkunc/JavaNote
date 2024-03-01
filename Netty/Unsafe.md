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
