# ChannelHandler 家族

## Channel 生命周期

Channel 定义了一组和 ChannelInBoundHandler API
密切相关的简单但功能强大的状态模型.

下面列出了Channel的这4个状态.

|状态| 描述|
|----|-----|
|ChannelUnregistered | Channel 已经被创建, 但还未注册到EventLoop
|ChannelRegistered | Channel 已经被注册到了 EventLoop
|ChannelActive | Channel 处于活动状态(已经连接到远程节点) 可以接收和发送数据
|ChannelInactive | Channel 没有连接到远程节点

## ChannelHandler 的生命周期

interface ChannelHandler定义的生命周期操作,
在 ChannelHandler 被添加到 ChannelPipeline 中或者被从 ChannelPipeline 中移除
时会调用这些操作. 这些方法中的每一个都接受一个 ChannelHandlerContext 参数

|类型|描述|
|----|----|
|handlerAdded|当把ChannelHandler添加到ChannelPipeline中时被调用
|handlerRemoved|当从ChannelPipeline中移除ChannelHandler时被调用
|execptionCaught|当处理过程中在ChannelPipeline中有错误产生时被调用.

# ChannelPipeline 接口
如果你任务 ChannelPipline 是一个拦截流经 Channel 的
入站和出站事件的ChannelHandler实例链,
那么就很容易看出这些ChannelHandler之间的交互是如何组成一个应用程序数据和事件处理逻辑的核心的.

每一个新创建的Channel都将会被分配一个新的ChannelPipeline.
这项关联是永久性的; Channel 既不能附加另外一个*ChannelPipeline*, 也不能分离其当前的.
在 Netty 组件的生命周期中, 这是一项固定的操作不需要开发人员的任何干预.

根据事件的起源, 事件将会被ChannelInboundHandler 或者 ChannelOutboundHandler处理.
随后,通过调用 ChannelHandlerContext 实现, 它将会被转发给同一超类型的下一个 ChannelHandler.


