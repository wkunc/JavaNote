# Executor

```java
public interface Executor {

    void execute(Runnable command);

}
```

```java
public interface ExecutorService extends Executor {
    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
            throws InterruptedException;

    //
    <T> Future<T> submit(Callable<T> task);
    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
            throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                    long timeout, TimeUnit unit)
            throws InterruptedException;

    <T> T invokeAll(Collection<? extends Callable<T>> tasks)
            throws InterruptedException, ExecutionExpection;

    <T> T invokeAll(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionExpection, TimeoutException;
}
```


