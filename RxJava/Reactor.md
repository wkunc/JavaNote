# 源码分析

## Reactor如何实现reactive

```java
Flux.just(1, 2, 3)
    .subscribe(System.out::println);
```

```java
public static <T> Flux<T> fromArray(T[] array) {
    if (array.length == 0) {
        return empty();
    }
    if (array.length == 1) {
        return just(array[0]);
    }
    return onAssembly(new FluxArray<>(array));
}
```

```java
	public final Disposable subscribe(
			@Nullable Consumer<? super T> consumer,
			@Nullable Consumer<? super Throwable> errorConsumer,
			@Nullable Runnable completeConsumer,
			@Nullable Context initialContext) {
		return subscribeWith(new LambdaSubscriber<>(consumer, errorConsumer,
				completeConsumer,
				null,
				initialContext));
	}
```


```java
final class FluxArray<T> extends Flux<T> implements Fuseable, SourceProducer<T> {

	final T[] array;

	@SafeVarargs
	public FluxArray(T... array) {
		this.array = Objects.requireNonNull(array, "array");
	}

	@SuppressWarnings("unchecked")
	public static <T> void subscribe(CoreSubscriber<? super T> s, T[] array) {
		if (array.length == 0) {
			Operators.complete(s);
			return;
		}
		if (s instanceof ConditionalSubscriber) {
			s.onSubscribe(new ArrayConditionalSubscription<>((ConditionalSubscriber<? super T>) s, array));
		}
		else {
			s.onSubscribe(new ArraySubscription<>(s, array));
		}
	}

	@Override
	public void subscribe(CoreSubscriber<? super T> actual) {
		subscribe(actual, array);
	}


	@Override
	public Object scanUnsafe(Attr key) {
		if (key == Attr.BUFFERED) return array.length;
		if (key == Attr.RUN_STYLE) return Attr.RunStyle.SYNC;
		return SourceProducer.super.scanUnsafe(key);
	}

    // 静态内部类实现了 SynchronousSubscription
	static final class ArraySubscription<T>
			implements InnerProducer<T>, SynchronousSubscription<T> {

		final CoreSubscriber<? super T> actual;

		final T[] array;

		int index;

		volatile boolean cancelled;

        // 当前下游需求的消息数量
		volatile long requested;
		@SuppressWarnings("rawtypes")
		static final AtomicLongFieldUpdater<ArraySubscription> REQUESTED =
				AtomicLongFieldUpdater.newUpdater(ArraySubscription.class, "requested");

		ArraySubscription(CoreSubscriber<? super T> actual, T[] array) {
			this.actual = actual;
			this.array = array;
		}

		@Override
		public void request(long n) {
			if (Operators.validate(n)) {
                // 更新requested数字
				if (Operators.addCap(REQUESTED, this, n) == 0) {
                    // 如果发现传入的是Long.MAX_VALUE. 就意味着不会堵塞
					if (n == Long.MAX_VALUE) {
						fastPath();
					}
					else {
						slowPath(n);
					}
				}
			}
		}

		void slowPath(long n) {
			final T[] a = array;
			final int len = a.length;
			final Subscriber<? super T> s = actual;

			int i = index;
			int e = 0;

			for (; ; ) {
				if (cancelled) {
					return;
				}

				while (i != len && e != n) {
					T t = a[i];

					if (t == null) {
						s.onError(new NullPointerException("The " + i + "th array element was null"));
						return;
					}

					s.onNext(t);

					if (cancelled) {
						return;
					}

					i++;
					e++;
				}

				if (i == len) {
					s.onComplete();
					return;
				}

				n = requested;

				if (n == e) {
					index = i;
					n = REQUESTED.addAndGet(this, -e);
					if (n == 0) {
						return;
					}
					e = 0;
				}
			}
		}

		void fastPath() {
			final T[] a = array;
			final int len = a.length;
			final Subscriber<? super T> s = actual;

			for (int i = index; i != len; i++) {
				if (cancelled) {
					return;
				}

				T t = a[i];

				if (t == null) {
					s.onError(new NullPointerException("The " + i + "th array element was null"));
					return;
				}

                // 调用订阅者的onNext() 方法, 也就是LambdaSubscriber.onNext
				s.onNext(t);
			}
			if (cancelled) {
				return;
			}
			s.onComplete();
		}

		@Override
		public void cancel() {
			cancelled = true;
		}

		@Override
		@Nullable
		public T poll() {
			int i = index;
			T[] a = array;
			if (i != a.length) {
				T t = a[i];
				Objects.requireNonNull(t);
				index = i + 1;
				return t;
			}
			return null;
		}

		@Override
		public boolean isEmpty() {
			return index == array.length;
		}

		@Override
		public CoreSubscriber<? super T> actual() {
			return actual;
		}

		@Override
		public void clear() {
			index = array.length;
		}

		@Override
		public int size() {
			return array.length - index;
		}

		@Override
		@Nullable
		public Object scanUnsafe(Attr key) {
			if (key == Attr.TERMINATED) return isEmpty();
			if (key == Attr.BUFFERED) return size();
			if (key == Attr.CANCELLED) return cancelled;
			if (key == Attr.REQUESTED_FROM_DOWNSTREAM) return requested;

			return InnerProducer.super.scanUnsafe(key);
		}
	}

}
```

