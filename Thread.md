# Object.wait(), notify() notifyAll()
java中每个对象都有一个对应的锁, synchronized 实例方法就是去获取这个锁.

Object 上就定义了这三个方法. 它们的作用和对象锁有关(Object Monitor).

Object Monitor 可以认为存在一个进入队列(entry queue) 一个等待队列(waiting queue),一个锁(lock)组成.

当我们使用 synchronized 方式获取锁时意味着当前线程在企图获取对应的锁对象, 此时可以认为线程加入了 entry queue 中.
多个线程竞争时, 只会有一个线程获取到锁对象(synchronized 锁是互斥锁).
获取锁成功的线程可以继续执行, 而其他竞争锁失败的线程会挂起(Waiting). 此时线程仍然存在与 entry queue 中.

```java
    /**
     * wait() 的作用就是, 让当前线程临时释放获取的锁并进入wait queue. 此时线程状态是 Wait
     * 此时线程只有等到 notified 或者 interrupted. 或者时间到了
     * 三种情况才会被唤醒.
     * 何为唤醒, 我理解为是从 wait queue 移动到 waiting queue 中.此时线程状态变为 Waiting
     * 中断的情况除外. 如果没有进行捕获异常的话, 线程会由于抛出异常而结束
     * 
     * 所以这个方法的调用前提是, 当前线程获取了这个对象对应的Lock. 
     * 否则就会throw IllegalMonitorStateException.
     *
     */
    public final native void wait(long timeoutMillis) throws InterruptedException;
```

根据wait()方法描述, 我们可以做一个实验. 看看如果不调用notify()会发生什么
```java


```
结果是如果没有调用notify()的话. 调用过wait()方法的线程永远处于wait状态, 即使当前已经没有任何线程持有对象锁.
线程也不会被唤醒去竞争锁. 因为线程状态是wait, 此时系统不会去调度这个线程. 也就是根本不会占用任何cpu时间片

当我们调用notfiy()时,对应的对象锁的wait queue中的某一个线程将会被唤醒
也就加入到 waiting queue 中. 此时系统会有机会调用到该线程.

notfiy() 只会唤起一个线程, 如果之前有多个线程调用过同一个对象的wait()方法, 此时waiting queue中有两个线程.
该方法只会随机挑选一个线程(ps: jdk1.6是添加了VM option. --XX:SyncKnobs 可以用设置唤醒线程的选择策略. Java13时Deprecated了)

notifyAll() 就是字面意思,唤醒所有waiting queue中的线程.
被唤醒的线程会竞争同一个锁, 获取锁后重新从 wait() 方法之后开始继续执行

```java
public final native void notify();

public final native void notifyAll();
```

# Thread.join()
Thread.join(), 方法通常用于让当前线程挂起等待目标线程完成后在继续执行.


分析 Thread.join() 代码
```java
    // 首先这个方法是一个 synchronized 方法, 意味着调用线程获取了目标线程的ObjectMonitor
    // 然后会调用 wait() 方法.
    // 而调用wait() 方法会让出对象锁, 并且使调用线程进入Wait状态
    // 根据之前的分析, 我们知道此时需要调用对应对象的 notify()或者notifyAll() 方法.
    // 才能唤醒调用线程.
    // 通过方法注释, 我们可以知道. JVM 在线程结束的时候会调用 notifyAll() 唤起线程
    // (ps: 我估计是 jdk/src/hotspot/share/runtime/javaThread.cpp ensure_join() 方法)
    // NOTE: 方法提示我们不要手动调用 Thread类型对象的 wait(), notify(), notifyAll() 方法
    // join 方法里的循环条件中判断了当前线程状态, 如果线程未结束, 即使手动调用 notifyAll() 方法唤起join的调用线程
    // 也会因为循环逻辑继续调用 wait() 回到Wait状态
    public final synchronized void join(final long millis)
    throws InterruptedException {
        if (millis > 0) {
            if (isAlive()) {
                final long startTime = System.nanoTime();
                long delay = millis;
                do {
                    wait(delay);
                } while (isAlive() && (delay = millis -
                        TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)) > 0);
            }
        } else if (millis == 0) {
            while (isAlive()) {
                wait(0);
            }
        } else {
            throw new IllegalArgumentException("timeout value is negative");
        }
    }
```

# interrupt 线程中断



