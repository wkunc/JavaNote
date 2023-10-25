# EventExecutorGroup

重新声明了 java.util.concurrent.ScheduledExecutorService 接口上那些返回值是 java.util.concurrent.Future 的方法.
将返回值声明为Netty提供的增强的 io.netty.util.concurrent.Future 类

``` java
interface EventExcutorGroup extends ScheduledExecutorService  {

    // 额外增加优雅的关闭方法, 并将 ExecutorServer.shutdown() 标记为过时
    boolean isShuttingDown();
    Future<?> shutdownGracefully(long quitePeriod, long timeout, TimeUnit unit);
    Future<?> shutdownGracefully(); Future<?> terminationFutre();

    // 核心
    EventExecutor next();
}
```

## AbstractEventExecutorGroup

将 `submit()`, `schedule()`, `scheduleAtFixedRate()`, `scheduleWithFixedDelay()`, `invokeAll()`, `invokeAny()` 等方法传递给 `next()` 方法获取的child EventExecutor 执行器去执行.
以及提供 `shutdownGracefully()` 的默认实现委托给另一个完整参数的 `shutdownGracefully()` 方法

``` java
@Override
public Future<?> shutdownGracefully() {
    return shutdownGracefully(DEFAULT_SHUTDOWN_QUIET_PERIOD, DEFAULT_SHUTDOWN_TIMEOUT, TimeUnit.SECONDS);
}
```

# MultithreadEventExecutorGroup

用多线程处理任务的基本实现

主要定义了初始化流程. 实现关闭,终结系列方法, 以及next()

``` java
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

    // 交给子类实现真正的创建一个EventLoop.
    protected abstract EventExecutor newChild(Executor executor, Object... args) throws Exception;

}
```

## MultithreadEventLoopGroup

实现了 EventLoopGroup 接口, 其实也就是把 register(Channel channel) 方法委托给child,

## NioEventLoopGroup

实现了 newChild() 方法 NioEventLoop 的初始化逻辑

``` java
  // 最长的构造器, MultithreadEventExecutorGroup 声明的构造器只有3个明确参数, 
  // 也就是这里的 nThreads, executor, chooserFactor. 
  // 剩下的其他参数说明都是可变参数 Object... args的一部分, 也就是说会传递给 newChild() 方法
  public NioEventLoopGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory,
                         SelectorProvider selectorProvider,
                         SelectStrategyFactory selectStrategyFactory,
                         RejectedExecutionHandler rejectedExecutionHandler,
                         EventLoopTaskQueueFactory taskQueueFactory,
                         EventLoopTaskQueueFactory tailTaskQueueFactory) {

    super(nThreads, executor, chooserFactory, selectorProvider, selectStrategyFactory,
            rejectedExecutionHandler, taskQueueFactory, tailTaskQueueFactory);
  }

  // args 可以通过构造器知道内容顺序都是啥
@Override
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    SelectorProvider selectorProvider = (SelectorProvider) args[0];
    SelectStrategyFactory selectStrategyFactory = (SelectStrategyFactory) args[1];
    RejectedExecutionHandler rejectedExecutionHandler = (RejectedExecutionHandler) args[2];
    EventLoopTaskQueueFactory taskQueueFactory = null;
    EventLoopTaskQueueFactory tailTaskQueueFactory = null;

    int argsLength = args.length;
    if (argsLength > 3) {
        taskQueueFactory = (EventLoopTaskQueueFactory) args[3];
    }
    if (argsLength > 4) {
        tailTaskQueueFactory = (EventLoopTaskQueueFactory) args[4];
    }
    return new NioEventLoop(this, executor, selectorProvider,
            selectStrategyFactory.newSelectStrategy(),
            rejectedExecutionHandler, taskQueueFactory, tailTaskQueueFactory);
}
```
