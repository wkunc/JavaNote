# Vertx

Vertx 作为核心类类似于 Spring的ApplicationContenxt. 
由许多的接口实现类组成比如: ClusterManager, FileSystem等等. 类似一个门面模式(Face)
这里先关注对 Netty EventLoop 的使用

```java
public class VertxImpl implements VertxInternal, MetricsProvider {

  final WorkerPool workerPool;
  final WorkerPool internalWorkerPool;
  private final VertxThreadFactory threadFactory;
  private final ExecutorServiceFactory executorServiceFactory;
  private ThreadFactory eventLoopThreadFactory;
  private final EventLoopGroup eventLoopGroup;
  private final EventLoopGroup acceptroEventLoopGroup;

  private final Transport transport;


  /**
  * 简单VertxBuilder的代码查看
  * threadFactory = VertxThreadFactory.INSTANCE;
  * executorServiceFactory = ExecutorServiceFactory.INSTANCE;
  *
  *
  *
  */
  VertxImpl(VertxOptions options, ClusterManager clusterManager,
            NodeSelector nodeSelector, VertxMetrics metrics,
            VertxTracer<?, ?> tracer, Transport transport,
            FileResolver fileResolver, VertxThreadFactory threadFactory,
            ExecutorServiceFactory executorServiceFactory) {

    // Sanity check
    if (Vertx.currentContext() != null) {
      log.warn("You're already on a Vert.x context, are you sure you want to create a new Vertx instance?");
    }

  
    Boolean useDaemonThread = options.getUseDaemonThread();
    // IO工作线程大小, 20
    int workerPoolSize = options.getWorkerPoolSize();
    // 阻塞任务线程池大小, 20
    int internalBlockingPoolSize = options.getInternalBlockingPoolSize();

  
    // EventLoop的最大执行时长, 默认 2s
    long maxEventLoopExecuteTime = options.getMaxEventLoopExecuteTime();
    TimeUnit maxEventLoopExecuteTimeUnit = options.getMaxEventLoopExecuteTimeUnit();
    ThreadFactory acceptorEventLoopThreadFactory = createThreadFactory(threadFactory, checker, useDaemonThread, maxEventLoopExecuteTime, maxEventLoopExecuteTimeUnit, "vert.x-acceptor-thread-", false);

    // 最大执行时长, 默认 60s
    TimeUnit maxWorkerExecuteTimeUnit = options.getMaxWorkerExecuteTimeUnit();
    long maxWorkerExecuteTime = options.getMaxWorkerExecuteTime();

    // 创建了两个 ExecutorService, 一个是work, 一个是internal work.
    ThreadFactory workerThreadFactory = createThreadFactory(threadFactory, checker, useDaemonThread, maxWorkerExecuteTime, maxWorkerExecuteTimeUnit, "vert.x-worker-thread-", true);
    ExecutorService workerExec = executorServiceFactory.createExecutor(workerThreadFactory, workerPoolSize, workerPoolSize);
    PoolMetrics workerPoolMetrics = metrics != null ? metrics.createPoolMetrics("worker", "vert.x-worker-thread", options.getWorkerPoolSize()) : null;
    ThreadFactory internalWorkerThreadFactory = createThreadFactory(threadFactory, checker, useDaemonThread, maxWorkerExecuteTime, maxWorkerExecuteTimeUnit, "vert.x-internal-blocking-", true);
    ExecutorService internalWorkerExec = executorServiceFactory.createExecutor(internalWorkerThreadFactory, internalBlockingPoolSize, internalBlockingPoolSize);
    PoolMetrics internalBlockingPoolMetrics = metrics != null ? metrics.createPoolMetrics("worker", "vert.x-internal-blocking", internalBlockingPoolSize) : null;

    closeFuture = new CloseFuture(log);
    maxEventLoopExecTime = maxEventLoopExecuteTime;
    maxEventLoopExecTimeUnit = maxEventLoopExecuteTimeUnit;
    eventLoopThreadFactory = createThreadFactory(threadFactory, checker, useDaemonThread, maxEventLoopExecTime, maxEventLoopExecTimeUnit, "vert.x-eventloop-thread-", false);
    // 创建了用于IO的 EventLoopGroup
    eventLoopGroup = transport.eventLoopGroup(Transport.IO_EVENT_LOOP_GROUP, options.getEventLoopPoolSize(), eventLoopThreadFactory, NETTY_IO_RATIO);
    // 接收者线程池, 大小为1并且ioRatio为100, 表示不会执行任何非IO工作.
    acceptorEventLoopGroup = transport.eventLoopGroup(Transport.ACCEPTOR_EVENT_LOOP_GROUP, 1, acceptorEventLoopThreadFactory, 100);
    // WorkerPool 只是简单的Executor容器, 帮助我们管理 PoolMetrics 和其监控的Pool保持同步关闭
    internalWorkerPool = new WorkerPool(internalWorkerExec, internalBlockingPoolMetrics);
    namedWorkerPools = new HashMap<>();
    workerPool = new WorkerPool(workerExec, workerPoolMetrics);
    defaultWorkerPoolSize = options.getWorkerPoolSize();
    maxWorkerExecTime = maxWorkerExecuteTime;
    maxWorkerExecTimeUnit = maxWorkerExecuteTimeUnit;

    this.checker = checker;
    this.useDaemonThread = useDaemonThread;
    this.executorServiceFactory = executorServiceFactory;
    this.threadFactory = threadFactory;
    this.transport = transport;
    this.sharedData = new SharedDataImpl(this, clusterManager);
    //省略其他代码...
  }

  // 一个简单的包装, 自定义了线程的名称, 指定了线程最大执行时长
  private static ThreadFactory createThreadFactory(VertxThreadFactory threadFactory, BlockedThreadChecker checker, Boolean useDaemonThread, long maxExecuteTime, TimeUnit maxExecuteTimeUnit, String prefix, boolean worker) {
    AtomicInteger threadCount = new AtomicInteger(0);
    return runnable -> {
      VertxThread thread = threadFactory.newVertxThread(runnable, prefix + threadCount.getAndIncrement(), worker, maxExecuteTime, maxExecuteTimeUnit);
      checker.registerThread(thread, thread.info);
      if (useDaemonThread != null && thread.isDaemon() != useDaemonThread) {
        thread.setDaemon(useDaemonThread);
      }
      return thread;
    };
  }

  //ExecutorServiceFactory中的提供的实现. 使用JDK的Eexcutor框架提供的固定线程池.
  ExecutorServiceFactory INSTANCE = (threadFactory, concurrency, maxConcurrency) ->
    Executors.newFixedThreadPool(maxConcurrency, threadFactory);

}
```

