# 路由中心NameServer总结
## 作为路由注册中心，有哪些路由信息注册到了NameServer？

![](/images/RocketMQ/RocketMQ路由元信息.jpg)

## 路由注册
（1）RocketMQ路由注册是通过Broker与NameServer的心跳功能实现的。Broker启动时向集群中所有的NameServer发送心跳语句。
（2）Broker启动后，NameServe与Broker保持长连接。每隔30s向集群中所有NameServer发送心跳包，NameServer每收到一个心跳包，将更新brokerLiveTable中关于Broker的状态信息以及路由表(topicQueueTable、brokerAddrTable、brokerLiveTable、filterServerTable)。

## 路由删除
（1）NameServer收到Broker心跳包时会更新brokerLiveTable缓存中BrokerLivelnfo的lastUpdateTimestamp，然后NameServer每隔10s扫描brokerLiveTable，如果连续120s 没有收到心跳包，NameServer将移除该Broker的路由信息同时关闭Socket连接。
（2）Broker在正常被关闭情况下，会执行unregisterBroker命令。

### NameServer通常以集群部署，且集群中的各个节点互相不通信，当NameServer集群中各节点路由信息不一致时，RocketMQ如何保证可用性？
当NameServer路由信息不一致时，RocketMQ如何保证可用性？什么时候路由信息会出现不一致？举一个例子，如下图所示，NameServer集群和Broker集群均包含2个节点，假设此时发生网络分区（Network Partition），各个NameServer仅维护了分区内的Broker路由信息，Client也仅能根据分区内NameServer提供的路由信息发送/消费消息（暂不考虑Client端路由信息缓存）。

![](/images/RocketMQ/NameServer一致性.jpg)

进一步思考，在什么场景下需要保证强一致性（Strong Consistency）？结合CAP，P一定会发生，CA二选一，那么NameServer是否需要提供强一致性呢？结合上图，此时分区a中的Client只能将消息投递到分区内的Broker，也只能消费存储在分区内Broker的消息。如果在网络分区前已经有消息被投递到分区b，那么当网络分区恢复后，分区b中的Broker重新将路由信息注册至分区a中的NameServer，Client可以继续消费消息。

如果考虑顺序消息，此时条件更加苛刻：为保证消息的顺序性，需要满足从Provider到Broker到Consumer都是单点单线程。所谓单点，自然满足强一致性。因此，在这种场景下可用性是无法得到保证的。

至此，NameServer在设计上做了trade-off，在乱序消息的场景下，NameServer是一个AP系统，并不需要强一致性的保证。

## NameServer如何保证数据的最终一致？
NameServer作为一个路由服务，需要提供服务注册、服务剔除、服务发现这些基本功能，但是NameServer节点之间并不通信，在某个时刻各个节点数据可能不一致的情况下，如何保证客户端可以最终拿到正确的数据。下面分别从路由注册、路由剔除，路由发现三个角度进行介绍。

    （1）路由注册
        对于Zookeeper、Etcd这样强一致性组件，数据只要写到主节点，内部会通过状态机将数据复制到其他节点，Zookeeper使用的是Zab协议，etcd使用的是raft协议。
        但是NameServer节点之间是互不通信的，无法进行数据复制。RocketMQ采取的策略是，在Broker节点在启动的时候，轮训NameServer列表，与每个NameServer节点建立长连接，发起注册请求。NameServer内部会维护一个Broker表，用来动态存储Broker的信息。
        同时，Broker节点为了证明自己是存活的，会将最新的信息上报给NameServer，然后每隔30秒向NameServer发送心跳包，心跳包中包含BrokerId、Broker地址、Broker名称、Broker所属集群名称等，然后NameServer接收到心跳包后，会更新时间戳，记录这个Broker的最新存活时间。
        NameServer在处理心跳包的时候，存在多个Broker同时操作一张Broker表，为了防止并发修改Broker表导致不安全，路由注册操作引入了ReadWriteLock读写锁，这个设计亮点允许多个消息生产者并发读，保证了消息发送时的高并发，但是同一时刻NameServer只能处理一个Broker心跳包，多个心跳包串行处理。这也是读写锁的经典使用场景，即读多写少。

    （2）路由剔除
        正常情况下，如果Broker关闭，则会与NameServer断开长连接，Netty的通道关闭监听器会监听到连接断开事件，然后会将这个Broker信息剔除掉。
        异常情况下，NameServer中有一个定时任务，每隔10秒扫描一下Broker表，如果某个Broker的心跳包最新时间戳距离当前时间超多120秒，也会判定Broker失效并将其移除。
        特别的，对于一些日常运维工作，例如：Broker升级，RocketMQ提供了一种优雅剔除路由信息的方式。如在升级一个Master节点之前，可以先通过命令行工具禁止这个Broker的写权限，生产者发送到这个Broker的请求，都会收到一个NO_PERMISSION响应，之后会自动重试其他的Broker。当观察到这个broker没有流量后，再将这个broker移除。

    （3）路由发现
        路由发现是客户端的行为，这里的客户端主要说的是生产者和消费者。具体来说：对于生产者，可以发送消息到多个Topic，因此一般是在发送第一条消息时，才会根据Topic获取从NameServer获取路由信息。

        对于消费者，订阅的Topic一般是固定的，所在在启动时就会拉取。
        那么生产者/消费者在工作的过程中，如果路由信息发生了变化怎么处理呢？如：Broker集群新增了节点，节点宕机或者Queue的数量发生了变化。细心的读者注意到，前面讲解NameServer在路由注册或者路由剔除过程中，并不会主动推送会客户端的，这意味着，需要由客户端拉取主题的最新路由信息。
        事实上，RocketMQ客户端提供了定时拉取Topic最新路由信息的机制。DefaultMQProducer和DefaultMQConsumer有一个pollNameServerInterval配置项，用于定时从NameServer并获取最新的路由表，默认是30秒，它们底层都依赖一个MQClientInstance类。
        MQClientInstance类中有一个updateTopicRouteInfoFromNameServer方法，用于根据指定的拉取时间间隔，周期性的的从NameServer拉取路由信息。在拉取时，会把当前启动的Producer和Consumer需要使用到的Topic列表放到一个集合中，逐个从NameServer进行更新。
        然而定时拉取，还不能解决所有的问题。因为客户端默认是每隔30秒会定时请求NameServer并获取最新的路由表，意味着客户端获取路由信息总是会有30秒的延时。这就带来一个严重的问题，客户端无法实时感知Broker服务器的宕机。如果生产者和消费者在这30秒内，依然会向这个宕机的broker发送或消费消息呢？
        这个问题，可以通过客户端重试机制来解决。

