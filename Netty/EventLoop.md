# EventExecutorGroup


重新声明了那些返回值是 java.util.concurrent.Future 的方法.
将返回值声明为netty提供的增强的 Future 类


```java
interface EventExcutorGroup extends ScheduledExecutorService  {

    // 额外增加优雅的关闭方法, 并将 ExecutorServer.shutdown() 标记为过时
    boolean isShuttingDown();
    Future<?> shutdownGracefully(long quitePeriod, long timeout, TimeUnit unit);
    Future<?> shutdownGracefully();
    Future<?> terminationFutre();

    // 核心
    EventExecutor next();
}
```

# AbstractEventExecutorGroup

将 submit(), schedule(), scheduleAtFixedRate(), scheduleWithFixedDelay(),
invokeAll(), invokeAny() 等方法传递给 next() 方法获取的child 执行器去执行.

以及 shutdownGracefully() 的默认实现


# MultithreadEventExecutorGroup
用多线程处理任务的基本实现

主要定义了初始化流程. 实现关闭,终结系列方法, 以及next()
```java
public abstract class MultithreadEventExecutorGroup extends AbstractEventExecutorGroup {

    private final EventExecutor[] children;
    private final Set<EventExecutor> readonlyChildren;
    private final AtomicInteger terminatedChildren = new AtomicInteger();
    private final Promise<?> terminationFuture = new DefaultPromise(GlobalEventExecutor.INSTANCE);
    private final EventExecutorChooserFactory.EventExecutorChooser chooser;


    protected MultithreadEventExecutorGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
        this(nThreads, threadFactory == null ? null : new ThreadPerTaskExecutor(threadFactory), args);
    }


    protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
        this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
    }

    // nThreads 需要的线程数量
    // 内部的Executor
    // EventExecutor选择器工厂. 
    // 默认会根据线程数量返回不同的 Chooser (通用的和2的n次方版本)
    protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        checkPositive(nThreads, "nThreads");

        // 如果 executor 为null使用默认的 Executor.
        // 简单的使用给定的 ThreadFactory 创建线程然后start的Executor
        // TODO DefaultThreadFactory, FastThreadLocalThread. 
        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        // 初始化子 EventExecutor 数组长度
        children = new EventExecutor[nThreads];

        // 循环创建每一个 EventExecutor
        // 如果某一个子 EventExecutor 创建失败了
        // 通过调用 EventExecutor.shutdownGracefully() 关闭之前创建的 
        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                // 调用抽象方法, 将 Executor 和 args 参数传入
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        // 创建 Chooseer. 默认通过数组长度确定
        chooser = chooserFactory.newChooser(children);

        // 创建子EventExector终止事件监听器.
        // 终结后增加 terminatedChildren 计数器, 
        // 当计数器和子EventExector数组长度一样相等时,
        // 意味着所有的Children都已经关闭了,
        // 将这个 EventExectoryGroup 的termainationFuture设置为true
        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }

    protected ThreadFactory newDefaultThreadFactory() {
        return new DefaultThreadFactory(getClass());
    }

    protected abstract EventExecutor newChild(Executor executor, Object... args) throws Exception;

}
```

