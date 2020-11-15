# RocketMQ消息发送源码分析
RocketMQ一般消息发送的话，主要经历几个步骤：验证消息、查找路由、消息发送。消息发送通过DefaultMQProducer#send()方法实现，代码如下：
```
public SendResult send(Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    Validators.checkMessage(msg, this);
    msg.setTopic(withNamespace(msg.getTopic()));
    return this.defaultMQProducerImpl.send(msg);
}

public SendResult send(Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return send(msg, this.defaultMQProducer.getSendMsgTimeout());
}

public SendResult send(Message msg,long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return this.sendDefaultImpl(msg, CommunicationMode.SYNC, null, timeout);
}
```
从上面代码分析看出，消息发送最终是调用DefaultMQProducerImpl#sendDefaultImpl()方法实现的，并且默认消息发送以同步方式发送，默认超时时间为3s。sendDefaultImpl方法声明及参数解释如下：
```
[DefaultMQProducerImpl.java]
private SendResult sendDefaultImpl(
    // 消息发送实体
    Message msg,   
    // 发送类别，枚举类型：同步发送、异步发送、单向发送                          
    final CommunicationMode communicationMode,
     // 如果是异步发送方式，则需要实现SendCallback回调  
    final SendCallback sendCallback, 
    // 超时时间          
    final long timeout                          
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
```
## 消息长度验证
消息发送之前，首先确保生产者处于运行状态，然后验证消息是否符合相应的规范，具体的规范要求是主题名称、消息体不能为空、消息长度不能等于0且默认不能超过允许发送消息的最大长度4M。
```
// 校验producer是否处于运行状态
this.makeSureStateOK();
// 校验消息格式,比如topic、是否有消息内容、消息内容长度
Validators.checkMessage(msg, this.defaultMQProducer);
// 调用id，用于下面打印日志，标记为同一次发送消息
final long invokeID = random.nextLong();
```
## 查找主题路由信息
消息发送之前，首先需要获取主题的路由信息，只有获取了这些信息才能知道消息要发送到具体的Broker节点。
```
// DefaultMQProducerImpl#sendDefaultImpl 获取topic路由信息
TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());

DefaultMQProducerImpl#tryToFindTopicPublishInfo
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    // 缓存中获取topic的路由信息
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    // 当无可用的Topic路由信息时，从Namesrv获取一次
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }

    // 本地有缓存，或者从NameSrv上查询到了，此处便会直接返回所找到的路由信息：topicPublishInfo。
    // 但是如果Topic事先没有在任何Broker上进行配置，那么Broker在向NameSrv注册路由信息时便不会带上该Topic的路由，所以生产者也就无法从NameSrv中查询到该Topic的路由了。
    // 这个时候会进入else分支，再次从NameSrv 获取路由信息
    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else {
        // 这次调用 updateTopicRouteInfoFromNameServer（）时，传入的参数 isDefault 为true，那么代码自然就进入了上面的 if 分支中
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```
tryToFindTopicPublishInfo查找topic的路由信息方法。返回的路由信息为TopicPublishInfo，包括的信息如下：
```
public class TopicPublishInfo {
    // 是否是顺序消息
    private boolean orderTopic = false;
    private boolean haveTopicRouterInfo = false;
    // 该topic的消息队列
    private List<MessageQueue> messageQueueList = new ArrayList<MessageQueue>();
    // 每选择一次消息队列，该值会自增1，用于选择消息队列
    private volatile ThreadLocalIndex sendWhichQueue = new ThreadLocalIndex();
    private TopicRouteData topicRouteData;
}

public class TopicRouteData extends RemotingSerializable {
    private String orderTopicConf;
    // topic队列元数据
    private List<QueueData> queueDatas;
    // topic分布的broker元数据
    private List<BrokerData> brokerDatas;
    // broker上过滤服务器地址列表
    private HashMap<String brokerAddr, List<String> Filter Server> filterServerTable;
}
```
tryToFindTopicPublishInfo查找topic的路由主要分为以下两步：

    （1）若生产者中缓存了topic的路由信息，并且该路由信息中包含了消息队列，则直接返回该路由信息。
    （2）若没有缓存或没有包含消息队列，则向NameServer查询该topic的路由信息。若最终没有找到路由信息，则抛出异常。
    
