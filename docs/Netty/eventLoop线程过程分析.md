# eventLoop线程过程分析

![](/images/Netty/Netty启动流程.png)

Netty启动器在调用bind()方法之后，会调用AbstractBootstrap的initAndRegister() 方法完成Channel的创建、初始化与注册。
```
// AbstractBootstrap
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        // 创建Channel
        channel = channelFactory.newChannel(); 
        // 初始化Channel
        init(channel); 
    } catch (Throwable t) {
        // 省略异常处理
        ······
    }
    // 注册Channel
    ChannelFuture regFuture = config().group().register(channel); 
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    } 
    return regFuture;
}
```
## 创建Channel
从上述代码可以看出，通过channelFactory.newChannel()实现Channel的创建，而channelFactory是在用户代码调用b.channel(NioServerSocketChannel.class)时创建的，工厂内部会保存传入的class对象，然后通过反射的方式创建对应的Channel：
```
// channelFactory其实就是ReflectiveChannelFactory的实例
public ReflectiveChannelFactory(Class<? extends T> clazz) {
    ObjectUtil.checkNotNull(clazz, "clazz");
    try {
        this.constructor = clazz.getConstructor();
    } catch (NoSuchMethodException e) {
        // 抛出异常
    }
}

@Override
public T newChannel() {
    try {
        return constructor.newInstance();
    } catch (Throwable t) {
        // 抛出异常
    }
}
```
Netty NIO服务端使用NioServerSocketChannel作为服务端Channel类型，那么接下来就要进入NioServerSocketChannel的构造方法。首先是创建jdk底层的ServerSocketChannel：
```
// NioServerSocketChannel
public NioServerSocketChannel() {
    // 创建socket
    this(newSocket(DEFAULT_SELECTOR_PROVIDER)); 
}

private static ServerSocketChannel newSocket(SelectorProvider provider){   
    // 创建jdk底层的ServerSocketChannel
    return provider.openServerSocketChannel(); 
}
```
在原生的jdk nio编程中会用ServerSocketChannel.open()的方式获取服务端的 ServerSocketChannel，实际上是调用了SelectorProvider.provider().openServerSocketChannel()，那么在Netty中也一样。SelectorProvider是Java提供的NIO的抽象类，会根据操作系统类型和版本确定具体的实现类：如果Linux内核版本>=2.6则具体的实现类为EPollSelectorProvider，否则为默认的 PollSelectorProvider。

创建完jdk底层的Channel后，NioServerSocketChannel会调用父类的构造方法创建三个重要的成员变量id、unsafe和pipeline：
```
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    // 调用父类构造方法创建id、unsafe和pipeline
    super(parent);
    // 保存jdk底层的ServerSocketChannel
    this.ch = ch;
    // 设置感兴趣的事件类型
    // 若是server端，设置的是SelectionKey.OP_ACCEPT
    this.readInterestOp = readInterestOp;
    try {
        // 设置为非阻塞
        ch.configureBlocking(false);
    } catch (IOException e) {
       // 异常处理，省略
       ····· 
    }
}
```
调用父类构造方法创建id、unsafe和pipeline
```
// AbstractChannel
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}

protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);
    // 初始化双向链表
    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```
从父类的名字AbstractChannel可以看出，这不仅是服务端Channel的父类，也是所有Netty Channel的父类，因此在所有Netty Channel中都有这三个成员变量。其中id用于标识唯一 Channel；unsafe用于处理底层读写操作；pipeline用于处理数据流入流出时的逻辑链。当然此时的pipeline还是空的，内部只有头尾两个节点。
## 初始化Channel
从上述代码可以看出，通过init(channel)实现Channel的初始化
```
@Override
void init(Channel channel) {
    // 设置JDK底层socket的参数
    setChannelOptions(channel, newOptionsArray(), logger);
    // 在服务端Channel中保存用户传入的自定义属性
    setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));
    ChannelPipeline p = channel.pipeline();
    
    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    }
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);
    // 向服务端Channel的pipeline添加handler
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```
## 注册Selector
初始化服务端Channel之后，就要把Channel注册到Selector上了。从initAndRegister() 方法代码可以看出，通过config().group().register(channel)实现注册Selector。其中的group()方法返回的是bossGroup，是调用b.group(bossGroup, workerGroup)保存起来的。注册的过程主要做了以下几件事：

    （1）绑定EventLoop
    （2）调用JDK底层注册
    （3）触发Pipeline上面的注册事件
