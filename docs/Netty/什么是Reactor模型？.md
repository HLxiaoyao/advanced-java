# 什么是Reactor模型？
IO多路复用 + 线程池 = Reactor模型，根据Reactor的数量和处理线程的数量，Reactor模型分为三类：

    （1）单Reactor单线程
    （2）单Reactor多线程
    （3）主从Reactor多线程
    
![](/images/Netty/Reactor模型.png)

## 单Reactor单线程

![](/images/Netty/单线程单Reactor模型.png)

上图中，Reactor中的select模块就是IO多路复用模型中的选择器，可以通过一个阻塞对象监听多路连接请求。

    （1）Reactor对象通过Select监控客户端请求事件，收到事件后，通过Dispatch进行分发。
    （2）如果是建立连接事件，则用Acceptor通过Accept处理连接请求，然后创建一个Handler对象，处理连接完成后的业务处理。
    （3）如果不是建立连接事件，则Reactor会分发调用连接对应的Handler处理。
    （4）Handler会处理Read-业务-Send流程。

这种模型，在客户端数量过多时，会无法支撑。因为只有一个线程，无法发挥多核CPU性能，且Handler处理某个连接的业务时，服务端无法处理其他连接事件。Redis内部就是应用这种模型。代码如下：

```
/**
 * 等待事件到来，分发事件处理
 */
class Reactor implements Runnable {
    private  Selector selector;
    private ServerSocketChannel serverSocket;
​   
    // 启动阶段
    public Reactor(int port) throws Exception {
        // Reactor初始化
        selector = Selector.open();
        serverSocket = ServerSocketChannel.open();
        serverSocket.socket().bind(new InetSocketAddress(port));
        //非阻塞
        serverSocket.configureBlocking(false);
​        // 接收accept事件
        SelectionKey sk = serverSocket.register(selector, SelectionKey.OP_ACCEPT);
        // attach Acceptor 处理新连接
        sk.attach(new Acceptor());
    }
​
    // 轮询阶段
    @Override
    public void run() {
        try {
            while (!Thread.interrupted()) {
                selector.select();
                Set selected = selector.selectedKeys();
                Iterator it = selected.iterator();
                while (it.hasNext()) {
                    SelectionKey key = (SelectionKey) it.next();
                    //分发事件处理
                    dispatch(key);
                    it.remove();
                }
            }
        } catch (IOException ex) {
            //do something
        }
    }
​
    void dispatch(SelectionKey k) {
        // 若是连接事件获取是acceptor
        // 若是IO读写事件获取是handler
        Runnable runnable = (Runnable) (k.attachment());
        if (runnable != null) {
            // 这里并没有开启新的线程，还是在Reactor轮询的线程里面
            runnable.run();
        }
    }
​
    /**
     * 连接事件就绪,处理连接事件
     */
    class Acceptor implements Runnable {
        @Override
        public void run() {
            try {
                SocketChannel c = serverSocket.accept();
                if (c != null) {// 注册读写
                    new Handler(c, selector);
                }
            } catch (Exception e) {
​
            }
        }
    }
}

/**
 * 处理读写业务逻辑
 */
class Handler implements Runnable {
    public static final int READING = 0, WRITING = 1;
    int state;
    final SocketChannel socket;
    final SelectionKey sk;
​
    public Handler(SocketChannel socket, Selector selector) throws Exception {
        this.state = READING;
        this.socket = socket;
        sk = socket.register(selector, SelectionKey.OP_READ);
        sk.attach(this);
        socket.configureBlocking(false);
    }
​
    @Override
    public void run() {
        if (state == READING) {
            read();
        } else if (state == WRITING) {
            write();
        }
    }
​
    private void read() {
        process();
        //下一步处理写事件
        sk.interestOps(SelectionKey.OP_WRITE);
        this.state = WRITING;
    }
​
    private void write() {
        process();
        //下一步处理读事件
        sk.interestOps(SelectionKey.OP_READ);
        this.state = READING;
    }
​
    /**
     * task 业务处理
     */
    public void process() {
        //do something
    }
}

// 测试代码
public class ReactorTest {
    public static void main(String[] args) throws Exception {
        Thread t = new Thread(new Reactor(9999));
        t.start();
        System.out.println("server start");
        t.join();
    }
}
```

## 单Reactor多线程

![](/images/Netty/单Reactor多线程.png)

Reactor单线程模型中I/O读写和数据处理属于同步操作，在同一个线程中处理，无法处理大量的连接，为了不影响Acceptor线程处理客户端连接，可以将Acceptor线程和I/O处理线程分离开来。图中多线程体现在两个部分：

    （1）Reactor主线程
        Reactor通过select监听客户请求，如果是连接请求事件，则由Acceptor处理连接，如果是其他请求，则由dispatch找到对应的Handler，这里的Handler只负责响应事件，读取和响应，会将具体的业务处理交由Worker线程池处理。

    （2）Worker线程池
        Worker线程池会分配独立线程完成真正的业务，并将结果返回给Handler，Handler收到响应后，通过send将结果返回给客户端。