```java
final class LambdaSubscriber<T>
		implements InnerConsumer<T>, Disposable {

	final Consumer<? super T>            consumer;
	final Consumer<? super Throwable>    errorConsumer;
	final Runnable                       completeConsumer;
	final Consumer<? super Subscription> subscriptionConsumer;
	final Context                        initialContext;

	volatile Subscription subscription;
	static final AtomicReferenceFieldUpdater<LambdaSubscriber, Subscription> S =
			AtomicReferenceFieldUpdater.newUpdater(LambdaSubscriber.class,
					Subscription.class,
					"subscription");

	/**
	 * Create a {@link Subscriber} reacting onNext, onError and onComplete. If no
	 * {@code subscriptionConsumer} is provided, the subscriber will automatically request
	 * Long.MAX_VALUE in onSubscribe, as well as an initial {@link Context} that will be
	 * visible by operators upstream in the chain.
	 *
	 * @param consumer A {@link Consumer} with argument onNext data
	 * @param errorConsumer A {@link Consumer} called onError
	 * @param completeConsumer A {@link Runnable} called onComplete with the actual
	 * context if any
	 * @param subscriptionConsumer A {@link Consumer} called with the {@link Subscription}
	 * to perform initial request, or null to request max
	 * @param initialContext A {@link Context} for this subscriber, or null to use the default
	 * of an {@link Context#empty() empty Context}.
	 */
	LambdaSubscriber(
			@Nullable Consumer<? super T> consumer,
			@Nullable Consumer<? super Throwable> errorConsumer,
			@Nullable Runnable completeConsumer,
			@Nullable Consumer<? super Subscription> subscriptionConsumer,
			@Nullable Context initialContext) {
		this.consumer = consumer;
		this.errorConsumer = errorConsumer;
		this.completeConsumer = completeConsumer;
		this.subscriptionConsumer = subscriptionConsumer;
		this.initialContext = initialContext == null ? Context.empty() : initialContext;
	}

	/**
	 * Create a {@link Subscriber} reacting onNext, onError and onComplete. If no
	 * {@code subscriptionConsumer} is provided, the subscriber will automatically request
	 * Long.MAX_VALUE in onSubscribe, as well as an initial {@link Context} that will be
	 * visible by operators upstream in the chain.
	 *
	 * @param consumer A {@link Consumer} with argument onNext data
	 * @param errorConsumer A {@link Consumer} called onError
	 * @param completeConsumer A {@link Runnable} called onComplete with the actual
	 * context if any
	 * @param subscriptionConsumer A {@link Consumer} called with the {@link Subscription}
	 * to perform initial request, or null to request max
	 */ //left mainly for the benefit of tests
	LambdaSubscriber(
			@Nullable Consumer<? super T> consumer,
			@Nullable Consumer<? super Throwable> errorConsumer,
			@Nullable Runnable completeConsumer,
			@Nullable Consumer<? super Subscription> subscriptionConsumer) {
		this(consumer, errorConsumer, completeConsumer, subscriptionConsumer, null);
	}

	@Override
	public Context currentContext() {
		return this.initialContext;
	}

    // 和 onNext, onComplete, onError 类似, 逻辑都是委托给传入的lambda
    // 重点是 subscriptionConsumer 实际上以及不推荐传入了(因为用户通常会忘记调用 Subscription.request(n) 方法)
    // 所以这里实际上就是调用传入的 Subscription.request(Long.MAX_VALUE)
    // 而在我们的例子中, Subscription 是FluxArray.ArraySubscription
	@Override
	public final void onSubscribe(Subscription s) {
		if (Operators.validate(subscription, s)) {
			this.subscription = s;
			if (subscriptionConsumer != null) {
				try {
					subscriptionConsumer.accept(s);
				}
				catch (Throwable t) {
					Exceptions.throwIfFatal(t);
					s.cancel();
					onError(t);
				}
			}
			else {
				s.request(Long.MAX_VALUE);
			}
		}
	}

	@Override
	public final void onComplete() {
		Subscription s = S.getAndSet(this, Operators.cancelledSubscription());
		if (s == Operators.cancelledSubscription()) {
			return;
		}
		if (completeConsumer != null) {
			try {
				completeConsumer.run();
			}
			catch (Throwable t) {
				Exceptions.throwIfFatal(t);
				onError(t);
			}
		}
	}

	@Override
	public final void onError(Throwable t) {
		Subscription s = S.getAndSet(this, Operators.cancelledSubscription());
		if (s == Operators.cancelledSubscription()) {
			Operators.onErrorDropped(t, this.initialContext);
			return;
		}
		if (errorConsumer != null) {
			errorConsumer.accept(t);
		}
		else {
			Operators.onErrorDropped(Exceptions.errorCallbackNotImplemented(t), this.initialContext);
		}
	}

	@Override
	public final void onNext(T x) {
		try {
			if (consumer != null) {
				consumer.accept(x);
			}
		}
		catch (Throwable t) {
			Exceptions.throwIfFatal(t);
			this.subscription.cancel();
			onError(t);
		}
	}

	@Override
	@Nullable
	public Object scanUnsafe(Attr key) {
		if (key == Attr.PARENT) return subscription;
		if (key == Attr.PREFETCH) return Integer.MAX_VALUE;
		if (key == Attr.TERMINATED || key == Attr.CANCELLED) return isDisposed();
		if (key == Attr.RUN_STYLE) return Attr.RunStyle.SYNC;

		return null;
	}


	@Override
	public boolean isDisposed() {
		return subscription == Operators.cancelledSubscription();
	}

	@Override
	public void dispose() {
		Subscription s = S.getAndSet(this, Operators.cancelledSubscription());
		if (s != null && s != Operators.cancelledSubscription()) {
			s.cancel();
		}
	}
}

```

