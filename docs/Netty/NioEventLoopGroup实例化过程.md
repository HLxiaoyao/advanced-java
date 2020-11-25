# NioEventLoopGroup实例化过程
## NioEventLoopGroup类层次结构

![](/images/Netty/NioEventLoopGroup类层次结构.jpeg)
```
public NioEventLoopGroup(int nThreads) {
    this(nThreads, (Executor) null);
}
```
这里设置了NioEventLoopGroup线程池中每个线程执行器默认是null(这里设置为null，在后面的构造器中会判断，如果为null就实例化一个线程执行器)。
```
public NioEventLoopGroup(int nThreads, Executor executor) {
    this(nThreads, executor, SelectorProvider.provider());
}
```
这里就存在于JDK的NIO的交互了，这里设置了线程池的SelectorProvider，通过SelectorProvider.provider()返回。
```
public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider) {
    this(nThreads, executor, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
}
```
在这个重载的构造器中又传入了默认的选择策略工厂DefaultSelectStrategyFactory
```
public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider, final SelectStrategyFactory selectStrategyFactory) {
    super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
}
```
这里就是调用父类MultithreadEventLoopGroup的构造器了，这里还添加了线程的拒绝执行策略。
```
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
```
构造器被定义成protect，表示只能在NioEventLoopGroup中被调用，一定层度上的保护作用。这里就是对线程数进行了判断，当nThreads为0 的时候就设置成DEFAULT_EVENT_LOOP_THREADS 这个常量。这个常量的定义如下：其实就是在MultithreadEventLoopGroup的静态代码段，其实就是将DEFAULT_EVENT_LOOP_THREADS赋值为CPU核心数*2；接下来就是调用的基类MultithreadEventExecutorGroup的构造器：
```
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
    this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
}
```
这个构造器里面多传入了一个参数 DefaultEventExecutorChooserFactory.INSTANCE，通过这个EventLoop选择器工厂可以实例化GenericEventExecutorChooser这个类，这个类是EventLoopGroup线程池里面的EventLoop的选择器，调用GenericEventExecutorChooser.next() 方法可以从线程池中选择出一个合适的EventLoop线程。然后就是重载调用MultithreadEventExecutorGroup类的构造器：
```
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory, Object... args) {
    // 参数校验nThread合法性
    if (nThreads <= 0) {
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
    }
    // executor校验非空，如果为空就创建ThreadPerTaskExecutor，该类实现了 Executor接口
    // 这个executor是用来执行线程池中的所有的线程，也就是所有的NioEventLoop，其实从NioEventLoop构造器中也可以知道，NioEventLoop构造器中都传入了executor这个参数。
    if (executor == null) {
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }
    // 数组中每个元素其实就是一个EventLoop，EventLoop是EventExecutor的子接口。
    children = new EventExecutor[nThreads];
    // for循环实例化children数组，NioEventLoop对象
    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            // newChild(executor, args) 函数在NioEventLoopGroup类中实现了, 
            // 实质就是就是存入了一个 NioEventLoop类实例
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
            // 抛出异常
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
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }
    }
    // 实例化线程工厂执行器选择器: 根据children获取选择器
    chooser = chooserFactory.newChooser(children);
    // 为每个EventLoop线程添加线程终止监听器
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
    // 将children添加到对应的set集合中去重，表示只可读
    Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
    Collections.addAll(childrenSet, children);
    readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```
## NioEventLoop实例化过程
在上述分析中看出，在实例化NioEventLoopGroup的过程中会调用NioEventLoopGroup的newChild()方法实例化NioEventLoop，进行EventExecutor[] children数组填充。
### NioEventLoop类层次结构

![](/images/Netty/NioEventLoop类层次结构.jpeg)

NioEventLoop继承于SingleThreadEventLoop；SingleThreadEventLoop又继承于 SingleThreadEventExecutor；SingleThreadEventExecutor是netty中对本地线程的抽象，它内部有一个Thread thread属性，存储了一个本地Java线程。因此可以认为一个NioEventLoop其实和一个特定的线程绑定，并且在其生命周期内，绑定的线程都不会再改变。

### NioEventLoop实例化
```
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
} 
```
最先调用NioEventLoop 里面的构造函数：
```
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider, SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler) {

    //调用父类构造器
    super(parent, executor, false, DEFAULT_MAX_PENDING_TASKS, rejectedExecutionHandler);

    if (selectorProvider == null) {
        throw new NullPointerException("selectorProvider");
    }
    if (strategy == null) {
        throw new NullPointerException("selectStrategy");
    }
    provider = selectorProvider;
    //new 一个selector实例，具体的类与平台和底层有关
    selector = openSelector();
    selectStrategy = strategy;
}
```
需要注意的是：构造器里面传入了 NioEventLoopGroup、Executor、SelectorProvider、SelectStrategyFactory、RejectedExecutionHandler。从这里可以看出，一个NioEventLoop属于某一个NioEventLoopGroup， 且处于同一个NioEventLoopGroup下的所有NioEventLoop公用Executor、SelectorProvider、SelectStrategyFactory和RejectedExecutionHandler。

需要注意的是，这里的SelectorProvider构造参数传入的是通过在NioEventLoopGroup里面的构造器里面的SelectorProvider.provider()方式获取的，而这个方法返回的是一个单例的SelectorProvider，所以所有的NioEventLoop公用同一个单例SelectorProvider。接下来调用父类SingleThreadEventLoop的构造器：
```
protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor, boolean addTaskWakesUp, int maxPendingTasks, RejectedExecutionHandler rejectedExecutionHandler) {
    //父类构造器
    super(parent, executor, addTaskWakesUp, maxPendingTasks, rejectedExecutionHandler);
    // 尾部任务队列
    tailTasks = newTaskQueue(maxPendingTasks);
}
```
队列的数量maxPendingTasks参数默认是SingleThreadEventLoop.DEFAULT_MAX_PENDING_TASK，其实就是Integer.MAX_VALUE; 对于new的这个队列，其实就是一个LinkedBlockingQueue无界队列。接着调用父类SingleThreadEventExecutor的构造器
```
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor, boolean addTaskWakesUp, int maxPendingTasks, RejectedExecutionHandler rejectedHandler) {
    // 设置EventLoop所属于的EventLoopGroup
    super(parent);
    this.addTaskWakesUp = addTaskWakesUp;
    // 默认是Integer.MAX_VALUE
    this.maxPendingTasks = Math.max(16, maxPendingTasks);
    this.executor = ObjectUtil.checkNotNull(executor, "executor");
    // 创建EventLoop的任务队列，默认是LinkedBlockingQueue
    taskQueue = newTaskQueue(this.maxPendingTasks);
    rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
}
```
## 总结
### NioEventLoopGroup实例化时序图

![](/images/Netty/NioEventLoopGroup实例化.png)

### MultithreadEventExecutorGroup实例化
以下是MultithreadEventExecutorGroup构造方法主要逻辑：

![](/images/Netty/MultithreadEventExecutorGroup实例化.png)

### NioEventLoop实例化时序图

![](/images/Netty/NioEventLoop实例化.png)



