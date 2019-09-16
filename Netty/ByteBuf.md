# ByteBuf
网络数据的基本单位总是字节. Java NIO 提供了*ByteBuffer*作为它的字节容器,
但是这个类使用起来过于复杂, 而且也有些繁琐.

Netty 的*ByteBuffer*的替代品是*ByteBuf*, 一个强大实现,

# ByteBuf 的API
Netty 的数据处理API通过两个组件暴露----abstract class ByteBuf 和 
interface ByteBufHolder.
下面是*ByteBuf*API的一些优势
* 它可以被用户自定义的缓冲区扩展.
* 通过内置的复合缓冲区类型实现了透明的*零拷贝*
* 容量可以按需增长(类似于JDK的StringBuilder)
* 在读和写这两种模式之间切换不需要调用 ByteBuffer.filp() 方法
* 读和写使用了不同的索引
* 支持方法的链式调用
* 支持引用计数
* 支持池化
