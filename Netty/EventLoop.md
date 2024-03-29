# EventLoop
![EventLoop类图|600](./NioEventLoop.png)
处理`Channle`的所有IO操作, 一个 EventLoop 实例通常会处理多个`Channel`, 但这可能取决于实现细节和内部结构(netty兼容一个连接一个线程的形式).
从类图上可以看到, EventLoop 接口是继承自 `EventExecutorGroup`.
就像 `EventLoopGroup` 接口继承自 `EventExecutorGroup` 一样 在向上可以看到 JDK 定义的 `ScheduledExecutorService`, `ExecutorService`, `Executor` 等接口

实现也是从JDK提供的`AbstractExecutorService` 继承而来的

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

## DefaultEventLoop

可以看到 run 方法的实现非常简单. - 调用 takeTask() 获得一个任务 - 执行 runnable - 更新最后执行时间 - 判断当前是否收到了shutdown请求, 如果有break出循环

``` java
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