第一步比较简单，只是一个简单的缓存读取操作，而第二步则是通过MQClientInstance#updateTopicRouteInfoFromNameServer方法实现的。

### NameServer查询topic路由信息
第一次发送消息时，本地没有缓存topic的路由信息，查询NameServer尝试获取，若路由信息未找到，则尝试用默认主题去查询，
```
public boolean updateTopicRouteInfoFromNameServer(final String topic) {
    return updateTopicRouteInfoFromNameServer(topic, false, null);
}
```
从上面源码可以看出，调用了类中的重载方法updateTopicRouteInfoFromNameServer(final String topic, boolean isDefault, DefaultMQProducer defaultMQProducer)，该方法可以拆分成以下几部分，首先从NameServer查询topic的路由信息。
```
if (isDefault && defaultMQProducer != null) {
    topicRouteData = this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(defaultMQProducer.getCreateTopicKey(),1000 * 3);
    if (topicRouteData != null) {
        for (QueueData data : topicRouteData.getQueueDatas()) {
            int queueNums = Math.min(defaultMQProducer.getDefaultTopicQueueNums(), data.getReadQueueNums());
            data.setReadQueueNums(queueNums);
            data.setWriteQueueNums(queueNums);
        }
    }
} else {
    // 远程调用RPC，从NameSrv获取topic的路由信息
    topicRouteData = this.mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3);
}
```
若isDefault为true，则使用默认主题去查询，若查到路由信息，则替换路由信息中读写队列个数为消息生产者中默认的队列个数。该默认Topic为TBW102，这个Topic就是用来创建其他Topic所用的。后面分析Topic创建机制会介绍。如果某Broker配置了autoCreateTopicEnable，允许自动创建Topic，那么在该Broker启动后，便会向自己的路由表中插入TBW102这个Topic，并注册到NameSrv，表明处理该Topic类型的消息。所以当消息所属的Topic，暂且叫Topic X吧，它本身没有在任何Broker上配置的时候，生产者就会查询Topic TBW102的路由信息，暂时作为Topic X的的路由，并插入到本地路由表中。当TopicX利用该路由发送到 Broker后，Broker发现自己并没有该Topic信息后，便会创建好该Topic，并更新到NameSrv中，表明后续接收TopicX的消息。
若isDefault为false，则使用参数topic去查询；若未查询到路由信息，则返回false，表示路由信息未变化。
在NameServer查询topic的路由信息topicRouteData后，就与本地缓存的topic路由元数据进行对比（不管本地是否缓存），看路由信息是否变化，是否需要更改本地缓存？
```
if (topicRouteData != null) {
    // topicRouteTable缓存的是topic与topic队列、broker元数据之间的映射关系
    // 即topic与TopicRouteData之间的映射关系，TopicRouteData是NameServer返回的topic路由信息元数据
    TopicRouteData old = this.topicRouteTable.get(topic);
    // 对比路由信息是否发生变化
    boolean changed = topicRouteDataIsChange(old, topicRouteData);
    if (!changed) {
        // 若没有发生变化时，这个方法会判断topic在本地是否存在，不存在还是属于需要更新
        changed = this.isNeedUpdateTopicRouteInfo(topic);
    } else {
        log.info("the topic[{}] route info changed, old[{}] ,new[{}]", topic, old, topicRouteData);
    }
    // 省略无关代码
    .....
}
```
上述代码块分析了是否需要更新本地缓存的路由缓存信息，若不需要更新，则方法结束；若需要更新本地路由缓存，那么需要更新哪些信息呢？
```
if (changed) {
    // 省略无关代码
    .....
    
    // 克隆一个TopicRouteData
    TopicRouteData cloneTopicRouteData = topicRouteData.cloneTopicRouteData();

    // 获取当前topic存在的borker集合
    for (BrokerData bd : topicRouteData.getBrokerDatas()) {
        this.brokerAddrTable.put(bd.getBrokerName(), bd.getBrokerAddrs());
    }
    // 省略无关代码
    .....
}
```
首先获取topic对应的broker元数据，更新本地缓存的topic存在的borker集合。
```
{
    // 省略无关代码
    .....
    
    // 更新发布的topic, 针对生产者，即将topicRouteData转成TopicPublishInfo
    TopicPublishInfo publishInfo = topicRouteData2TopicPublishInfo(topic, topicRouteData);
    publishInfo.setHaveTopicRouterInfo(true);

    // 更新该生产者组中每个生产者的topicPublishInfoTable
    Iterator<Entry<String, MQProducerInner>> it = this.producerTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, MQProducerInner> entry = it.next();
        MQProducerInner impl = entry.getValue();
        if (impl != null) {
            impl.updateTopicPublishInfo(topic, publishInfo);
        }
    }
    // 省略无关代码
    .....
}
```
然后调用topicRouteData2TopicPublishInfo方法，将topicRouteData元数据转成TopicPublishInfo。在topicRouteData2TopicPublishInfo方法内会循环遍历路由信息的QueueData信息，若队列没有写权限，则继续遍历下一个QueueData；根据brokerName找到brokerData信息，找不到或没有找到Master，则遍历下一个QueueData；根据写队列个数，根据topic+序列号创建MessageQueue，填充TopicPublishInfo的List<MessageQueue>。代码如下
```
public static TopicPublishInfo topicRouteData2TopicPublishInfo(final String topic, final TopicRouteData route) {
    TopicPublishInfo info = new TopicPublishInfo();
    // 设置TopicPublishInfo中的topic路由元数据
    info.setTopicRouteData(route);
    if (route.getOrderTopicConf() != null && route.getOrderTopicConf().length() > 0) {
        String[] brokers = route.getOrderTopicConf().split(";");
        for (String broker : brokers) {
            String[] item = broker.split(":");
            int nums = Integer.parseInt(item[1]);
            for (int i = 0; i < nums; i++) {
                MessageQueue mq = new MessageQueue(topic, item[0], i);
                info.getMessageQueueList().add(mq);
            }
        }
        info.setOrderTopic(true);
    } else {
        // 获取topic对应的队列元数据
        List<QueueData> qds = route.getQueueDatas();
        Collections.sort(qds);
        // 遍历topic下的队列，QueueData是topic在每一个broker上的队列配置
        for (QueueData qd : qds) {
            // 是否是可写队列
            if (PermName.isWriteable(qd.getPerm())) {
                BrokerData brokerData = null;
                // 查询QueueData对应的BrokerData
                for (BrokerData bd : route.getBrokerDatas()) {
                    if (bd.getBrokerName().equals(qd.getBrokerName())) {
                        brokerData = bd;
                        break;
                    }
                }

                if (null == brokerData) {
                    continue;
                }

                // 只有Master节点的Broker才能接收消息，对于非Master节点的需要过滤掉
                if (!brokerData.getBrokerAddrs().containsKey(MixAll.MASTER_ID)) {
                    continue;
                }

                // 按照QueueData配置的写队列个数，生成对应数量的MessageQueue。
                for (int i = 0; i < qd.getWriteQueueNums(); i++) {
                    MessageQueue mq = new MessageQueue(topic, qd.getBrokerName(), i);
                    info.getMessageQueueList().add(mq);
                }
            }
        }
        info.setOrderTopic(false);
    }
    return info;
}
```

