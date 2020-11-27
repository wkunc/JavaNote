# EventLoop 和线程模型

# EventLoop 接口

运行任务来处理啊和连接的生命周期内发送的事件是任何网络框架的基本功能.
与之相应的编程上的构造通常被称为 *事件循环*

Netty 使用了 io.netty.channel.EentLoop 接口来适配术语

```java
while(!terminated) {
    // 阻塞, 知道有事件已经就绪可被运行
    List<Runable> readyEvents = blockUntilEventsReady();
    // 循环遍历,并处理所有事件
    for (Runnable ev : readyEvents) {
        ev.run();
    }
}
```
