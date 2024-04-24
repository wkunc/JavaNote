# Channel

通道表示与诸如硬件设备, 文件, 网络套接字 的实体的抽象.

通道只有两种状态, 开放或关闭. 通道在创建的时候保持打开.
一旦关闭, 它将保持关闭状态.
调用一个关闭状态的Channel上的 I/O 操作都会抛出 ClosedChannelException.

可以调用 isOpen() 方法来测试Channel是否打开

```java
public interface Channel extend Closeable() {
    public boolean isOpen();
    public void close() throws IOException;
}
```

----
![](../imgs/Channel.PNG)

# FileChannel

![](../imgs/FileChannel.PNG)

```java
public interface ReadableByteChannel extends Channel {
    public int read(ByteBuffer dst) throws IOException;
}
```