从上述源码可以总结为以下几个步骤

    （1）若isDefault为true，则使用默认主题去查询，若查到路由信息，则替换路由信息中读写队列个数为消息生产者中默认的队列个数。若isDefault为false，则使用参数topic去查询；若未查询到路由信息，则返回false，表示路由信息未变化。
    （2）如果路由信息找到，与本地缓存中的路由信息进行对比，判断路由信息是否发生了改变，如果未发生改变，则直接返回false。
    （3）若路由信息发生变化，则更新MQClientInstance Broker地址缓存表brokerAddrTable。
    （4）若路由信息发生变化，则根据NameServer返回的TopicRouteData中的List<QueueData>转换成List<MessageQueue>列表。具体实现在topicRouteData2TopicPublishInfo()方法中，然后会更新该MQClientInstance所管理的所有消息发送关于topic的路由信息。
    
## 选择消息队列
如果找到路由信息会根据它选择消息队列，返回的消息队列会按照broker、序号排序。例如topicA在broker-a，broker-b上分别创建了4个队列，那么返回的消息队列：［｛“ brokerName”:”broker-a”,”queueId”:0}, {“brokerName”:”broker-a”,”queueId”:1},{“brokerName”: ”broker-a”, “queueId”:2},｛“BrokerName”：“broker-a”，”queueId” ：3，
{“brokerName”：”broker-b”,”queueId”:0}, {“brokerName”：”broker-b”，“queueId”：1}, {“brokerName”:”broker-b”,”queueId”:2}, {“brokerName”:"broker-b”，”queueId” ：3 }]，那RocketMQ如何选择消息队列呢？

首先消息发送端采用重试机制，由retryTimesWhenSendFailed参数指定同步方式重试次数，异步重试机制在收到消息发送结构后执行回调之前进行重试。由retryTimesWhenSendAsyncFailed指定，接下来就是循环执行，选择消息队列、发送消息，发送成功则返回，收到异常则重试。代码如下：
```
// 省略无关代码
.....
// timesTotal表示可以重试的次数
for (; times < timesTotal; times++) {
    // lastBrokerName表示上一次发送的BrokerName
    String lastBrokerName = null == mq ? null : mq.getBrokerName();
    // 选择某个Queue 用来发送消息
    MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
}
// 省略无关代码
.....

// DefaultMQProducerImpl#selectOneMessageQueue
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    // MQFaultStrategy#selectOneMessageQueue
    return this.mqFaultStrategy.selectOneMessageQueue(tpInfo, lastBrokerName);
}
```
虽然在消息发送的方法中，选择消息队列都是selectOneMessageQueue()方法一个入口，但是该方法内存在两种选择消息队列的方式。

    （1）sendLatencyFaultEnable=false，默认不启用Broker故障延迟机制。
    （2）sendLatencyFaultEnable=true，启用Broker故障延迟机制。
    
### 默认机制（不启动故障延迟）
当sendLatencyFaultEnable=false时，最终调用TopicPublishInfo#selectOneMessageQueue()方法来选择消息队列。
```
// MQFaultStrategy#selectOneMessageQueue
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    // 开启sendLatencyFaultEnable开关
    if (this.sendLatencyFaultEnable) {
        // 代码省略
        ....
    }
    // 上一次发送的broker中随机选一个队列
    return tpInfo.selectOneMessageQueue(lastBrokerName);
}

// TopicPublishInfo#selectOneMessageQueue
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
    if (lastBrokerName == null) {
        // 当lastBrokerName为空时，同样将计数器进行自增，按照计数器数值对MessageQueue总个数进行取模，再根据取模结果，取出MessageQueue列表中的某个Queue，直接返回。
        return selectOneMessageQueue();
    } else {
        // 当lastBrokerName不为空时，将计数器进行自增，再遍历TopicPulishInfo中的MessageQueue列表，按照计数器数值对MessageQueue总个数进行取模，
        // 再根据取模结果，取出MessageQueue列表中的某个Queue，并判断Queue所属Broker的Name是否和lastBrokerName一致，一致则继续遍历。
        int index = this.sendWhichQueue.getAndIncrement();
        for (int i = 0; i < this.messageQueueList.size(); i++) {
            int pos = Math.abs(index++) % this.messageQueueList.size();
            if (pos < 0)
                pos = 0;
            MessageQueue mq = this.messageQueueList.get(pos);
            if (!mq.getBrokerName().equals(lastBrokerName)) {
                return mq;
            }
        }
        return selectOneMessageQueue();
    }
}
```
首先在一次消息发送过程中，可能会多次执行选择消息队列这个方法，lastBrokerName就是上一次选择的执行发送消息失败的Broker。第一次执行消息队列选择时，lastBrokerName为null，此时直接用sendWhichQueue自增再获取值，与当前路由表中消息
队列个数取模，返回该位置的MessageQueue(selectOneMessageQueue()方法）。源码如下：
```
public MessageQueue selectOneMessageQueue() {
    // 计数器自增
    int index = this.sendWhichQueue.getAndIncrement();
    // 按照计数器数值对MessageQueue总个数进行取模
    int pos = Math.abs(index) % this.messageQueueList.size();
    if (pos < 0)
        pos = 0;
    // 再根据取模结果，取出MessageQueue列表中的某个Queue，直接返回
    return this.messageQueueList.get(pos);
}
```
如果消息发送再失败的话，下次进行消息队列选择时规避上次MesageQueue所在的Broker，否则还是很有可能再次失败。

该算法在一次消息发送过程中能成功规避故障的Broker，但如果Broker若机，由于路由算法中的消息队列是按Broker排序的，如果上一次根据路由算法选择的是宕机的Broker的第一个队列，那么随后的下次选择的是宕机Broker的第二个队列，消息发送很有可能会失败，再次引发重试，带来不必要的性能损耗。那么有什么方法在一次消息发送失败后，暂时将该Broker排除在消息队列选择范围外呢?
还有，Broker不可用后路由信息中为什么还会包含该Broker的路由信息呢？这是因为，首先 NameServer检测Broker是否可用是有延迟的，最短为一次心跳检测间隔(1Os)；其次，NameServer不会检测到Broker宕机后马上推送消息给消息生产者，而是消息生产者每隔30s更新一次路由信息，所以消息生产者最快感知Broker最新的路由信息也需要30s。如果能引人一种机制，在Broker宕机期间，如果一次消息发送失败后，可以将该Broker暂时排除在消息队列的选择范围中。

### Broker故障延迟机制
```
// MQFaultStrategy#selectOneMessageQueue
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    // 开启sendLatencyFaultEnable开关
    if (this.sendLatencyFaultEnable) {
        try {
            // 计数器自增
            int index = tpInfo.getSendWhichQueue().getAndIncrement();
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                // 判断该broker是否有延迟
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                    // 若没有延迟，则判断是否是上一次发送的broker
                    if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                        return mq;
                }
            }

            // 获取一个broker
            final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
            int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
            // 判断拿到的broker的可写队列数是否大于0
            if (writeQueueNums > 0) {
                // selectOneMessageQueue获取一个队列
                final MessageQueue mq = tpInfo.selectOneMessageQueue();
                if (notBestBroker != null) {
                    mq.setBrokerName(notBestBroker);
                    mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
                }
                return mq;
            } else {
                latencyFaultTolerance.remove(notBestBroker);
            }
        } catch (Exception e) {
            log.error("Error occurred when selecting message queue", e);
        }
        return tpInfo.selectOneMessageQueue();
    }
    // 省略代码，不开启sendLatencyFaultEnable开关
    ....
}
```
开启sendLatencyFaultEnable开关的情况下，消息队列选择可以概括为以下几个步骤：

    （1）根据对消息队列进行轮询或一个消息队列
    （2）验证该消息队列是否可用，latencyFaultTolerance.isAvailable()方法，后面详细分析。
    （3）如果返回MessageQueue可用，移除latencyFaultTolerance关于该topic条目，说明该Broker故障已经恢复。
    
#### Broker故障延迟机制核心类

    （1）LatencyFaultTolerance：延迟机制接口规范
    （2）FaultItem：失败条目（规避规则条目），LatencyFaultToleranceImpl内部类，该类的属性如下：
        private final String name; // 条目唯一键，这里是brokerName
        private volatile long currentLatency; // 本次消息发送延迟
        private volatile long startTimestamp; // 故障规避开始时间
    （3）MQFaultStrategy：消息失败策略，延迟实现的门面类，该类下的几个重要属性：
        // 根据currentLatency为发送消息的执行时间，从latencyMax尾部向前找到第一个比currentLatency小的值的索引index，如果没有找到，则返回0。然后根据这个索引从notAvailableDuration数组中取出对应的时间，这个时长内，Broker将设置为不可用。
        private long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L};
        private long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L};
        
