# Simple Ways to Create a Flux or Mono and Subscribe to It
简单的方式创建一个 Flux or Mono 并且订阅(Subscribe) 它
-----------

开始使用 Flux 和 MOno 的最简单的方法是使用在各自类中发现的众多工厂方法之一

1. empty()
2. just()
3. fromIterable()
4. range()
5. interval()


## subscribe method

```java
// 1. 订阅并触发序列
subscribe(); 

// 2. 对每个产生的值做一些什么
subscribe(Consumer<? super T> consumer); 

// 3. deal with values but also react to an error.
subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer); 

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer); 

subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer,
          Consumer<? super Subscription> subscriptionConsumer);
```

## on Backpressure and Ways to Reshape Requests


#4.4 Programmatically creating a sequence

## 4.4.1 Synchronous generate

The simplest from of programmatic creation of a Flux is the generate method, which taskes a generator function.
最简单的编程式创建 Flux 的方式是 generate() 方法.

这用于同步 and 一对一发射, 这意味着 sink(接收器)是一个 SynchronousSink(同步接收器).
并且next() 方法在每次回调调用中最多只能调用一次.


