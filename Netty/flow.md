# Netty Client 初始化流程

# BootStrap

## AbstractBootstrap

```java
// 设置ChannelFactory, 采用反射方式调用指定 Channel 的默认构造器
public B channel(Class<? extents C channelClass>) {
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass)
}


// 创建 Channel 并绑定到指定到本地端口
// 通常是 Server 调用, 因为Client有 Connect 方法
private ChannelFutre doBind(final SocketAddress loaclAddress) {

}

```


```java
public ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
    // 1. 用 ChnnelFactory 创建 Channel
    // 2. 将指定到 ChannelHandler 放到 pipline 到末尾, 配置Channel选项 
    // 3. 将Channel注册到指定到 EventLoopGroup
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();

    // 简化异常处理等步骤
    // 4. 
    return doResolveAndConnect0(final SocketAddress remoteAddress, final SocketAddress localAddress);
}
private ChannelFuture doResolveAndConnect0(final Channel channel, SocketAddress remoteAddress,
                                           final SocketAddress localAddress, final ChannelPromise promise) {

    // 获取 地址解析器, 复制解析远程地址
    AddressResolver<SocketAddress> resolver = this.resolver.getResolver(eventLoop);

    // 
    final Future<SocketAddress> resolveFuture = resolver.resolve(remoteAddress);

    // 
    doConnect(resolveFuture.getNow(), localAddress, promise);

}

private static void doConnect( final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise connectPromise) {
    final Channel channel = connectPromise.channel();
    if (localAddress == null) {
        channel.connect(remoteAddress, connectPromise);
    } else {
        channel.connect(remoteAddress, localAddress, connectPromise);
    }
}
// AbstractChannel 将 connect() 方法委托给 pipeline 实现
// DefaultChannelPipeline 实现调用 tail.connect(), 会经过所有ChannelHandler 最后到达HeadContext. 调用 Unsafe.connect 方法

```