下面从源码分析Broker故障延迟机制，下面是DefaultMQProducerImpl#sendDefaultImpl()方法部分代码，其作用是在消息发送之后更新本次发送消息的broker是否可用，以及是否需要在一段时间内设置为不可用。
```
long beginTimestampFirst = System.currentTimeMillis();
long beginTimestampPrev = beginTimestampFirst;
// 发送消息
sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
// 发送消息结束时间
endTimestamp = System.currentTimeMillis();
this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);

/**
* @param brokerName broker名称
* @param currentLatency 本次消息发送执行时间（发送了多久）
* @param isolation isolation是否隔离，该参数为true，则使用默认时长30s来计算Broker故障规避时长，若为false，则使用本次消息发送执行时间来计算broker故障规避时长
*/
public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
    this.mqFaultStrategy.updateFaultItem(brokerName, currentLatency, isolation);
}

public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
    if (this.sendLatencyFaultEnable) {
        // 该参数为true，则使用默认时长30s来计算Broker故障规避时长，若为false，则使用本次消息发送执行时间来计算broker故障规避时长
        long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
        this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);
    }
}
```
从上述源码看出，如果isolation为true，则使用30s作为computeNotAvailableDuration方法的参数；如果isolation为false，则使用本次消息发送时延作为 computeNotAvailableDuration方法的参数，那computeNotAvailableDuration的作用是计算因本次消息发送故障需要将Broker规避的时长，也就是接下来多久的时间内该Broker将不参与消息发送队列负载。

