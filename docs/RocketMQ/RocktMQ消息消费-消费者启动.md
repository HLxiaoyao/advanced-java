# RocktMQ消息消费-消费者启动
消息消费以组的模式开展，一个消费组内可以包含多个消费者，每一个消费组可订阅多个主题，消费组之间有集群模式与广播模式两种消费模式。集群模式，主题下的同一条消息只允许被其中一个消费者消费。广播模式，主题下的同一条消息将被集群内的所有消费者消费一次。消息服务器与消费者之间的消息传送也有两种方式：推模式、拉模式。所谓的拉模式，是消费端主动发起拉消息请求，而推模式是消息到达消息服务器后，推送给消息消费者。RocketMQ消息推模式的实现基于拉模式，在拉模式上包装一层，一个拉取任务完成后开始下一个拉取任务。

集群模式下，多个消费者如何对消息队列进行负载呢？消息队列负载机制遵循一个通用的思想：一个消息队列同一时间只允许被一个消费者消费，一个消费者可以消费多个消息队列。

RocketMQ支持局部顺序消息消费，也就是保证同一个消息队列上的消息顺序消费。不支持消息全局顺序消费，如果要实现某一主题的全局顺序消息消费，可以将该主题的队列数设置为1，牺牲高可用性。
RocketMQ支持两种消息过滤模式：表达式(TAG、SQL92)与类过滤模式。

