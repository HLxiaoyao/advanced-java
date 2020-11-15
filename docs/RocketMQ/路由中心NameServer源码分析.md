# 路由中心NameServer源码分析
## NameServer架构设计
消息中间件的设计思路一般基于主题的订阅发布机制消息生产者（Producer）发送某一主题的消息到消息服务器，消息服务器负责该消息的持久化存储，消息消费者(Consumer）订阅感兴趣的主题，消息服务器根据订阅信息（路由信息）将消息推送到消费者（PUSH模式）或者消息消费者主动向消息服务器拉取消息（PULL模式），从而实现消息生产者与消息消费者解调。

为了避免消息服务器的单点故障导致的整个系统瘫痪，通常会部署多台消息服务器共同承担消息的存储。那消息生产者如何知道消息要发往哪台消息服务器呢？如果某一台消息服务器若机了，那么生产者如何在不重启服务的情况下感知呢？

RocketMQ为了避免上述问题的产生设计了一个NameServer，用于类似注册中心一样去感知当前有哪些消息服务器（Broker）以及他们的状态。NameServer架构图如下所示：

![](/images/RocketMQ/NameServer架构图.png)

NameServer是一个非常简单的Topic路由注册中心，其角色类似Dubbo中的zookeeper，支持Broker的动态注册与发现。主要包括两个功能

    （1）Broker管理：Boroker启动时向所有的NameServer注册，NameServer接受Broker集群的注册信息并且保存下来作为路由信息的基本数据。NameServer与每台Broker服务器保持长连接，并每间隔30s检测Broker是否存活，若Broker宕机，则从注册表中将其移出。但是路由信息变更不会立即通知消息生产者，这样设计是为了降低NameServer实现的复杂性，在消息发送端提供容错机制来保证消息发送的高可用。
    （3）路由信息管理：每个NameServer将保存关于Broker集群的整个路由信息和用于客户端查询的队列信息。然后Producer和Conumser通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。例如消息生产者在发送消息之前先从NameServer获取Broker服务器地址列表，然后根据负载均衡算法从列表中选择一台消息服务器进行消息发送。
    
NameServer通常也是集群的方式部署，各实例间相互不进行信息通讯。Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完整的路由信息。当某个NameServer因某种原因下线了，Broker仍然可以向其它NameServer同步其路由信息。NameServer实例时间互不通信，这本身也是其设计亮点之一，即允许不同NameServer之间数据不同步(像Zookeeper那样保证各节点数据强一致性会带来额外的性能消耗)。

## NameServer启动流程
NameServer的启动类是org.apache.rocketmq.namesrv.NamesrvStartup，源码如下
```
public static NamesrvController main0(String[] args) {
    try {
        // 主要是初始化一些配置信息，比如namesrvConfig、nettyServerConfig信息，前者是加载namesrv的配置信息，后者是netty的相关配置信息
        NamesrvController controller = createNamesrvController(args);
        // 初始化、启动NamesrvController类
        start(controller);
        String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
        log.info(tip);
        System.out.printf("%s%n", tip);
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }
    return null;
}
```
### createNamesrvController()方法
```
public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
    // 设置RocketMQ版本号
    System.setProperty(RemotingCommand.REMOTING_VERSION_KEY, Integer.toString(MQVersion.CURRENT_VERSION));
    Options options = ServerUtil.buildCommandlineOptions(new Options());
    commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());
    if (null == commandLine) {
        System.exit(-1);
        return null;
    }

    // 创建NameServer业务参数
    final NamesrvConfig namesrvConfig = new NamesrvConfig();
    // 创建NameServer网络参数
    final NettyServerConfig nettyServerConfig = new NettyServerConfig();
    // 默认监听端口
    nettyServerConfig.setListenPort(9876);
    // 通过文件来配置参数
    if (commandLine.hasOption('c')) {
        String file = commandLine.getOptionValue('c');
        if (file != null) {
            InputStream in = new BufferedInputStream(new FileInputStream(file));
            properties = new Properties();
            properties.load(in);
            MixAll.properties2Object(properties, namesrvConfig);
            MixAll.properties2Object(properties, nettyServerConfig);
            namesrvConfig.setConfigStorePath(file);
            System.out.printf("load config properties file OK, %s%n", file);
            in.close();
        }
    }
    // 通过命令行参数
    if (commandLine.hasOption('p')) {
        InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
        MixAll.printObjectProperties(console, namesrvConfig);
        MixAll.printObjectProperties(console, nettyServerConfig);
        System.exit(0);
    }
    MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);

    if (null == namesrvConfig.getRocketmqHome()) {
        System.out.printf("Please set the %s variable in your environment to match the location of the RocketMQ installation%n", MixAll.ROCKETMQ_HOME_ENV);
        System.exit(-2);
    }

    /**
    * 初始化Logback日志工厂
    * RocketMQ默认使用Logback作为日志输出
    */
    LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
    JoranConfigurator configurator = new JoranConfigurator();
    configurator.setContext(lc);
    lc.reset();
    configurator.doConfigure(namesrvConfig.getRocketmqHome() + "/conf/logback_namesrv.xml");

    log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);
    MixAll.printObjectProperties(log, namesrvConfig);
    MixAll.printObjectProperties(log, nettyServerConfig);

    /**
    * 初始化NamesrvController
    * 该类是Name Server的主要控制类
    */
    final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

    controller.getConfiguration().registerConfig(properties);
    return controller;
}
```
从上面源码可以将createNamesrvController()方法概括为以下两个步骤：

    （1）解析参数初始化NamesrvConfig、NettyServerConfig对象
    （2）创建NamesrvController对象

#### 配置参数解析
上面代码首先解析参数到NamesrvConfig、NettyServerConfig对象，参数来源有如下两种方式：

    （1）-c configFile通过-c命令制定配置文件路径
    （2）使用“--属性名 属性值”的方式，例如--listenPort 9876
    
**（1）NamesrvConfig属性值：**
```
public class NamesrvConfig {
    // RocketMQ主目录
    private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));
    // NameServer存储KV配置属性的持久化路径
    private String kvConfigPath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "kvConfig.json";
    // NameServer默认配置文件路径，不生效。NameServer启动时如果要
通过配置文件配置NameServer启动属性的话，请使用-c选项
    private String configStorePath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "namesrv.properties";
    private String productEnvName = "center";
    private boolean clusterTest = false;
    // 是否支持顺序消息，默认是不支持
    private boolean orderMessageEnable = false;
}
```
**（2）NettyServerConfig属性值：**
```
public class NettyServerConfig implements Cloneable {
     // NettyServer监听端口，默认会被初始化为9876
	  private int listenPort = 8888;
	  // Netty业务线程池线程个数，处理如broker注册，topic路由信息查询、topic删除等与producer、broker交互request
    private int serverWorkerThreads = 8;
    // Netty public任务线程池线程个数，Netty网络设计根据业务类型会创建不同的线程池，比如处理消息发送、消息消费、心跳检测等。如果该业务类型（RequestCode）未注册线程池， 则由public线程池执行
    private int serverCallbackExecutorThreads = 0;
    // IO线程池线程个数，主要是NameServer、Broker端解析请求、返回相应的线程个数，这类线程主要是处理网络请求的，解析请求包，然后转发到各个业务线程池完成具体的业务操作，然后将结果再返回调用方。
    private int serverSelectorThreads = 3;
    // 单向发送信号量，防止单向发送请求并发度过高
    private int serverOnewaySemaphoreValue = 256;
    // 异步发送信号量，防止单向发送请求并发度过高
    private int serverAsyncSemaphoreValue = 64;
    // 网络连接最大空闲时间，默认120s。如果连接
空闲时间超过该参数设置的值，连接将被关闭
    private int serverChannelMaxIdleTimeSeconds = 120;
	 // 网络socket发送缓存区大小，默认64k
    private int serverSocketSndBufSize = NettySystemConfig.socketSndbufSize;
    // 网络socket接收缓存区大小，默认64k
    private int serverSocketRcvBufSize = NettySystemConfig.socketRcvbufSize;
    // ByteBuffer是否开启缓存，建议开启
    private boolean serverPooledByteBufAllocatorEnable = true;
    // 是否启用Epoll IO模型，Linux环境建议开启
    private boolean useEpollNativeSelector = false;
}
```
#### 创建NamesrvController实例
根据启动属性创建NamesrvController实例，NamesrvController实例为NameServer的核心控制器
```
public NamesrvController(NamesrvConfig namesrvConfig, NettyServerConfig nettyServerConfig) {
    this.namesrvConfig = namesrvConfig;
    this.nettyServerConfig = nettyServerConfig;
    this.kvConfigManager = new KVConfigManager(this);
    this.routeInfoManager = new RouteInfoManager();
    this.brokerHousekeepingService = new BrokerHousekeepingService(this);
    this.configuration = new Configuration(log,this.namesrvConfig, this.nettyServerConfig);
    this.configuration.setStorePathFromConfig(this.namesrvConfig, "configStorePath");
}
```
### NameServer启动--start()方法
```
public static NamesrvController start(final NamesrvController controller) throws Exception {
    if (null == controller) {
        throw new IllegalArgumentException("NamesrvController is null");
    }

    // 初始化controller实例
    boolean initResult = controller.initialize();
    if (!initResult) {
        controller.shutdown();
        System.exit(-3);
    }

    /**
    * 注册ShutdownHookThread服务。在JVM退出之前，调用ShutdownHookThread来进行关闭服务，释放资源
    */
    Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
        @Override
        public Void call() throws Exception {
            controller.shutdown();
            return null;
        }
    }));

    controller.start();
    return controller;
}
```
从上面源码可以看出，NameServer启动可以概括为以下两个步骤：

    （1）通过initialize()方法初始化NamesrvController实例
    （2）注册JVM钩子函数
    （3）启动NameServer

#### initialize()方法
```
public boolean initialize() {
    // 从kvConfigPath加载文件内容至KVConfigManager
    this.kvConfigManager.load();

    // 初始化RemotingServer
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

    // workerThread线程池，默认线程数为8
    this.remotingExecutor = Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

    // 注册requestProcessor，默认为DefaultRequestProcessor，用来处理netty接收到的信息
    this.registerProcessor();

    // 启动定时线程，延迟5秒执行，每隔10s判断broker是否依然存活
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 5, 10, TimeUnit.SECONDS);

    /**
    * 延迟1分钟启动、每10分钟执行一次的定时任务
    * 作用式打印出kvConfig配置
    */
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            NamesrvController.this.kvConfigManager.printAllPeriodically();
        }
    }, 1, 10, TimeUnit.MINUTES);

    if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
        try {
            fileWatchService = new FileWatchService(
                new String[] {
                    TlsSystemConfig.tlsServerCertPath,
                    TlsSystemConfig.tlsServerKeyPath,
                    TlsSystemConfig.tlsServerTrustCertPath
                },
                new FileWatchService.Listener() {
                    boolean certChanged, keyChanged = false;
                    @Override
                    public void onChanged(String path) {
                        if(path.equals(TlsSystemConfig.tlsServerTrustCertPath)) {
                            log.info("The trust certificate changed, reload the ssl context");
                            reloadServerSslContext();
                        }
                        if (path.equals(TlsSystemConfig.tlsServerCertPath)) {
                            certChanged = true;
                         }
                        if (path.equals(TlsSystemConfig.tlsServerKeyPath)) {
                            keyChanged = true;
                        }
                        if (certChanged && keyChanged) {
                            log.info("The certificate and private key changed, reload the ssl context");
                            certChanged = keyChanged = false;
                            reloadServerSslContext();
                        }
                    }
                    private void reloadServerSslContext() {
                        ((NettyRemotingServer)remotingServer).loadSslContext();
                    }
                });
        } catch (Exception e) {
            log.warn("FileWatchService created error, can't load the certificate dynamically");
        }
    }
    return true;
} 
```
从上述源码中分析，NameServerController的初始化可以概括为以下几个步骤：

    （1）读取kvConfigPath下的文件，并转化成类型为HashMap<String Namespace, HashMap<String Key, String Value>>的Map中，nameServer中相关配置信息存储，均采用线程不安全的HashMap容器存储，结合读写锁ReadWriteLock最大化并发度，KvConfig就是一个典型的例子，后续路由信息的存储也是同理。创建NettyServer网络处理对象
    （2）注册一个心跳机制的线程池，它会在启动后5秒开始每隔10秒扫描一次不活跃的broker。在定时任务中，NameServer会遍历RouteInfoManager#brokerLiveTable这个属性，该属性存储的是集群中所有broker的活跃信息，主要是BrokerLiveInfo#lastUpdateTimestamp属性，它描述了broker上一次更新的活跃时间戳。若lastUpdateTimestamp属性超过120秒未更新，则该broker会被视为失效并从brokerLiveTable中移除。
    （3）注册一个每隔10分钟打印一次KV配置。
    
#### 注册JVM钩子函数
注册JVM钩子函数，以便监听Broker、消息生产者的网络请求。这里是一种常用的编程技巧，若代码中使用了线程池，一种优雅停机的方式就是注册一个JVM钩子函数，在JVM进程关闭之前，先将线程池关闭，及时释放资源。

#### 启动NameServer--start()
```
public void start() throws Exception {
    // 启动remotingServer实例
    this.remotingServer.start();

    if (this.fileWatchService != null) {
        this.fileWatchService.start();
    }
}
```
## NameServer路由注册
NameServer主要作用是为消息生产者和消息消费者提供关于主题topic的路由信息，那么NameServer需要怎样存储路由的基础信息呢？
### 路由元信息
NameServer的路由实现类是org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager，可以通过该类知道NameServer到底存储哪些信息？
```
// Topic消息队列路由信息，消息发送时根据路由表进行负载均衡
private final HashMap<String topic, List<QueueData>> topicQueueTable;
// Broker基础信息，包含brokerName、所属集群名称、主备Broker地址
private final HashMap<String brokerName, BrokerData> brokerAddrTable;
//  Broker集群信息，存储集群中所有Broker名称
private final HashMap<String clusterName, Set<String brokerName>> clusterAddrTable;
// Broker状态信息。NameServer每次收到心跳包时会替换该信息
private final HashMap<String brokerAddr, BrokerLiveInfo> brokerLiveTable;
// Broker上的FilterServer列表，用于类模式消息过滤
private final HashMap<String brokerAddr, List<String> Filter Server> filterServerTable;
```
RocketMQ基于订阅发布机制，一个Topic拥有多个消息队列，一个Broker为每一主题默认创建4 个读队列4个写队列。多个Broker组成一个集群，BrokerName由相同的多台Broker组成Master-Slave架构，brokerld为0代表Master, 大于0表示Slave。BrokerLivelnfo中的lastUpdateTimestamp存储上次收到Broker心跳包的时间。

### 路由注册
RocketMQ路由注册是通过Broker与NameServer的心跳功能实现的。Broker启动时向集群中所有的NameServer发送心跳语旬，每隔30s向集群中所有NameServer发送心跳包，NameServer收到Broker心跳包时会更新brokerLiveTable缓存中BrokerLivelnfo的lastUpdateTimestamp，然后NameServer每隔10s扫描brokerLiveTable，如果连续120s 没有收到心跳包，NameServer将移除该Broker的路由信息同时关闭Socket连接。

#### NameServer处理心跳包
org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor网络处理器解析请求类型，如果请求类型为RequestCode.REGISTER_BROKER，则请求最终转发到RoutelnfoManager#registerBroker()
```
case RequestCode.REGISTER_BROKER:
        Version brokerVersion = MQVersion.value2Version(request.getVersion());
        if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
            return this.registerBrokerWithFilterServer(ctx, request);
        } else {
            return this.registerBroker(ctx, request);
        }
```
```
// DefaultRequestProcessor#registerBroker
public RemotingCommand registerBroker(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {
    
    // 省略
    ....
    // RoutelnfoManager#registerBroker()
    RegisterBrokerResult result = this.namesrvController.getRouteInfoManager().registerBroker(
            requestHeader.getClusterName(),
            requestHeader.getBrokerAddr(),
            requestHeader.getBrokerName(),
            requestHeader.getBrokerId(),
            requestHeader.getHaServerAddr(),
            topicConfigWrapper,
            null,
            ctx.channel()
    );
    
    // 省略
```
```
// RoutelnfoManager#registerBroker()
public RegisterBrokerResult registerBroker(final String clusterName, final String brokerAddr, final String brokerName, final long brokerId, final String haServerAddr, final TopicConfigSerializeWrapper topicConfigWrapper, final List<String> filterServerList, final Channel channel) {
    RegisterBrokerResult result = new RegisterBrokerResult();
    try {
        try {
            // 加锁
            this.lock.writeLock().lockInterruptibly();

            // 更新cluster和broker对应关系
            Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
            if (null == brokerNames) {
                brokerNames = new HashSet<String>();
                this.clusterAddrTable.put(clusterName, brokerNames);
            }
            brokerNames.add(brokerName);
            boolean registerFirst = false;

            // 更新brokername和brokerdata的分组
            BrokerData brokerData = this.brokerAddrTable.get(brokerName);
            if (null == brokerData) {
                registerFirst = true;
                brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
                this.brokerAddrTable.put(brokerName, brokerData);
            }
            Map<Long, String> brokerAddrsMap = brokerData.getBrokerAddrs();
            //将从设备切换为主设备：首先在namesrv中删除<1，IP：PORT>，然后添加<0，IP：PORT> 
            //同一IP：PORT在brokerAddrTable中必须只有一条记录
            Iterator<Entry<Long, String>> it = brokerAddrsMap.entrySet().iterator();
            while (it.hasNext()) {
                Entry<Long, String> item = it.next();
                if (null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey()) {
                    it.remove();
                }
            }

            String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
            registerFirst = registerFirst || (null == oldAddr);

            // 如果是master broker，第一次注册或者是topic信息发生变化了，更新topicQueueTable
            if (null != topicConfigWrapper && MixAll.MASTER_ID == brokerId) {
                if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion()) || registerFirst) {
                    ConcurrentMap<String, TopicConfig> tcTable =
                            topicConfigWrapper.getTopicConfigTable();
                    if (tcTable != null) {
                        for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                            this.createAndUpdateQueueData(brokerName, entry.getValue());
                        }
                    }
                }
            }

            // 更新broker的心跳时间
            BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr, new BrokerLiveInfo(
                    System.currentTimeMillis(),
                    topicConfigWrapper.getDataVersion(),
                    channel,
                    haServerAddr));
            if (null == prevBrokerLiveInfo) {
                log.info("new broker registered, {} HAServer: {}", brokerAddr, haServerAddr);
            }

            // 更新filter server table
            if (filterServerList != null) {
                if (filterServerList.isEmpty()) {
                    this.filterServerTable.remove(brokerAddr);
                } else {
                    this.filterServerTable.put(brokerAddr, filterServerList);
                }
            }

            // 如果是slave broker注册，如果master存在，则返回master broker信息
            if (MixAll.MASTER_ID != brokerId) {
                String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                if (masterAddr != null) {
                    BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                    if (brokerLiveInfo != null) {
                        result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                        result.setMasterAddr(masterAddr);
                    }
                }
            }
        } finally {
            this.lock.writeLock().unlock();
        }
    } catch (Exception e) {
        log.error("registerBroker Exception", e);
    }
    return result;
}
```
从上面源码分析，NameServer处理心跳包分为以下几个步骤：

    （1）首先加锁防止并发修改RoutelnfoManager中的路由表，判断Broker所属集群是否存在，如果不存在，则创建，然后将broker名加入到集群Broker集合中。即更新cluster和broker对应关系。
    （2）维护BrokerData信息，首先从brokerAddrTable根据brokerName尝试获取Broker（包括主备）信息，若不存在，则创建BrokerData并放入到brokerAddrTable，registerFirst设置为true；若存在，直接替换原来的，registerFirst设置为false，表示非第一次注册。
    （3）如果Broker为Master，并且BrokerTopic配置信息发生变化或者是初次注册，则需要创建或更新Topic路由元数据，填充topicQueueTable。其中包括MixALL.DEFAULT_TOPIC的路由信息。当消息生产者发送主题时，若该主题未创建并且BrokerConfig的autoCreateTopicEnable为true时，将返回MixALL.DEFAULT_TOPIC当路由信息。
    （4）更新BrokerLiveInfo的信息（例如broker的心跳时间），BrokerLiveInfo是存活Broker信息表，BrokerLiveInfo是执行路由删除的重要依据。
    （5）注册Broker的过滤器Server地址列表，一个Broker上会关联多个FilterServer消息过滤器。
    （6）若是Broker从节点注册，则需要查找该Broker的Master的节点信息，并更新对应的masterAddr属性。
    
NameServe与Broker保持长连接，Broker状态存储在brokerLiveTable中，NameServer每收到一个心跳包，将更新brokerLiveTable中关于Broker的状态信息以及路由表(topicQueueTable、brokerAddrTable、brokerLiveTable、filterServerTable)。更新上述路由表(HashTable)使用了锁粒度较少的读写锁，允许多个消息发送者(Producer)并发读，保证消息发送时的高并发。但同一时刻NameServer只处理一个Broker心跳包，多个心跳包请求串行执行。这也是读写锁经典使用场景

### 路由删除
NameServer会每隔10s扫描brokerLiveTable状态表，如果BrokerLive的
lastUpdateTimestamp的时间戳距当前时间超过120s，则认为Broker失效，移除该Broker，关闭与Broker连接，并同时更新topicQueueTable、brokerAddrTable、brokerLiveTable、filterServerTable。RocktMQ有两个触发点来触发路由删除：

    （1）NameServer定时扫描brokerLiveTable检测上次心跳包与当前系统时间的时间差，如果时间戳大于120s，则需要移除该Broker信息。
    （2）Broker在正常被关闭的情况下，会执行unregisterBroker指令。
    
由于不管是何种方式触发的路由删除，路由删除的方法都是一样的，就是从topicQueueTable、brokerAddrTable、brokerLiveTable、filterServerTable删除与该Broker相关的信息，但RocketMQ这两种方式维护路由信息时会抽取公共代码。以第一种方式为示例进行分析。在NameServerController初始化时会启动一个定时扫描brokerLiveTable状态表的定时任务，该定时任务所执行的代码如下：
```
public void scanNotActiveBroker() {
    Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
    // 遍历brokerLivelnfo路由表
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> next = it.next();
        // 检测BrokerLiveInfo lastUpdateTimestamp上次收到心跳包的时间如果超过当前时间120s
        long last = next.getValue().getLastUpdateTimestamp();
        if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
            // 若超过120s，则关闭Channel
            RemotingUtil.closeChannel(next.getValue().getChannel());
            // 从brokerLivelnfo路由表移除
            it.remove();
            log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
            this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
        }
    }
}
```
scanNotActiveBroker在NameServer中每10s执行一次。遍历brokerLivelnfo路由表（HashMap），检测BrokerLiveInfo lastUpdateTimestamp上次收到心跳包的时间如果超过当前时间120s, NameServer则认为该Broker已不可用，故需要将它移除，关闭Channel，然后删除与该Broker相关的路由信息，路由表维护过程，需要申请写锁。具体的路由表维护是在RouteInfoManager#onChannelDestroy()方法中。

**（1）依据Channel从brokerLiveTable中需要移除的brokerAddr：**
```
String brokerAddrFound = null;
if (channel != null) {
    try {
        try {
            // 加锁，查找与channel匹配的BrokerLiveInfo
            this.lock.readLock().lockInterruptibly();
            Iterator<Entry<String, BrokerLiveInfo>> itBrokerLiveTable = this.brokerLiveTable.entrySet().iterator();
            while (itBrokerLiveTable.hasNext()) {
                Entry<String, BrokerLiveInfo> entry = itBrokerLiveTable.next();
                if (entry.getValue().getChannel() == channel) {
                    brokerAddrFound = entry.getKey();
                    break;
                }
            }
        } finally {
            this.lock.readLock().unlock();
        }
    } catch (Exception e) {
        log.error("onChannelDestroy Exception", e);
    }
}

if (null == brokerAddrFound) {
    brokerAddrFound = remoteAddr;
} else {
    log.info("the broker's channel destroyed, {}, clean it's data structure at once", brokerAddrFound);
}
```
遍历brokerLiveTable表，找到与需要删除的Channel相匹配的brokerAddr，若没有找到，则使用参数传递进来的remoteAddr。

**（2）brokerLiveTable与filterServerTable移除：**
```
try {
    try {
        // 加锁
        this.lock.writeLock().lockInterruptibly();
        // brokerLiveTable移除
        this.brokerLiveTable.remove(brokerAddrFound);
        // filterServerTable移除
        this.filterServerTable.remove(brokerAddrFound);
        // 省略部分
        ...
    }
}
```
**（3）brokerAddrTable移除：**
```
try {
    try {
        // 省略部分
        ....
        
        String brokerNameFound = null;
        boolean removeBrokerName = false;
        // brokerAddrTable移除
        Iterator<Entry<String, BrokerData>> itBrokerAddrTable = this.brokerAddrTable.entrySet().iterator();
        while (itBrokerAddrTable.hasNext() && (null == brokerNameFound)) {
            BrokerData brokerData= itBrokerAddrTable.next().getValue();
            Iterator<Entry<Long, String>> it = brokerData.getBrokerAddrs().entrySet().iterator();
            while (it.hasNext()) {
                Entry<Long, String> entry = it.next();
                Long brokerId = entry.getKey();
                String brokerAddr = entry.getValue();
                if (brokerAddr.equals(brokerAddrFound)) {
                    brokerNameFound = brokerData.getBrokerName();
                    it.remove();
                    break;
                }
            }

            if (brokerData.getBrokerAddrs().isEmpty()) {
                removeBrokerName = true;
                itBrokerAddrTable.remove();
            }
        }
        // 省略部分
        ...
    }
}
```
遍历brokerAddrTable，然后从brokerAddrTable的BrokerData中找到与需要删除的brokerAddrp匹配的Broker，从BrokerData中移除，若移除后在BrokerData中不再包含其他Broker，则在brokerAddrTable中移除该brokerName对应的条目。

**（4）clusterAddrTable移除：**
根据BrokerName，从clusterAddrTable中找到Broker并从集群中移除。若移除后，集群中不包含其他任何Broker，则将该集群从clusterAddrTable中移除。

**（5）topicQueueTable移除：**
根据BrokerName，遍历所有主题的队列，若队列中包含了当前Broker的队列，则移除；若topic只包含待移除Broker的队列的话，从路由表中删除该topic。

### 路由发现
RocketMQ路由发现是非实时的，当Topic路由出现变化后，NameServer不主动推送给客户端， 而是由客户端定时拉取主题最新的路由。根据主题名称拉取路由信息的命令编码为：GET_ROUTEINTOBY_TOPIC。NameServer路由发现实现类： DefaultRequestProcessor#getRouteInfoByTopic。

#### 路由结果
NameServer返回的路由结果由TopicRouteData来进行封装，源码如下：
```
public class TopicRouteData extends RemotingSerializable {
    // 顺序消息配置内容，来自于kvConfig
    private String orderTopicConf;
    // topic队列元数据
    private List<QueueData> queueDatas;
    // topic分布的broker元数据
    private List<BrokerData> brokerDatas;
    // broker上过滤服务器地址列表
    private HashMap<String brokerAddr, List<String> Filter Server> filterServerTable;
}
```















