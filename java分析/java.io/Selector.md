# Selector
一个Selector可以处理多个通道
只有非阻塞模式下的Channel才能与Selector配合使用.
这意味着FileChannel不能和Selector一起工作
(因为FileChannel不能切换到非阻塞模式)


```java
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```

register()方法的第二个参数是一个 "interest集合"
意思是在通过Selector监听Channel时对什么事情感兴趣.
1. Connect (某个channel成功连接到另一个服务器称为连接就绪)
2. Accept (一个serversocketchannel 准备好接收进入的连接"接受就绪")
3. Read ()
4. Write ()

SelectionKey.OP\_CONNECT
SelectionKey.OP\_ACCEPT
SelectionKey.OP\_READ
SelectionKey.OP\_WRITE


