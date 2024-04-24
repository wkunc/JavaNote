# Reactor

它的API和Java11中的标准Flow接口完全一致(可能是这个框架的接口被采纳标准化了)

Publisher 是生产者, 推送者, 被观察者 是数据源.
Subscriber 是消费者, 是订阅者, 观察者, 是数据消费的地方.

Subscription 是生产者给消费者的标志, 当订阅者订阅推送者时,
推送者给订阅者一个标志.

# Publisher

对应Java 11 中的 Flow.Publisher接口

```java
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}

public static interface Publisher<T> {
    public void subscribe(Subscriber<? super T> sub);
}
```

# Subscriber

对应 Flow.Subscriber 接口

```java
public interface Subscriber<T> {
    public void onSubscribe(Subscription);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}

public static interface Subscriber<T> {
    public void onSubscribe(Subscription subscription);
    public void onNext(T item);
    public void onError(Throwable throwable);
    public void onConmplete();
}
```

# Subscription

```java
public interface Subscription {
    public void request(long n);
    public void cancel();
}
public static interface Subscription {
    public void request(long n);
    public void cancel();
}
```

# Processor

```java
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {

}

public static interface Processor<T, R> extends Subscriber<T>, Publisher<R> {

}
```