具体算法：从latencyMax数组尾部开始寻找，找到第一个比currentLatency小的下标，然后从notAvailableDuration数组中获取需要规避的时长，该方法最终调用 LatencyFaultTolerance的updateFaultltem。源码如下：
```
public void updateFaultItem(final String name, final long currentLatency, final long notAvailableDuration) {
    // 通过brokerName找到对应的FaultItem
    FaultItem old = this.faultItemTable.get(name);
    // 若brokerName对应的FaultItem为null
    if (null == old) {
        // 新建
        final FaultItem faultItem = new FaultItem(name);
        // currentLatency为发送消息的执行时间，根据执行时间来看落入哪个区间，在0~100的时间内notAvailableDuration都是0，都是可用的，
        // 大于该值后，可用的时间就会开始变大了，而在报错的时候isolation参数为true，那么该broker在600000毫秒后才可用(感觉时间有点久)
        faultItem.setCurrentLatency(currentLatency);
        // startTimestamp=当前时间+notAvailableDuration
        // notAvailableDuration为notAvailableDuration数组某个位置的值
        // currentLatency如果大于等于50小于100，则notAvailableDuration为0
        // currentLatency如果大于等于100小于550，则notAvailableDuration为0
        // currentLatency如果大于等于550小于1000，则notAvailableDuration为300000
        // 即只有当前时间大于startTimestamp时间才可用
        faultItem.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);

        old = this.faultItemTable.putIfAbsent(name, faultItem);
        if (old != null) {
            // 更新currentLatency与startTimestamp
            old.setCurrentLatency(currentLatency);
            old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
        }
    } else {
        // 更新currentLatency与startTimestamp
        old.setCurrentLatency(currentLatency);
        old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
    }
}
```
根据broker名称从缓存表中获取FaultItem，若未找到，则创建FaultItem，若找到则更新FaultItem，注意：
    
    （1）currentLatency、startTimestamp被volatile修饰
    （2）startTimestamp是当前系统时间加上需要规避的时长。startTimestamp是怕奴蛋当前broker是否可用的直接依据。
    
