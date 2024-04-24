#Netty

# ByteBuf

网络数据的基本单位总是字节. Java NIO 提供了*ByteBuffer*作为它的字节容器,
但是这个类使用起来过于复杂, 而且也有些繁琐.

Netty 的*ByteBuffer*的替代品是*ByteBuf*, 一个强大实现,
即解决了JDK API 的局限性, 又为网络应用程序的开发者提供了更好的API.

# ByteBuf 的API

Netty 的数据处理API通过两个组件暴露----abstract class ByteBuf 和 interface ByteBufHolder.

下面是*ByteBuf*API的一些优点:

* 它可以被用户自定义的缓冲区扩展.
* 通过内置的复合缓冲区类型实现了透明的*零拷贝*
* 容量可以按需增长(类似于JDK的StringBuilder)
* 在读和写这两种模式之间切换不需要调用 ByteBuffer.filp() 方法
* 读和写使用了不同的索引
* 支持方法的链式调用
* 支持引用计数
* 支持池化

其他类可用于管理ByteBuf实例的分配,
以及执行各种针对数据容器本身和它所持有的数据操作.

# ByteBuf 类----Netty的数据容器

因为所有的网络通信都涉及字节序列的移动, 所以高效易用的数据结构明显必不可少.

ByteBuf 通过使用不同的索引来简化对它包含的数据进行访问.

## 它是如何工作的

ByteBuf 维护了两个不同的索引: 一个用于读取, 一个用于写入.
当你从ByteBuf读取时, 它的 readerIndex 将会被递增已经被读取的字节数.
同样, 当你写入ByteBuf时, 它的 writerIndex 也会被递增

名称以 read 或 write 开头的方法ByteBuf 方法, 将会推进对应的索引.
而以 set 或者 get 开头的操作则不会. 后面的这些方法将在作为一个参数

### ByteBuf 的使用模式

在使用 Netty 时, 你将遇到几种常见的围绕 ByteBuf 而构建的使用模式.

1. 堆缓冲区
   最常用的 ByteBuf 模式是将数据存储在JVM的堆空间中.
   这种模式被称为*支撑数组(backing array)*,
   它能在没有使用池化的情况下提供快速的分配和释放.

```java
Byte heapBuf = ...;
if (heapBuf.hasArray()) {
    byte[] array = heapBuf.array();
    int offset = heapBuf.arrayOffset() + heapBuf.readerIndex();
    int length = heapBuf.readableBytes();
    handleArray(array, offset, length)
}
```

2. 直接缓冲区
   *直接缓冲区*是另一种ByteBuf模式. 我们期望用于对象创建的内存分配永远来自于堆中.
   但这并不是必须的----NIO在JDK1.4中引入的ByteBuffer类允许JVM实现通过本地调用
   来分配内存. 这主要是为了避免每次调用本地I/O操作之前(或者之后)
   将缓冲区的内容复制到一个中间缓冲区.

ByteBuffer的Javadoc明确指出: 直接缓冲区的内容将驻流在常规的会被GC的堆之外.
这也解释了为何*直接缓冲区*对于网络数据传输的选择.
如果你的数据包含在一个*堆缓冲区*中, 那么事实上, 在通过套接字发送它之前.
将会在内部把你的缓冲区复制到一个直接缓冲区中.

*直接缓冲区*的主要确定是,相对于基于堆的缓冲区, 它们分配和释放都比较昂贵.
如果你正在处理遗留代码, 你也可能会遇到另一个缺点: 因为数据不是堆上,
所以不得不进行一次复制.

3. 复合缓冲区
   最后一种模式使用的是*复合缓冲区*, 它为多个ByteBuf提供一个聚合视图.
   在这里你可以工具需要添加或者删除ByteBuf实例,
   这是一个JDK的ByteBuffer实现完全缺失的特性.

Netty 通过一个ByteBuf子类, CompositeByteBuf 实现了这个模式,
它提供了一个将多个缓冲区标识为单个合并缓冲区的虚拟表示.

# 字节级操作

## 随机访问索引

## 顺序访问索引

## 可丢弃字节

## 可读字节

## 可写字节

## 索引管理

# ByteBuf 分配

管理ByteBuf实例的不同方式

## 按需分配: ByteBufAllocator 接口

为了降低分配和释放内存的开销,
Netty 通过 interface ByteBufAllocator 实现了 ByteBuf 的池化,
它可以用来分配我们所描述的任意类型的 ByteBuf 实例.
使用池化是特定于应用程序的决定, 其并不会以任何方式改变ByteBuf API的语义.