```  
// MultithreadEventLoopGroup    
public ChannelFuture register(Channel channel) {
    // 选择一个eventLoop
    return next().register(channel); 
}
```
从上述代码可以看出，会从bossGroup中选择一个eventLoop，然后调用其register(channel)方法进行注册，而eventLoop的register(channel)方法最终会调用AbstractChannel#AbstractUnsafe.register方法进行注册。下面分析的NioEventLoop启动其实就是注册Selector的过程。
  
### eventLoop线程已启动
AbstractChannel#AbstractUnsafe.register方法中可以看出，当eventLoop线程已启动时，调用register0(promise)完成注册Selector。
```
private void register0(ChannelPromise promise) {
    try {
        //...
        boolean firstRegistration = neverRegistered;
        // 调用JDK底层的register()
        doRegister(); 
        neverRegistered = false;
        registered = true;
        // 触发handlerAdded事件
        pipeline.invokeHandlerAddedIfNeeded(); 

        safeSetSuccess(promise);
        // 触发channelRegistered事件
        pipeline.fireChannelRegistered();

        // 当前状态为活跃则触发ChannelActive事件（注册时不活跃，绑定端口后活跃）
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        //...
    }
}
```
register0内部会先调用JDK底层的register()进行注册，然后在pipeline上触发一系列的事件。先看JDK底层的注册逻辑：
```
// AbstractNioChannel
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            // 调用JDK底层的register()注册
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this); 
            return;
        } catch (CancelledKeyException e) {
            //...
        }
    }
}
```
在每个NioEventLoop对象上，都独有一个Selector对象，这是在创建NioEventLoop对象时就初始化好的。然后通过JDK底层的register()注册Java原生NIO的Channel对象到Selector对象上。
JDK底层的register()的第三个参数是attachment，传进去的this是Netty自己的Channel。这步操作非常巧妙。之前可能会疑惑，Selector每次返回的是JDK底层的Channel，那么Netty是怎么知道它对应哪个Netty Channel的呢？这里找到了答案：Netty通过把自己的 Channel作为attachment绑定在JDK底层的Channel上，在每次返回的时候带出来。
## NioEventLoop未启动
**问题：**

    （1）EventLoop实际上是对应于一个线程，那么这个EventLoop是何时启动的？
    （2）EventLoop与Channel的关联？并且是何时关联的？
    
当调用AbstractChannel#AbstractUnsafe.register方法后, 就完成了Channel和 EventLoop的关联，即将Channel注册到eventLoop上。注册之后，在register方法中会执行eventLoop的execute()方法，在该方法中会完成EventLoop的启动。

```
// AbstractChannel#AbstractUnsafe.register方法
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    ObjectUtil.checkNotNull(eventLoop, "eventLoop");
    // 是否已经注册
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure( new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }
    // Channel与eventLoop关联
    AbstractChannel.this.eventLoop = eventLoop;
    // 同一个channel的注册、读、写等都在eventLoop完成,避免多线程的锁竞争
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            // 若eventLoop线程没有启动，启动该线程
            // 当前线程非eventLoop时，会把注册任务提交给eventLoop的任务队列
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            // 省略
            ...
        }
    }
}
```
上面调用了eventLoop.execute()方法，eventLoop是一个NioEventLoop的实例，而 NioEventLoop没有实现execute()方法, 因此调用的是SingleThreadEventExecutor.execute()方法

