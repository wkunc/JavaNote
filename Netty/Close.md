## Close 流程

目前发现只有调用方会触发close事件 调用 channel.close() 或者 channelContext.close() 触发

channel.closeFuture()上添加的listener会在调用doClose()方法时被调用
实际上在调用 javachannel().close()方法之前 然后调用fireChannelInactiveAndDeregister().
而fireChannelInactiveAndDeregister()方法先调 doDeregister() 方法完成了实际上的取消注册动作然后 fireChannelInactive(),
fireChannelUnregistered()

在HeadContenxt.channelUnregistered()里最后会将handler按照顺序删除,并通知handler.

主动调用 `channel.close()` `channelContext.close()` 方法, 执行流程和上面一样.但是有close()事件. 并且也不是在 `read`
过程中发现并调用close()