这里Reactor处理所有的事件监听和响应，高并发情景下容易出现性能瓶颈。代码如下：

```
/**
 * 多线程处理读写业务逻辑
 */
class MultiThreadHandler implements Runnable {
    public static final int READING = 0, WRITING = 1;
    int state;
    final SocketChannel socket;
    final SelectionKey sk;
    //多线程处理业务逻辑
    ExecutorService executorService = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    public MultiThreadHandler(SocketChannel socket, Selector selector) throws Exception {
        this.state = READING;
        this.socket = socket;
        sk = socket.register(selector, SelectionKey.OP_READ);
        sk.attach(this);
        socket.configureBlocking(false);
    }
​
    @Override
    public void run() {
        if (state == READING) {
            read();
        } else if (state == WRITING) {
            write();
        }
    }
​
    private void read() {
        //任务异步处理
        executorService.submit(() -> process());
​
        //下一步处理写事件
        sk.interestOps(SelectionKey.OP_WRITE);
        this.state = WRITING;
    }

    private void write() {
        //任务异步处理
        executorService.submit(() -> process());
​
        //下一步处理读事件
        sk.interestOps(SelectionKey.OP_READ);
        this.state = READING;
    }

    /**
     * task 业务处理
     */
    public void process() {
        //do IO ,task,queue something
    }
}
```

## 主从Reactor多线程

![](/images/Netty/主从Reactor多线程.png)

这种模式是对单Reactor的改进，由原来单Reactor改成了Reactor主线程与Reactor子线程。Reactor主线程的MainReactor对象通过select监听连接事件，收到事件后通过Acceptor处理连接事件。

    （1）当Acceptor处理完连接事件之后，MainReactor将连接分配给SubReactor。MainReactor可以对应多个SubReactor。
    （2）SubReactor将连接加入到连接队列进行监听，并创建handler进行事件处理。
    （3）当有新的事件发生时，SubReactor就会调用对应的handler处理。
    （4）handler通过read读取数据，交由Worker线程池处理业务。
    （5）Worker线程池分配线程处理完数据后，将结果返回给handler。
    （6）handler收到返回的数据后，通过send将结果返回给客户端。

这种优点多多，各个模块各司其职，缺点就是实现复杂。主从Reactor多线程模型分为Main Reactor线程池、Sub Reactor线程池和工作线程池（不包含I/O读写）。

    （1）Main Reactor线程池：线程池中包含多个Acceptor线程，完成客户端的连接请求
    （2）Sub Reactor线程池：线程池中的线程主要负责处理I/O事件，完成I/O读写操作和数据的编码解码

下面看下这种线程模型解决了什么问题：

    （1）Main Reacto线程（Acceptor线程）池化，为了解决单Acceptor线程处理大量连接的效率问题
    （2）Sub Reactor线程池完成I/O读写和编码解码，将这部分任务从工作线程handler中剥离出来
    （3）工作线程池不负责I/O相关，只处理目标数据类型的数据，处理请求是无需关心解码，发送响应时无需关心编码

```
public class MthreadReactor {

    // subReactors集合, 一个selector代表一个subReactor
    Selector[] selectors=new Selector[2];
    int next = 0;
    final ServerSocketChannel serverSocket;

    MthreadReactor(int port) throws IOException {
        // Reactor初始化
        selectors[0]=Selector.open();
        selectors[1]= Selector.open();
        serverSocket = ServerSocketChannel.open();
        serverSocket.socket().bind(new InetSocketAddress(port));
        //非阻塞
        serverSocket.configureBlocking(false);
        
        //分步处理,第一步,接收accept事件
        SelectionKey sk = serverSocket.register( selectors[0], SelectionKey.OP_ACCEPT);
        //attach callback object, Acceptor
        sk.attach(new Acceptor());
    }

    public void run() {
        try {
            while (!Thread.interrupted()) {
                for (int i = 0; i <2 ; i++) {
                    selectors[i].select();
                    Set selected =  selectors[i].selectedKeys();
                    Iterator it = selected.iterator();
                    while (it.hasNext())
                    {
                        //Reactor负责dispatch收到的事件
                        dispatch((SelectionKey) (it.next()));
                    }
                    selected.clear();
                }

            }
        } catch (IOException ex) {
            /* ... */ 
        }
    }

    void dispatch(SelectionKey k) {
        Runnable r = (Runnable) (k.attachment());
        //调用之前注册的callback对象
        if (r != null) {
            r.run();
        }
    }

    class Acceptor { // ...
        public synchronized void run() throws Exception {
            // 主selector负责accept
            SocketChannel connection = serverSocket.accept(); 
            if (connection != null) {
                // 选个subReactor去负责接收到的connection
                new Handler(connection, selectors[next]); 
            }
            if (++next == selectors.length) next = 0;
        }
    }
}
```