## 消息发送
```
// DefaultMQProducerImpl#sendDefaultImpl
// 发送消息
sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
```
从上面代码看，消息发送通过调用DefaultMQProducerImpl#sendKernelImpl()方法实现的，首先看下该方法的参数：
```
/**
* @param msg 待发送的消息
* @param mq 消息待发送的队列，该队列是通过selectOneMessageQueue选择的
* @param communicationMode 消息发送模式：同步、异步、单向
* @param sendCallback 如果是异步发送，则需要实现SendCallback
* @param topicPublishInfo topic对应的路由信息表
* @param timeout 发送超时时间，由客户端指定
*/
private SendResult sendKernelImpl(final Message msg,
        final MessageQueue mq,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback,
        final TopicPublishInfo topicPublishInfo,
        final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
```
下面就将DefaultMQProducerImpl#sendKernelImpl()方法拆分一步步分析消息发送的过程。

### 获取broker网络地址
```
// 通过brokerName获取broker，只返回brokerName对应主节点的地址
String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());

// 若broker地址为空，再次尝试获取
if (null == brokerAddr) {
    tryToFindTopicPublishInfo(mq.getTopic());
    brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
}
```
根据MessageQueue获取Broker的网络地址（只返回brokerName对应的master节点的网络地址）。如果MQClientlnstance的brokerAddrTable未缓存该Broker的信息，则从NameServer主动更新一下topic的路由信息。如果路由更新后还是找不到Broker信息，则抛出MQClientException，提示Broker不存在。