```
public void execute(Runnable task) {
    ObjectUtil.checkNotNull(task, "task");
    execute(task, !(task instanceof LazyRunnable) && wakesUpForTask(task));
}

 /**
   * 判断Thread.currentThread()当前线程是不是与NioEventLoop绑定的本地线程;
   * 如果Thread.currentThread()== this.thread, 那么只用将execute()方法中的task添加到任务队列中就好;
   * 如果Thread.currentThread()== this.thread 返回false, 那就先调用startThread()方法启动本地线程,然后再将task添加到任务队列中.
   */
private void execute(Runnable task, boolean immediate) {
    // 是否在EventLoop线程中
    boolean inEventLoop = inEventLoop();
    // 将任务task添加到SingleThreadEventExecutor的taskQueue
    addTask(task);
    // 当前不在EventLoop的线程
    if (!inEventLoop) {
        // 启动EventLoop独占的线程
        startThread();
        // 若已经关闭（SingleThreadEventExecutor的状态为关闭状态），移除任务，并进行拒绝
        if (isShutdown()) {
            boolean reject = false;
            try {
                if (removeTask(task)) {
                    reject = true;
                }
            } catch (UnsupportedOperationException e) {
                  
            }
            if (reject) {
                reject();
            }
        }
    }
    // 唤醒线程
    if (!addTaskWakesUp && immediate) {
        wakeup(inEventLoop);
    }
}
```
继续调用SingleThreadEventExecutor.startThread()方法启动线程
```
private void startThread() {
    if (state == ST_NOT_STARTED) {
        //设置thread 的状态 this.state是 ST_STARTED 已启动
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            // 是否启动标识
            boolean success = false;
            try {
                //真正启动线程的函数, 其作用类似于 thread.start()
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
```
```
private void doStartThread() {
    //启动线程之前，必须保证thread 是null，其实也就是thread还没有启动
    assert thread == null;
    executor.execute(new Runnable() {
        @Override
        public void run() {
            // 记录当前线程，即将当前线程绑定到EventLoop中
            thread = Thread.currentThread();
            // 如果当前线程已经被标记打断，则进行打断操作
            if (interrupted) {
                thread.interrupt();
            }
            // 是否执行成功
            boolean success = false;
            // 更新最后执行时间
            updateLastExecutionTime();
            try {
                // 执行任务，实际上是调用NioEventLoop.run()方法实现
                SingleThreadEventExecutor.this.run();
                // 标记执行成功
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                // 省略，确保事件循环结束之后，关闭线程、清理资源
                ...
            }
        }
    });
}
```
## NioEventLoop运行
由上面分析可知，NioEventLoop运行的核心就是SingleThreadEventExecutor.this.run()方法，即SingleThreadEventExecutor类的run()方法，这是一个抽象方法。在NioEventLoop实现了这个方法。NioEventLoop.run()源码如下：
```
protected void run() {
    int selectCnt = 0;
    // 死循环，NioEventLoop事件循环的核心在此
    for (;;) {
        try {
            int strategy;
            try {
                // 通过select/selectNow调用查询当前是否有就绪的IO事件
                // 当selectStrategy.calculateStrategy()返回的是CONTINUE(默认情况下，不存在这种情况)，就结束此轮循环，进入下一轮循环;
                // 当返回的是SELECT，就表示任务队列为空，就调用select(Boolean)，进行Selector阻塞select。
                // 若任务队列不为空，则会通过selectNow进行Selector非阻塞查询，若查询结果>=0，则表示有就绪的IO任务需要处理。因为在任务队列不为空的情况下，不能进行Selector阻塞select，因为还需要去执行任务。
                strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                switch (strategy) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.BUSY_WAIT:
                case SelectStrategy.SELECT:
                    long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                    if (curDeadlineNanos == -1L) {
                        curDeadlineNanos = NONE; 
                    }
                    nextWakeupNanos.set(curDeadlineNanos);
                    try {
                        if (!hasTasks()) {
                            strategy = select(curDeadlineNanos);
                        }
                    } finally {
                        nextWakeupNanos.lazySet(AWAKE);
                    }
                default:
                }
            } catch (IOException e) {
                rebuildSelector0();
                selectCnt = 0;
                handleLoopException(e);
                continue;
            }

            // 当select的结果>=0时，会走到以下逻辑。即有IO事件就绪时，处理这些IO事件
            //上面的try代码块中strategy的结果就是执行一次select操作的结果，也就是事件的数量
            //select的次数加一（不管hasTasks是否为true都会执行一次select操作，如果是CONTINUE的话，则不会走到这里而是直接下一次循环，并且CONTINUE这个变量除了这个方法的case中也没有其他地方用到过），重置cancelledKeys和needsToSelectAgain
            // selectCnt加一 如果连续512次加一之后没有被重置为0 ，那么netty会认为进入空转了
            selectCnt++;
            cancelledKeys = 0;
            needsToSelectAgain = false;
            //ioRatio表示此线程分配给IO操作所占的时间比(即运行processSelectedKeys耗时在整个循环中所占用的时间)，ioRatio的配置不同，分成略有差异的2种
            final int ioRatio = this.ioRatio;
            boolean ranTasks;
            // ioRatio为100，则不考虑时间占比的分配
            if (ioRatio == 100) {
                try {
                    // strategy>0表示有IO就绪事件，则直接处理
                    if (strategy > 0) {
                        processSelectedKeys();
                    }
                } finally {
                    // 运行所有普通任务和定时任务，不限制时间
                    ranTasks = runAllTasks();
                }
            } else if (strategy > 0) {
                // 这是ioRatio<100，并且存在IO就绪事件的情况，则考虑时间占比的分配
                // 记录当前时间
                final long ioStartTime = System.nanoTime();
                try {
                    // 处理IO就绪事件
                    processSelectedKeys();
                } finally {
                    // 通过当前时间，计算出前面IO事件处理使用的时间
                    final long ioTime = System.nanoTime() - ioStartTime;
                    // 运行所有普通任务和定时任务，限制时间。但是注意，它是以 #processSelectedKeys()方法执行时花费的时间作为基准，来计算runAllTasks(long timeoutNanos)方法可执行的时间。
                    ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            } else {
                // 这是没有IO就绪事件的情况，则直接执行普通任务与定时任务
                ranTasks = runAllTasks(0); 
            }

            if (ranTasks || strategy > 0) {
                if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                    // 省略，异常日志
                }
                selectCnt = 0;
            } else if (unexpectedSelectorWakeup(selectCnt)) {
                selectCnt = 0;
            }
        } catch (CancelledKeyException e) {
            // 省略，异常日志
                    ...
        } catch (Throwable t) {
            handleLoopException(t);
        }
        
        // 优雅关闭
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```    
### SelectStrategy
SelectStrategy选择(select)策略接口，源码如下：
```
public interface SelectStrategy {
    // 表示使用阻塞 select 的策略。
    int SELECT = -1;
    
    // 表示需要进行重试的策略。
    int CONTINUE = -2;

    // 该方法有3种返回情况；(1)SELECT：-1 ，表示使用阻塞select的策略
    // (2)CONTINUE：-2，表示需要进行重试的策略。实际上默认情况下，不会返回 CONTINUE的策略
    // (3)>= 0,表示不需要select,目前已经有可以执行的任务了
    int calculateStrategy(IntSupplier selectSupplier, boolean hasTasks) throws Exception;  
}
```
### DefaultSelectStrategy
DefaultSelectStrategy实现SelectStrategy接口，默认选择策略实现类，源码如下：
```
final class DefaultSelectStrategy implements SelectStrategy {
    // 单例
    static final SelectStrategy INSTANCE = new DefaultSelectStrategy();

    private DefaultSelectStrategy() { }

    @Override
    public int calculateStrategy(IntSupplier selectSupplier, boolean hasTasks) throws Exception {
        return hasTasks ? selectSupplier.get() : SelectStrategy.SELECT;
    }
}
```
当hasTasks为true时，表示当前已经有任务，所以调用IntSupplier#get()方法，返回当前 Channel新增的IO就绪事件的数量。当hasTasks为false时，直接返回SelectStrategy.SELECT，进行阻塞select Channel感兴趣的就绪IO事件。NioEventLoop中IntSupplier的实现代码如下：
```
// IntSupplier接口
public interface IntSupplier {
    int get() throws Exception;
}

// selectNowSupplier属性是IntSupplier在NioEventLoop中的实现，在它的内部会调用 selectNow()方法
private final IntSupplier selectNowSupplier = new IntSupplier() {
    @Override
    public int get() throws Exception {
        return selectNow();
    }
};
```
### selectNow
```
int selectNow() throws IOException {
    try {
        // 立即(无阻塞)返回Channel新增的感兴趣的就绪IO事件数量
        return selector.selectNow(); 
    } finally {
        // 若唤醒标识wakeup为true时，调用Selector#wakeup()方法，唤醒 Selector??
        if (wakenUp.get()) {
            selector.wakeup();
        }
    }
}
```
### select()方法
NioEventLoop在运行过程中，若任务队列为空，就会调用select()方法对Selector进行阻塞轮询。下面从SingleThreadEventExecutor.this.run()方法中截取该部分相关的代码如下：
```
case SelectStrategy.SELECT:
    // 获取延时队列中即将执行的任务，若没有，则返回-1
    // 若有即将执行的延时任务，则返回该延时任务的延时时间
    long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
    if (curDeadlineNanos == -1L) {
        curDeadlineNanos = NONE; 
    }
    nextWakeupNanos.set(curDeadlineNanos);
    try {
        // 再次判断任务队列中是否有任务
        if (!hasTasks()) {
            // 进行IO的select操作
            strategy = select(curDeadlineNanos);
        }
    } finally {
        nextWakeupNanos.lazySet(AWAKE);
    }
default:
```
```
private int select(long deadlineNanos) throws IOException {
    // deadlineNanos为NONE，说明没有延时任务，并且普通任务队列也为空。即当前没有其他任务需要执行，则直接进行阻塞select，查询Channel是否有就绪的IO事件
    if (deadlineNanos == NONE) {
        return selector.select();
    }
    // 若deadlineNanos不为NONE，则表明还有定时任务需要执行，则计算本次select的超时时长。即select超时之后，就去执行定时任务
    // / 1000000L 是为了纳秒转为毫秒
    long timeoutMillis = deadlineToDelayNanos(deadlineNanos + 995000L) / 1000000L;
    // 若超时时间<=0，则直接进行非阻塞selectNow，否则进行阻塞超时select
    return timeoutMillis <= 0 ? selector.selectNow() : selector.select(timeoutMillis);
}
```

## 问题
### 什么是Jdk的Selector空轮询

![](/images/Netty/JDK-Epoll-Bug.png)

正常情况下，selector.select()操作是阻塞的，只有被监听的fd有读写操作时，才被唤醒
但是，在这个bug中，没有任何fd有读写请求，但是select()操作依旧被唤醒。很显然，这种情况下，selectedKeys()返回的是个空数组。然后按照逻辑执行到while(true)处，循环执行，导致死循环。最终导致CPU使用率100%。

### Netty如何解决了Jdk的Selector空轮询bug
Netty对Selector的select操作周期进行统计，每完成一次空的select操作进行一次计数，若在某个周期内连续发生N次(默认512次)空轮询，则触发了epoll死循环bug。Netty解决空轮询Bug是调用rebuildSelector()方法，重建Selector对象
```
public void rebuildSelector() {
        if (!inEventLoop()) {
            execute(new Runnable() {
                @Override
                public void run() {
                    rebuildSelector0();
                }
            });
            return;
        }
        rebuildSelector0();
}
```

## 总结
### Channel创建、初始化、注册

![](/images/Netty/Channel创建初始化注册.png)

### NioEventLoop运行

![](/images/Netty/NioEventLoop运行.png)