# 特殊要点

## GroupedFlux只能被订阅一次
groupBy()分组方法可以产生一个`Flux<GroupedFlux>`, GroupFlux 代表每个分组的流.
且这个`GroupedFlux`只允许订阅一次

```java
	static final class UnicastGroupedFlux<K, V> extends GroupedFlux<K, V>
			implements Fuseable, QueueSubscription<V>, InnerProducer<V>

		@Override
		public void subscribe(CoreSubscriber<? super V> actual) {
        // 检查并记录订阅次数
			if (once == 0 && ONCE.compareAndSet(this, 0, 1)) {
				actual.onSubscribe(this);
				ACTUAL.lazySet(this, actual);
				drain();
			}
			else {
				Operators.error(actual, new IllegalStateException(
						"GroupedFlux allows only one Subscriber"));
			}
		}
    }
```

但是有时代码就是需要对这个group进行多次map
我们可以使用 Flux.publish(Function<? super Flux<T>, ? extends Publisher<? extends R>> transform) 方法来完成
避免产生多次订阅
```java
	public final <R> Flux<R> publish(Function<? super Flux<T>, ? extends Publisher<?
			extends R>> transform) {
		return publish(transform, Queues.SMALL_BUFFER_SIZE);
	}

    /**
     在函数持续时间内共享一个序列,
     该函数可以根据需要转换和多次使用该序列, 而不会导致对上游的多个订阅.
    */
	public final <R> Flux<R> publish(Function<? super Flux<T>, ? extends Publisher<?
			extends R>> transform, int prefetch) {
		return onAssembly(new FluxPublishMulticast<>(this, transform, prefetch, Queues
				.get(prefetch)));
	}
```

### 代码示例
一个人有多张卡, 且一个人有多个设备权限. 所以需要生成的指令是卡数量和设备数量的乘积
```java
    Flux<Permission> just = Flux.just(
            new Permission(Type.user, "1"),
            new Permission(Type.visitor, "2"),
            new Permission(Type.depart, "3"),
            new Permission(Type.user, "4"),
            new Permission(Type.visitor, "5"),
            new Permission(Type.depart, "6"),
    );
    Flux<Card> cards = ....
    just.groupBy(p -> p.type())
        .flatMap(g -> {
            switch (g.key()) {
                case user:
                    // 利用 publish() 方法避免产生多次订阅
                    return g.publish(list -> user(list, cardNumber));
                    // return user(g);
                default:
                    return Flux.empty();
            }
        })
        .subscribe();

    public static Flux<Void> user(Flux<Permission> permission, Flux<String> cardNumber) {
        return cardNumber.flatMap(card -> {
            return u.flatMap(p -> {
                logger.info("permission: {}, card {}", p, card);
                return Flux.<Void>empty();
            });
        })
    }
```

## flatMap() Vs concatMap()
[](https://stackoverflow.com/questions/71971062/whats-the-difference-between-flatmap-flatmapsequential-and-concatmap-in-project)