### 为消息分配全局唯一ID
```
// 省略无关代码
.....
if (brokerAddr != null) {
    // 根据broker地址计算得到VIP通道地址
    brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);
    // 获取消息体byte数组
    byte[] prevBody = msg.getBody();
    try {
        // 接着对消息进行前置处理，为消息分配全局唯一Id，对于批量消息，它的全局唯一id是单独生成的
        if (!(msg instanceof MessageBatch)) {
            MessageClientIDSetter.setUniqID(msg);
        }

        boolean topicWithNamespace = false;
        if (null != this.mQClientFactory.getClientConfig().getNamespace()) {
            msg.setInstanceId(this.mQClientFactory.getClientConfig().getNamespace());
            topicWithNamespace = true;
        }

        int sysFlag = 0;
        boolean msgBodyCompressed = false;
        // 判断消息是否需要压缩，若需要压缩，则计算最新的sysFlag
        if (this.tryToCompressMessage(msg)) {
            sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
            msgBodyCompressed = true;
        }

        // 获取消息属性，key=PROPERTY_TRANSACTION_PREPARED = "TRAN_MSG";
        // 判断是否为事务消息
        final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
        // 如果是事务消息，通过sysFlag与TRANSACTION_PREPARED_TYPE按位或，计算最新的sysFlag
        if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
            sysFlag |= MessageSysFlag.TRANSACTION_PREPARED_TYPE;
        }
        // 省略无关代码
        .....
    }
}
```
为消息分配全局唯一ID，若消息体默认超过4k，则会对消息体使用zip压缩，并且设置消息的系统标记为MessageSysFlag.COMPRESSED_FLAG（已压缩，但是批量消息不进行压缩）
，如果是事务Prepared消息，则设置消息的系统标记为MessageSysFlag.TRANSACTION_PREPARED TYPE。