## Context
Handler 执行上下文.
通常来说一个 Context 是EventLoop Contenxt, 与一个指定的 EventLoop 线程绑定.
因此该上下文的执行始终发生在完全相同的event loop线程上.
```java
public interface Context {

  // 主要提供的方法，运行一个异步任务.
  void runOnContext(Handler<Void> action);

  // 运行阻塞任务, 与runOnContext不同, 不会运行在event loop线程中.
  <T> void executeBlocking(Handler<Promise<T>> blockingCodeHandler, boolean ordered, Handler<AsyncResult<@Nullable T>> resultHandler);

  <T> Future<@Nullable T> executeBlocking(Handler<Promise<T>> blockingCodeHandler, boolean ordered);

}

public interface ContextInternal extends Context {
  static ContextInternal current() {
    Thread thread = Thread.currentThread();
    if (thread instanceof VertxThread) {
      return ((VertxThread) thread).context();
    } else {
      VertxImpl.ContextDispatch current = VertxImpl.nonVertxContextDispatch.get();
      if (current != null) {
        return current.context;
      }
    }
    return null;
  }

}
```

# NetServer

## TCPServerBase

```java
public abstract class TCPServerBase implements Closeable, MetricsProvider {

  protected final Context creatingContext;
  protected final VertxInteral vertx;
  protected final NetServerOptionso options;

  // Per Server
  protected EventLoop eventLoop;
  private Handler<Channel> worker;
  // Server 监听状态, 已经监听则为true;
  private volatile boolean listening;
  private ContextInternal listenContext;
  private TCPServerBase actualServer;

  // Main
  private SSLHelper sslHelper;
  private ServerChannelLoadBalancer channelBalancer;
  private Future<Channel> bindFuture;
  private Set<TCPServerBase> servers;
  private TCPMetrices<?> metrics;
  private volatile int actualPort;

  public TCPServerBase(VertxInternal vertx, NetServerOptions options) {
    this.vertx = vertx;
    this.options = new NetServerOptions(options);
    this.creatingContext = vertx.getContext();
  }
  
}
```

