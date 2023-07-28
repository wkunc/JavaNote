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

    // 交给子类实现真正的创建一个EventLoop.
    protected abstract EventExecutor newChild(Executor executor, Object... args) throws Exception;

}
```


## MultithreadEventLoopGroup
  实现了 EventLoopGroup 接口, 其实也就是把 register(Channel channel) 方法委托给child,

## NioEventLoopGroup
  实现了 newChild() 方法 NioEventLoop 的初始化逻辑
```java
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


# EventLoop
  处理Channle的所有IO操作, 一个 EventLoop 实例通常会处理多个 Channel, 但这可能取决于实现细节和内部结构(netty兼容一个连接一个线程的形式)
  从类图上可以看到, EventLoop 接口是继承自 EventExecutorGroup. 就像 EventLoopGroup 接口继承自 EventExecutorGroup 一样
  在向上可以看到 Jdk 定义的 ScheduledExecutorService, ExecutorService, Executor 等接口
  
  实现也是从JDK提供的AbstractExecutorService 继承而来的

## AbstractEventExecutor
``` java
public abstract class AbstractEventExecutor extends AbstractExecutorService implements EventExecutor {

}
```
## AbstractScheduledEventExecutor
   schedule 方法的相关实现

## SingleThreadEventExecutor
   ``` java
   private void execute(Runnable task, boolean immediate) {
     boolean inEventLoop = inEventLoop();
     // 将提交的task添加到任务队列中
      addTask(task);
      // 如果调用 execute() 的线程不是, EventLoop 线程.
      // 说明可能是首次调用, 调用startThread()方法, 启动 EventLoop
     if (!inEventLoop) {
         startThread();
        // 发现当前Executor已经关闭时的处理, 不是核心逻辑
         if (isShutdown()) {
             boolean reject = false;
             try {
                 if (removeTask(task)) {
                     reject = true;
                 }
             } catch (UnsupportedOperationException e) {
                 // The task queue does not support removal so the best thing we can do is to just move on and
                 // hope we will be able to pick-up the task before its completely terminated.
                 // In worst case we will log on termination.
             }
             if (reject) {
                 reject();
             }
         }
    }

      if (!addTaskWakesUp && immediate) {
          wakeup(inEventLoop);
      }
   }

   // 如果EventLoop状态是 ST_NOT_STARTED (未启动) 的情况下尝试启动
   // 这里只是简单的修改 EventLoop 状态为 ST_STARTED 已启动
   // 并且这里使用JDK提供的原子修改操作, 这样多个线程同时调用 execute() 方法时, 保证了 doStartThread() 方法只被调用一次
   private void startThread() {
    if (state == ST_NOT_STARTED) {
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            boolean success = false;
            try {
                doStartThread();
                success = true;
            } finally {
                if (!success) {
                    STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                }
            }
        }
    }
  }

   // 这个方法调用了内部的 executor 的 execute() 方法
   // 回顾 EventLoopGroup 实现, 会发现通常情况这里的 executor 是一个调用一次新建一个thread的逻辑

   // 1. 给EventLoop 绑定线程
   // 2. 调用抽象的 run() 方法(子类实现通常是个死循环, 处理任务队列)
   // 3. 编写一个 finally 块, 当run() 方法结束意味着 EventLoop 也结束了. 进行状态的改变以及清理工作
       private void doStartThread() {
        assert thread == null;
        executor.execute(new Runnable() {
            @Override
            public void run() {
         // EventLoop 绑定这个 thread 线程
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }

                boolean success = false;
                updateLastExecutionTime();
                try {
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                    for (;;) {
                        int oldState = state;
                        if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                                SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                            break;
                        }
                    }

                    // Check if confirmShutdown() was called at the end of the loop.
                    if (success && gracefulShutdownStartTime == 0) {
                        if (logger.isErrorEnabled()) {
                            logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                    SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
                                    "be called before run() implementation terminates.");
                        }
                    }

                    try {
                        // Run all remaining tasks and shutdown hooks. At this point the event loop
                        // is in ST_SHUTTING_DOWN state still accepting tasks which is needed for
                        // graceful shutdown with quietPeriod.
                        for (;;) {
                            if (confirmShutdown()) {
                                break;
                            }
                        }

                        // Now we want to make sure no more tasks can be added from this point. This is
                        // achieved by switching the state. Any new tasks beyond this point will be rejected.
                        for (;;) {
                            int oldState = state;
                            if (oldState >= ST_SHUTDOWN || STATE_UPDATER.compareAndSet(
                                    SingleThreadEventExecutor.this, oldState, ST_SHUTDOWN)) {
                                break;
                            }
                        }

                        // We have the final set of tasks in the queue now, no more can be added, run all remaining.
                        // No need to loop here, this is the final pass.
                        confirmShutdown();
                    } finally {
                        try {
                            cleanup();
                        } finally {
                            // Lets remove all FastThreadLocals for the Thread as we are about to terminate and notify
                            // the future. The user may block on the future and once it unblocks the JVM may terminate
                            // and start unloading classes.
                            // See https://github.com/netty/netty/issues/6596.
                            FastThreadLocal.removeAll();

                            STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                            threadLock.countDown();
                            int numUserTasks = drainTasks();
                            if (numUserTasks > 0 && logger.isWarnEnabled()) {
                                logger.warn("An event executor terminated with " +
                                        "non-empty task queue (" + numUserTasks + ')');
                            }
                            terminationFuture.setSuccess(null);
                        }
                    }
                }
            }
        });
      }

     /**
     * Run the tasks in the {@link #taskQueue}
     */
    protected abstract void run();
   }
```
     
## SingleThreadEventExecutor

## DefaultEventLoop
可以看到 run 方法的实现非常简单. 
- 调用 takeTask() 获得一个任务
- 执行 runnable
- 更新最后执行时间
- 判断当前是否收到了shutdown请求, 如果有break出循环
  
```java
   @Override
   protected void run() {
    for (;;) {
        Runnable task = takeTask();
        if (task != null) {
            runTask(task);
            updateLastExecutionTime();
        }

        if (confirmShutdown()) {
            break;
        }
    }
   }
```