## RocketMQ为什么自研nameserver而不用zk？
（1）RocketMQ只需要一个轻量级的维护元数据信息的组件，为此引入zk增加维护成本还强依赖另一个中间件了。RocketMQ追求的是AP，而不是CP，也就是需要高可用。
（2）zk是CP，因为zk节点间通过zap协议有数据共享，每个节点数据会一致，但是zk集群当挂了一半以上的节点就没法使用了。
（3）nameserver是AP，节点间不通信，这样会导致节点间数据信息会发生短暂的不一致，但每个broker都会定时向所有nameserver上报路由信息和心跳。当某个broker下线了，nameserver也会延时30s才知道，而且不会通知客户端（生产和消费者），只能靠客户端自己来拉，rocketMQ是靠消息重试机制解决这个问题的，所以是最终一致性。但nameserver集群只要有一个节点就可用。

## 当Broker不可用后，NameServer并不会立即将变更后的注册信息推送至 Client（Producer/Consumer），此时RocketMQ如何保证Client正常发送/消费消息？
这个在消息发送与消息消费时会介绍。

## 若NameServer集群全部宕机，RocketMQ还能正常运行吗？：
因为在消息发送端、消息消费端都有缓存topic以及Broker的路由信息，因此即使NameServer集群全部宕机，RocketMQ还是能正常运行。但新起的 Producer、Consumer、Broker就无法工作。

## 如何配置Namesrv地址到生产者和消费者？
将Namesrv地址列表提供给客户端(生产者和消费者)，有四种方法：

    （1）编程方式，就像producer.setNamesrvAddr("ip:port")
    （2）Java启动参数设置，使用rocketmq.namesrv.addr
    （3）环境变量，使用NAMESRV_ADDR
    （4）HTTP端点，例如说：http://namesrv.rocketmq.xxx.com 地址，通过DNS解析获得Namesrv真正的地址
    
## 请说说你对 Namesrv 的了解？
1、 Namesrv用于存储Topic、Broker关系信息，功能简单，稳定性高。
    
    多个Namesrv之间相互没有通信，单台Namesrv宕机不影响其它 Namesrv 与集群。
    多个Namesrv之间的信息共享，通过Broker主动向多个Namesrv都发起心跳。正如上文所说，Broker需要跟所有Namesrv连接。
    即使整个Namesrv集群宕机，已经正常工作的Producer、Consumer、Broker仍然能正常工作，但新起的Producer、Consumer、Broker就无法工作。这点和Dubbo有些不同，不会缓存Topic等元信息到本地文件。

2、 Namesrv压力不会太大，平时主要开销是在维持心跳和提供 Topic-Broker的关系数据。但有一点需要注意，Broker向Namesr发心跳时，会带上当前自己所负责的所有Topic信息，如果Topic个数太多（万级别），会导致一次心跳中，就Topic的数据就几十M，网络情况差的话，网络传输失败，心跳失败，导致Namesrv误认为Broker心跳失败。

另外，内网环境网络相对是比较稳定的，传输几十M问题不大。同时如果真的要优化，Broker可以把心跳包做压缩，再发送给Namesrv。不过这样也会带来 CPU的占用率的提升。