```java
private synchronized Future<Channel> listen(SocketAddress localAddress, ContextInternal context) {
  // 首先判断是否已经监听过了
  if (listening) {
    throw new IllegalStateException("Listen already called");
  }

  this.listenContext = context;
  this.listening = true;
  // 从context 中获取关联的 eventLoop
  this.eventLoop = context.nettyEventLoop();

  SocketAddress bindAddress;
  Map<ServerID, TCPServerBase> sharedNetServers = vertx.sharedTCPServers((Class<TCPServerBase>) getClass());
  synchronized (sharedNetServers) {
    actualPort = localAddress.port();
    String hostOrPath = localAddress.isInetSocket() ? localAddress.host() : localAddress.path();
    TCPServerBase main;
    boolean shared;
    ServerID id;
    if (actualPort > 0 || localAddress.isDomainSocket()) {
      id = new ServerID(actualPort, hostOrPath);
      main = sharedNetServers.get(id);
      shared = true;
      bindAddress = localAddress;
    } else {
      if (actualPort < 0) {
        id = new ServerID(actualPort, hostOrPath + "/" + -actualPort);
        main = sharedNetServers.get(id);
        shared = true;
        bindAddress = SocketAddress.inetSocketAddress(0, localAddress.host());
      } else {
        id = new ServerID(actualPort, hostOrPath);
        main = null;
        shared = false;
        bindAddress = localAddress;
      }
    }
    PromiseInternal<Channel> promise = listenContext.promise();
    if (main == null) {
      // The first server binds the socket
      actualServer = this;
      bindFuture = promise;
      sslHelper = createSSLHelper();
      worker =  childHandler(listenContext, localAddress, sslHelper);
      servers = new HashSet<>();
      servers.add(this);
      // accpectorEventLoopGroup中只有一个EventLoop. 
      // 所以其实所有的Server Socket IO请求都是由同一个EventLoop处理的.
      // 由所以它的ioRatio 设置为了 100
      channelBalancer = new ServerChannelLoadBalancer(vertx.getAcceptorEventLoopGroup().next());

      // Register the server in the shared server list
      if (shared) {
        sharedNetServers.put(id, this);
      }

      listenContext.addCloseHook(this);

      // Initialize SSL before binding
      sslHelper.init(listenContext).onComplete(ar -> {
        if (ar.succeeded()) {
          // 在SSL初始化完成后
          // Socket bind
          channelBalancer.addWorker(eventLoop, worker);
          // Netty 的部分
          ServerBootstrap bootstrap = new ServerBootstrap();
          // 注意这里的childGroup 参数是 ChannelBalancer 里的 VertxEventLoopGroup
          // 是一个手动添加EventLoop的动态EventLoopGroup.
          // 也就是说首次调用时, 这个 childGroup 中只有一个EventLoop
          // 所有的连接都在这一个EventLoop上处理.
          // 结合ShareServer机制可以用于自定义使用的EventLoop个数.
          bootstrap.group(vertx.getAcceptorEventLoopGroup(), channelBalancer.workers());
          if (sslHelper.isSSL()) {
            bootstrap.childOption(ChannelOption.ALLOCATOR, PartialPooledByteBufAllocator.INSTANCE);
          } else {
            bootstrap.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
          }

          // ChannelBalancer 重点对象
          bootstrap.childHandler(channelBalancer);
          applyConnectionOptions(localAddress.isDomainSocket(), bootstrap);

          // Actual bind
          io.netty.util.concurrent.Future<Channel> bindFuture = AsyncResolveConnectHelper.doBind(vertx, bindAddress, bootstrap);
          bindFuture.addListener((GenericFutureListener<io.netty.util.concurrent.Future<Channel>>) res -> {
            if (res.isSuccess()) {
              Channel ch = res.getNow();
              log.trace("Net server listening on " + hostOrPath + ":" + ch.localAddress());
              if (shared) {
                ch.closeFuture().addListener((ChannelFutureListener) channelFuture -> {
                  synchronized (sharedNetServers) {
                    sharedNetServers.remove(id);
                  }
                });
              }
              // Update port to actual port when it is not a domain socket as wildcard port 0 might have been used
              if (bindAddress.isInetSocket()) {
                actualPort = ((InetSocketAddress)ch.localAddress()).getPort();
              }
              metrics = createMetrics(localAddress);
              promise.complete(ch);
            } else {
              promise.fail(res.cause());
            }
          });
        } else {
          promise.fail(ar.cause());
        }
      });

      bindFuture.onFailure(err -> {
        if (shared) {
          synchronized (sharedNetServers) {
            sharedNetServers.remove(id);
          }
        }
        listening = false;
      });

      return bindFuture;
    } else {
      // Server already exists with that host/port - we will use that
      actualServer = main;
      metrics = main.metrics;
      sslHelper = main.sslHelper;
      worker =  childHandler(listenContext, localAddress, sslHelper);
      actualServer.servers.add(this);
      actualServer.channelBalancer.addWorker(eventLoop, worker);
      listenContext.addCloseHook(this);
      main.bindFuture.onComplete(promise);
      return promise.future();
    }
  }
}

```