## 消息消费者概述
消息消费分为推和拉两种模式，下面以推模式的消费者MQPushConsumer做介绍。其实现类DefaultMQPushConsumer的核心属性如下：
```
public class DefaultMQPushConsumer extends ClientConfig implements MQPushConsumer {
    // 消费者所属组
    private String consumerGroup;
    // 消息消费模式，集群模式和广播模式，默认是集群模式
    private MessageModel messageModel = MessageModel.CLUSTERING;
    // 根据消息进度从消息服务器拉取不到消息时重新计算消费策略
    // 若从消息进度服务OffsetStore读取到MessageQueue中的偏移量不小于0，则使用读取到的偏移量，只有在读到的偏移量小于0时，上述策略才会生效。
    private ConsumeFromWhere consumeFromWhere = ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET;
    // 集群模式下消息队列负载策略
    private AllocateMessageQueueStrategy allocateMessageQueueStrategy;
    // 订阅信息
    private Map<String /* topic */, String /* sub expression */> subscription = new HashMap<String, String>();
     // 消息业务监听器
    private MessageListener messageListener;
    // 消息消费进度存储器
    private OffsetStore offsetStore;
    // 消费者最小线程数
    private int consumeThreadMin = 20;
    // 消费者最大线程数，由于消费者线程池使用无界队列，故消费者线程个数其实最多只有consumeThreadMin个
    private int consumeThreadMax = 20;
    // 并发消费消息时处理队列最大跨度，默认2000，表示如果消息处理队列中偏移量最大的消息与偏移量最小的消息的跨度超过2000，则延迟50毫秒再拉取消息
    private int consumeConcurrentlyMaxSpan = 2000;
    // 每隔1000次流控后打印流控日志
    private int pullThresholdForQueue = 1000;
    // 推模式下拉取任务时间间隔，默认一次拉取任务完成继续拉取
    private long pullInterval = 0;
    // 消息并发消费时一次消费消息条数，
    private int consumeMessageBatchMaxSize = 1;
    // 每次消息拉取所拉取的条数，默认32条
    private int pullBatchSize = 32;
    // 每次拉取消息是否更新订阅信息，默认为false
    private boolean postSubscriptionWhenPull = false;
    // 最大消费重试次数。若消息消费次数超过最大重试次数，则消息会被转移到一个失败队列，等待被删除
    private int maxReconsumeTimes = -1;
    // 延迟将该队列的消息提交到消费者线程的等待时间
    private long suspendCurrentQueueTimeMillis = 1000;
    // 消息消费超时时间，默认15分钟
    private long consumeTimeout = 15;
}
```
## DefaultMQPushConsumer使用（推模式）
```
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("CID_JODIE_1");
// 订阅topic
consumer.subscribe("TopicTest111", "*");
// 从头开始消费
consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
// 消费者名称
consumer.setInstanceName("consumer1");
// namesrv地址
consumer.setNamesrvAddr("127.0.0.1:9876");
// 注册监听器
consumer.registerMessageListener(new MessageListenerConcurrently() {
    
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
consumer.start();
```
## 初始化DefaultMQPushConsumer
DefaultMQPushConsumer的构造方法如下：
```
public DefaultMQPushConsumer(final String consumerGroup) {
    this(null, consumerGroup, null, new AllocateMessageQueueAveragely());
}
```
调用了它的同名构造方法，采用AllocateMessageQueueAveragely策略（平均散列队列算法）
```
public DefaultMQPushConsumer(final String namespace, final String consumerGroup, RPCHook rpcHook,
    AllocateMessageQueueStrategy allocateMessageQueueStrategy) {
    this.consumerGroup = consumerGroup;
    this.namespace = namespace;
    this.allocateMessageQueueStrategy = allocateMessageQueueStrategy;
    defaultMQPushConsumerImpl = new DefaultMQPushConsumerImpl(this, rpcHook);
}
```
可以看到实际初始化是通过DefaultMQPushConsumerImpl实现的，DefaultMQPushConsumer持有一个defaultMQPushConsumerImpl的引用。
```
// DefaultMQPushConsumerImpl.java
public DefaultMQPushConsumerImpl(DefaultMQPushConsumer defaultMQPushConsumer, RPCHook rpcHook) {
    // 初始化DefaultMQPushConsumerImpl，将defaultMQPushConsumer的实际引用传入
    this.defaultMQPushConsumer = defaultMQPushConsumer;
    // 传入rpcHook并指向本类的引用
    this.rpcHook = rpcHook;
}
```
### 注册消费监听MessageListener
消费监听接口MessageListener有两个具体的实现，分别为并行消费监听MessageListenerConcurrently与顺序消费监听MessageListenerOrderly，MessageListenerConcurrently的注册过程如下：
```
@Override
public void registerMessageListener(
            MessageListenerConcurrently messageListener) {
    // 将实现指向本类引用
    this.messageListener = messageListener;
    // 进行真实注册
    this.defaultMQPushConsumerImpl.registerMessageListener(messageListener);
}
```
接着看defaultMQPushConsumerImpl.registerMessageListener
```
// DefaultMQPushConsumerImpl.java
public void registerMessageListener(MessageListener messageListener) {
    this.messageListenerInner = messageListener;
}
```
### 订阅topic
topic订阅主要通过方法subscribe实现，首先看一下DefaultMQPushConsumer的subscribe实现
```
@Override
public void subscribe(String topic, String subExpression) throws MQClientException {
    this.defaultMQPushConsumerImpl.subscribe(withNamespace(topic), subExpression);
}
```
可以看到是调用了DefaultMQPushConsumerImpl的subscribe方法
```
public void subscribe(String topic, String subExpression) throws MQClientException {
    try {
        // 构建主题的订阅数据，默认为集群消费
        SubscriptionData subscriptionData = FilterAPI.buildSubscriptionData(this.defaultMQPushConsumer.getConsumerGroup(),
            topic, subExpression);
        // 将topic的订阅数据进行保存
        this.rebalanceImpl.getSubscriptionInner().put(topic, subscriptionData);
        if (this.mQClientFactory != null) {
            // 如果MQClientInstance不为空，则向所有的broker发送心跳包，加锁
            this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
        }
    } catch (Exception e) {
        throw new MQClientException("subscription exception", e);
    }
}
```
```
// FilterAPI.java
public static SubscriptionData buildSubscriptionData(final String consumerGroup, String topic,
    String subString) throws Exception {
    // 构造一个SubscriptionData实体，设置topic、表达式（tag）
    SubscriptionData subscriptionData = new SubscriptionData();
    subscriptionData.setTopic(topic);
    subscriptionData.setSubString(subString);

    // 如果tag为空或者为"*"，统一设置为"*"，即订阅所有消息
    if (null == subString || subString.equals(SubscriptionData.SUB_ALL) || subString.length() == 0) {
        subscriptionData.setSubString(SubscriptionData.SUB_ALL);
    } else {
        // tag不为空，则先按照‘|’进行分割
        String[] tags = subString.split("\\|\\|");
        if (tags.length > 0) {
            // 遍历tag表达式数组
            for (String tag : tags) {
                if (tag.length() > 0) {
                    String trimString = tag.trim();
                    if (trimString.length() > 0) {
                        // 将每个tag的值设置到tagSet中
                        subscriptionData.getTagsSet().add(trimString);
                        subscriptionData.getCodeSet().add(trimString.hashCode());
                    }
                }
            }
        } else {
            // tag解析异常
            throw new Exception("subString split error");
        }
    }
    return subscriptionData;
}
```
## 消费者启动流程
消费者的启动是由DefaultMQPushConsumerImpl#start()方法实现的。下面进行拆分讲解。
### 构建topic订阅信息
```
switch (this.serviceState) {
    case CREATE_JUST:
        this.serviceState = ServiceState.START_FAILED;
        // 基本的参数校验，group name不能是DEFAULT_CONSUMER
        this.checkConfig();

        //2、将DefaultMQPushConsumer的订阅信息copy到RebalanceService中
        //如果是cluster模式，如果订阅了topic,则自动订阅%RETRY%topic
        this.copySubscription();
        // 省略部分代码 ....
}
```
首先进行基本的参数校验，然后构建topic订阅信息SubscriptionData，构建代码如下：
```
private void copySubscription() throws MQClientException {
    try {
        // 取出订阅信息
        Map<String, String> sub = this.defaultMQPushConsumer.getSubscription();
        if (sub != null) {
            for (final Map.Entry<String, String> entry : sub.entrySet()) {
                final String topic = entry.getKey();
                final String subString = entry.getValue();
                // 构造订阅关系subscriptionData
                SubscriptionData subscriptionData = FilterAPI.buildSubscriptionData(this.defaultMQPushConsumer.getConsumerGroup(),topic, subString);
                // 加入到rebalanceImpl的subscriptionInner中，subscriptionInner是一个ConcurrentMapHashMap
                // 就是订阅关系表，以topic为key
                this.rebalanceImpl.getSubscriptionInner().put(topic, subscriptionData);
            }
        }

        // 注册监听器
        if (null == this.messageListenerInner) {
            this.messageListenerInner = this.defaultMQPushConsumer.getMessageListener();
        }

        switch (this.defaultMQPushConsumer.getMessageModel()) {
            // 如果是广播模式，不需要重试
            case BROADCASTING:
                break;
            // 如果是集群模式，则将重试topic也加入到subscriptionInner中，
            // 消息重试是以消费组为单位，topic为%RETRY%+消费组名
            case CLUSTERING:
                final String retryTopic = MixAll.getRetryTopic(this.defaultMQPushConsumer.getConsumerGroup());
                SubscriptionData subscriptionData = FilterAPI.buildSubscriptionData(this.defaultMQPushConsumer.getConsumerGroup(),retryTopic, SubscriptionData.SUB_ALL);
                this.rebalanceImpl.getSubscriptionInner().put(retryTopic, subscriptionData);
                break;
            default:
                break;
        }
    } catch (Exception e) {
        throw new MQClientException("subscription exception", e);
    }
}
```
构建主题订阅信息SubscriptionData并加入到RebalanceImpl的订阅信息中。订阅关系来源主要有两个：
**（1）通过调用DefaultMQPushConsumerImpl#subscribe(String topic, String subExpression)方法来添加订阅关系，这是在构建消费者实例之后，实例启动之前，设置的消费者实例属性（调用）：**
```
public void subscribe(String topic, String subExpression) throws MQClientException {
    try {
        // 构建SubscriptionData，主要是解析subExpression，并设置到SubscriptionData实例的tagsSet中
        SubscriptionData subscriptionData = FilterAPI.buildSubscriptionData(this.defaultMQPushConsumer.getConsumerGroup(),topic, subExpression);
        // SubscriptionData并加入到RebalanceImpl的订阅信息
        this.rebalanceImpl.getSubscriptionInner().put(topic, subscriptionData);
        // mQClientFactory是客户端向服务器发起连接的连接器，
        if (this.mQClientFactory != null) {
            // 发送心跳包，这里只发一次
            this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
        }
    } catch (Exception e) {
        throw new MQClientException("subscription exception", e);
    }
}
```
（2）订阅重试主题信息。从这里可以看出，RocketMQ消息重试是以消费者组为单位，而不是主题，消息重试主题名为%RETRY%+消费组名。消费者在启动的时候会自动订阅该主题，参与该主题的消息队列负载。
### 初始化MQClientInstance、RebalanceImpl等
```
//3、修改InstanceName参数值为PID
if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {
    this.defaultMQPushConsumer.changeInstanceNameToPID();
}
//4、新建一个MQClientInstance,客户端管理类，所有的i/o类操作由它管理，缓存客户端和topic信息，各种service一个进程只有一个实例
this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);
// 客户端负载均衡
this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
//5、Queue分配策略，默认AVG
// 对于同一个group内的consumer，RebalanceImpl负责分配具体每个consumer应该消费哪些queue上的消息,以达到负载均衡的目的。
// Rebalance支持多种分配策略，比如平均分配、一致性Hash等(具体参考AllocateMessageQueueStrategy实现类)。默认采用平均分配策略(AVG)。
this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);
//6、PullRequest封装实现类，封装了和broker的通信接口
this.pullAPIWrapper = new PullAPIWrapper(mQClientFactory,this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
//7、消息被客户端过滤时会回调hook
this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);
```
初始化MQClientInstance、RebalanceImpl实例，RebalanceImpl中默认采用平均分配策略进行负载。然后初始化PullAPIWrapper实例，该实例是PullRequest封装实现类，封装了和broker的通信接口。
### 初始化消费进度
```
//8、consumer客户端消费offset持久化接口(存储器)
if (this.defaultMQPushConsumer.getOffsetStore() != null) {
    this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
} else {
    switch (this.defaultMQPushConsumer.getMessageModel()) {
        //广播消息本地持久化offset
        case BROADCASTING:
            this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
            break;
        //集群模式持久化到broker
        case CLUSTERING:
            this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
            break;
        default:
            break;
    }
    this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
}
//9、如果是本地持久化会从文件中load
this.offsetStore.load();
```
初始化消息进度。若消息消费是集群模式，那么消息进度保存在Broker上；若是广播模式，那么消息消费进度保存在消费端。
### 创建消费线程
```
// 根据不同的消费模式，初始化具体的消费service
//10、消费服务，顺序和并发消息逻辑不同,接收消息并调用listener消费，处理消费结果
if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
    this.consumeOrderly = true;
    this.consumeMessageService = new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
} else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
    this.consumeOrderly = false;
    this.consumeMessageService = new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
}
// 启动consumeMessageService消费线程服务，--定时清理过期消息
this.consumeMessageService.start();
```
根据是否是顺序消费，创建消费端消费线程服务。consumeMessageService主要负责消息消费，内部维护一个线程池。
### 启动MQClientInstance
```
// 向broker注册consumer
boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
if (!registerOK) {
    this.serviceState = ServiceState.CREATE_JUST;
    this.consumeMessageService.shutdown();          
}
//13、启动MQClientInstance，会启动PullMessageService和RebalanceService
mQClientFactory.start();
```
向MQClientInstance注册消费者，并启动MQClientInstance，在一个JVM中的所有消费者、生产者持有同一个MQClientInstance，MQClientInstance只会启动一次。

![](/images/RocketMQ/consumer初始化.png)