### 执行前置钩子函数
```
/**
* 如果发送时注册了发送钩子方法，则先执行该发送钩子逻辑进行前置增强，这种方式类似于切面的逻辑
*/
if (this.hasSendMessageHook()) {
    // 设置消息发送上下文
    context = new SendMessageContext();
    context.setProducer(this);
        context.setProducerGroup(this.defaultMQProducer.getProducerGroup());
    context.setCommunicationMode(communicationMode);
    context.setBornHost(this.defaultMQProducer.getClientIP());
    context.setBrokerAddr(brokerAddr);
    context.setMessage(msg);
    context.setMq(mq);
    context.setNamespace(this.defaultMQProducer.getNamespace());
    // 如果是事务消息，则上下文中标记消息类型为事务半消息Trans_Msg_Half
    String isTrans =    msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
    if (isTrans != null && isTrans.equals("true")) {
        context.setMsgType(MessageType.Trans_Msg_Half);
    }

    // 如果是延时消息，则标记消息类型为延时消息Delay_Msg
    if (msg.getProperty("__STARTDELIVERTIME") != null || msg.getProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL) != null) {
        context.setMsgType(MessageType.Delay_Msg);
    }
    // 执行发送前置钩子方法
    this.executeSendMessageHookBefore(context);
}
```
如果注册了消息发送钩子函数，则执行消息发送之前的增强逻辑。通过DefaultMQProducerlmpl#registerSendMessageHook注册钩子处理类，并且可以注册多个。

### 构建发送请求包
```
/**
* 对消息发送请求头进行实例化
*/
SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
// 设置请求头参数：发送者组
requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
// 设置请求头参数：topic
requestHeader.setTopic(msg.getTopic());
// 设置默认topic，其实就是MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC=TBW102,如果开启了自动创建topic，则会创建该topic
requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
// 默认topic对应的消息队列数量
requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
// 当前要发送的消息对应的队列id
requestHeader.setQueueId(mq.getQueueId());
// 系统标识，前面逻辑计算得到
requestHeader.setSysFlag(sysFlag);
// 消息诞生时间，系统当前时间
requestHeader.setBornTimestamp(System.currentTimeMillis());
// 消息flag
requestHeader.setFlag(msg.getFlag());
requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
// 由于是发送消息，所以设置为0
requestHeader.setReconsumeTimes(0);
requestHeader.setUnitMode(this.isUnitMode());
// 是否为批量消息
requestHeader.setBatch(msg instanceof MessageBatch);
// 如果当前消息的topic以MixAll.RETRY_GROUP_TOPIC_PREFIX开头, 表明当前topic实际上是topic对应的重试topic，则执行消息重试发送相关的逻辑
if(requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
    // 获取重试次数
    // 重试次数不为null则清除重试次数
    String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
    if (reconsumeTimes != null) {
        requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
        MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
    }

    // 获取最大重试次数
    String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
    if (maxReconsumeTimes != null) {
        requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
        MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
    }
}
```
### 根据消息发送方式进行发送
```
switch (communicationMode) {
    /**
    * 异步发送方式
    */
    case ASYNC:
        // 省略代码
        .....
        break;
    /**
    * 单向发送与同步发送
    */
    case ONEWAY:
    case SYNC:
        // 省略代码
        .....
        break;
    default:
        assert false;
        break;
}
```
#### 同步发送
```
case ONEWAY:
case SYNC:
    long costTimeSync = System.currentTimeMillis() - beginStartTime;
    if (timeout < costTimeSync) {
        throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
    }
    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                brokerAddr,mq.getBrokerName(),msg,requestHeader,
                timeout - costTimeSync,communicationMode,context,this);
    break;
```
MQ客户端发送消息的入口是MQClientAPIImpl#sendMessage。请求命令是RequestCode.SEND_MESSAGE，可以找到该命令的类：org.apache.rocketmq.client.impl.MQClientAPIImpl#sendMessage
处理类是org.apache.rocketmq.broker.processor.SendMessageProcessor。

### 执行后置钩子函数
```
if (this.hasSendMessageHook()) {
    context.setSendResult(sendResult);
    this.executeSendMessageHookAfter(context);
}
```







    