```java
class ServerChannelLoadBalancer extends ChannelInitializer<Channel> {

  private final VertxEventLoopGroup workers;
  private final ChannelGroup channelGroup;
  // 
  private final ConcurrentMap<EventLoop, WorkerList> workerMap = new ConcurrentHashMap<>();

  // We maintain a separate hasHandlers variable so we can implement hasHandlers() efficiently
  // As it is called for every HTTP message received
  private volatile boolean hasHandlers;

  // 唯一的调用是 TCPserverBase.java 141 channelBalancer = new ServerChannelLoadBalancer(vertx.getAcceptorEventLoopGroup().next());
  // 所以这里其实就是vertx中的唯一用来处理ServerSocket accept事件的EventLoop(Acceptor).
  ServerChannelLoadBalancer(EventExecutor executor) {
    // 这里 Vertx 自己实现了一个简单的 EventLoopGroup逻辑
    // 可以添加的 EventLoop 的 EventLoopGroup.
    // 简单的说就是用了一个 List, 保持真正的EventLoop, 并且提供 add 和 remove 方法
    this.workers = new VertxEventLoopGroup();
    // 用了 Netty ChannelGroup. 每个子线程添加到其中. 
    // 然后在server stop时可以统一close
    this.channelGroup = new DefaultChannelGroup(executor);
  }

  @Override
  protected void initChannel(Channel ch) {
    // 通过 channel 绑定的EventLoop 获取对应的处理逻辑.
    // 这里的 worker 是ServerBootstrap的childGroup.
    Handler<Channel> handler = chooseInitializer(ch.eventLoop());
    if (handler == null) {
      ch.close();
    } else {
      channelGroup.add(ch);
      handler.handle(ch);
    }
  }

  private Handler<Channel> chooseInitializer(EventLoop worker) {
    WorkerList handlers = workerMap.get(worker);
    return handlers == null ? null : handlers.chooseHandler();
  }

  /**
  * EventLoop Channel的处理线程
  * handler 是处理Channel的逻辑
  */
  public synchronized void addWorker(EventLoop eventLoop, Handler<Channel> handler) {
    // 1. 先把 eventLoop 添加到 eventLoopGroup中
    workers.addWorker(eventLoop);
    WorkerList handlers = new WorkerList();
    WorkerList prev = workerMap.putIfAbsent(eventLoop, handlers);
    if (prev != null) {
      handlers = prev;
    }
    handlers.addWorker(handler);
    hasHandlers = true;
  }

  public synchronized boolean removeWorker(EventLoop worker, Handler<Channel> handler) {
    WorkerList handlers = workerMap.get(worker);
    if (handlers == null || !handlers.removeWorker(handler)) {
      return false;
    }
    if (handlers.isEmpty()) {
      workerMap.remove(worker);
    }
    if (workerMap.isEmpty()) {
      hasHandlers = false;
    }
    //Available workers does it's own reference counting -since workers can be shared across different Handlers
    workers.removeWorker(worker);
    return true;
  }

}
```
