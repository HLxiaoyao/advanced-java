# RocketMQ消息消费-消息拉取
本文基于PUSH模型来介绍拉取机制，因为其内部包括了PULL模型。消息消费有两种模式：广播模式与集群模式，广播模式比较简单，每一个消费者需要去拉取订阅主题下所有消费队列的消息，接下来主要基于集群模式介绍。在集群模式下，同一个消费组内有多个消息消费者，同一个主题存在多个消费队列，那么消费者如何进行消息队列负载呢？每一个消费组内维护一个线程池来消费消息，那么这些线程又是如何分工合作的呢？
消息队列负载，通常的做法是一个消息队列在同一时间只允许被一个消息消费者消费，一个消息消费者可以同时消费多个消息队列。那么RocketMQ是如何实现的呢？

## PullMessageService实现机制
PullMessageService继承的是ServiceThread，它是一个服务线程，在构建MQClientInstance实例时被实例化，通过run方法启动，代码如下：
```
public void run() {
    log.info(this.getServiceName() + " service started");
    /**
    * 这是一种通用的设计技巧，stopped声明为volatile，每执行一次业务逻辑检查一下其运行状态，可以通过其他线程将stopped设置为true，从而停止该线程
    */
    while (!this.isStopped()) {
        try {
            // 从本地阻塞队列获取一个消息拉取任务，若pullRequestQueue为空，则线程将阻塞，直到有拉取任务被放入
            PullRequest pullRequest = this.pullRequestQueue.take();
            // 调用pullMessage进行消息拉取
            // pullRequest是何时添加的呢？
            this.pullMessage(pullRequest);
        } catch (InterruptedException ignored) {
        } catch (Exception e) {
            log.error("Pull Message Service Run Method exception", e);
        }
    }
    log.info(this.getServiceName() + " service end");
}
```
PullMessageService是消息拉取服务线程，该方法是其核心逻辑，几个核心要点如下：

    （1）while (!this.isStopped())这是一种通用的设计技巧，stopped声明为volatile，每执行一次业务逻辑检查一下其运行状态，可以通过其他线程将stopped设置为true，从而停止该线程
    （2）从本地阻塞队列pullRequestQueue获取一个消息拉取任务，若pullRequestQueue为空，则线程将阻塞，直到有拉取任务被放入
    （3）调用pullMessage进行消息拉取。
    
那么有以下问题：（1）PullRequest任务是何时添加的呢？PullMessageService提供了延迟添加与立即添加两种方式将PullRequest放入到pullRequestQueue中。代码如下：
```
public void executePullRequestLater(final PullRequest pullRequest, final long timeDelay) {
    if (!isStopped()) {
        this.scheduledExecutorService.schedule(new Runnable() {
            @Override
            public void run() {
                PullMessageService.this.executePullRequestImmediately(pullRequest);
            }
        }, timeDelay, TimeUnit.MILLISECONDS);
    } else {
        log.warn("PullMessageServiceScheduledThread has shutdown");
    }
}

    /**
     * 立即添加
     * @param pullRequest
     */
public void executePullRequestImmediately(final PullRequest pullRequest) {
    try {
        this.pullRequestQueue.put(pullRequest);
    } catch (InterruptedException e) {
        log.error("executePullRequestImmediately pullRequestQueue.put", e);
    }
}
```
那么这两种添加方式又是在哪里被调用的呢？一个是在RocketMQ根据PullRequest拉取任务执行完一次消息拉取之后，又将PullRequest对象放入到pullRequestQueue；第二个是在RebalanceImpl中创建。下面介绍一下PullRequest。
### PullRequest
```
public class PullRequest {
    // 消费者组
    private String consumerGroup;
    // 待拉取消费队列
    private MessageQueue messageQueue;
    // 消息处理队列，从Broker拉取到的消息先存入processQueue中，然后再提交到消费者消费线程池进行消费
    private ProcessQueue processQueue;
    // 待拉取的MessageQueue偏移量
    private long nextOffset;
    // 是否被锁定
    private boolean lockedFirst = false;
}
```
### ProcessQueue实现机制
ProcessQueue是MessageQueue在消费端重现、快照。PullMessageService从消息服务器默认每次拉取32条消息，按消息的队列偏移里顺序存放在ProcessQueue中，PullMessageService然后将消息提交到消费者消费线程池，消费成功后从ProcessQueue中移除。ProcessQueue核心属性如下：
```
public class ProcessQueue {
    // 存储消息用的，消息存储容器，键为消息在ConsumeQueue中的偏移量，MessageExt为消息实体
    private final TreeMap<Long, MessageExt> msgTreeMap = new TreeMap<Long, MessageExt>();
    // ProcessQueue中消息总数
    private final AtomicLong msgCount = new AtomicLong();
    private final AtomicLong msgSize = new AtomicLong();
    private final Lock lockConsume = new ReentrantLock();
    // 顺序消息处理时，消息的存储容器，键为消息在ConsumeQueue中的偏移量，MessageExt为消息实体
    // 消息消费线程从ProcessQueue的msgTreeMap中取出消息前，先将消息临时存储在consumingMsgOrderlyTreeMap中
    private final TreeMap<Long, MessageExt> consumingMsgOrderlyTreeMap = new TreeMap<Long, MessageExt>();
    private final AtomicLong tryUnlockTimes = new AtomicLong(0);
    // ProcessQueue中包含的最大队列偏移量
    private volatile long queueOffsetMax = 0L;
    // 当前ProcessQueue是否被丢弃
    private volatile boolean dropped = false;
    // 上一次开始消息拉取时间戳
    private volatile long lastPullTimestamp = System.currentTimeMillis();
    // 上一次消息消费时间戳
    private volatile long lastConsumeTimestamp = System.currentTimeMillis();
    private volatile boolean locked = false;
    private volatile long lastLockTimestamp = System.currentTimeMillis();
    private volatile boolean consuming = false;
    private volatile long msgAccCnt = 0;
}
```
### 消息拉取流程
下面以并发消息消费来分析整个消息消费流程，消息拉取可以分为以下3个步骤：

    （1）消息拉取客户端消息拉取请求封装
    （2）消息服务器查找并返回消息
    （3）消息拉取客户端处理返回的消息
    
#### 客户端封装消息拉取请求
```
// 从pullRequest中获取ProcessQueue
final ProcessQueue processQueue = pullRequest.getProcessQueue();
// 如果processQueue已经被丢弃则结束拉取流程
if (processQueue.isDropped()) {
    log.info("the pull request[{}] is dropped.", pullRequest.toString());
    return;
}
// 若ProcessQueue队列的状态未被丢弃，则更新ProcessQueue的lastPullTimestamp为当前时间
pullRequest.getProcessQueue().setLastPullTimestamp(System.currentTimeMillis());

try {
    // 判断当前消费者线程是否处于运行状态
    this.makeSureStateOK();
} catch (MQClientException e) {
    log.warn("pullMessage exception, consumer state not ok", e);
    // 若不是处于运行状态，则将拉取任务延迟1s，再次放入PullMessageService的拉取任务队列pullRequestQueue中，并结束本次消息拉取
    this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
    return;
}

// 对消费者是否挂起进行判断，如果消费者状态为已挂起，则将拉取请求延迟1s后再次放到PullMessageService的消息拉取任务队列中。
if (this.isPause()) {
    this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_SUSPEND);
    return;
}
```
从PullRequest中获取ProcessQueue，如果处理队列当前状态未被丢弃，则更新ProcessQueue的lastPullTimestamp为当前时间戳；如果当前消费者被挂起，则将拉取任务延迟1s再次放入到PullMessageService的拉取任务中，结束本次消息拉取。
```
/**
* 进行消息拉取流控。从消息消费数量与消费哦间隔两个维度进行控制
*/
// ProcessQueue队列消息总数
long cachedMessageCount = processQueue.getMsgCount().get();
// ProcessQueue队列消息大小
long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024);
// 若当前ProcessQueue处理的消息条数超过来pullThresholdForQueue，默认是1000，则将触发流控，放弃本次拉取任务。
// 并且该队列的下一次拉取任务将在50毫秒后才加入到拉取任务队列中
if (cachedMessageCount > this.defaultMQPushConsumer.getPullThresholdForQueue()) {
    this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
    // 没触发1000次流控后，打印提示语
    if ((queueFlowControlTimes++ % 1000) == 0) {
        // 省略流控日志
    }
    return;
}

// ProcessQueue中消息的总大小超过pullThresholdSizeForQueue,默认是100M，则触发流控
if (cachedMessageSizeInMiB > this.defaultMQPushConsumer.getPullThresholdSizeForQueue()) {
    this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
    if ((queueFlowControlTimes++ % 1000) == 0) {
        // 省略流控日志
    }
    return;
}

// 如果不是顺序消费，判断processQueue中队列最大偏移量与最小偏移量之间的间隔，如果大于ConsumeConcurrentlyMaxSpan（拉取偏移量阈值==2000）触发流控，
// 结束本次拉取；50毫秒之后将该拉取任务再次加入到消息拉取任务队列中。
if (!this.consumeOrderly) {
    if (processQueue.getMaxSpan() > this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
        if ((queueMaxSpanFlowControlTimes++ % 1000) == 0) {
            // 省略流控日志
        }
        return;
    }
} 
```
进行消息拉取流量控制。流控策略主要如下：

    （1）消息处理总数，如果ProcessQueue当前处理的消息超过了pullThresholdForQueue=1000将触发流量控制，放弃本次拉取任务，并且该队列的下一次拉取任务将在50毫秒后才加入到拉取任务队列中。
    （2）若不是顺序消费且ProcessQueue中消息的总大小超过pullThresholdSizeForQueue，默认是100M，则触发流控
    （3）ProcessQueue中队列最大偏移量与最小偏移量的间距，不能超过consumeConcurrencyMaxSpan，否则触发流量控制。

```
// 拉取该topic的订阅信息
final SubscriptionData subscriptionData = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
 // 若为空，结束本次消息拉取
if (null == subscriptionData) {
    // 3秒之后将该拉取任务再次加入到消息拉取任务队列中。
    this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
    log.warn("find the consumer's subscription failed, {}", pullRequest);
    return;
}
```
拉取该主题订阅信息（即从rebalanceImpl的subscriptionInner中获取），如果为空，结束本次消息拉取，然后在延迟3s后将该拉取任务再次加入到消息拉取任务队列中。
```
/**
* 以下是构建消息拉取系统标记
*/
boolean commitOffsetEnable = false;
long commitOffsetValue = 0L;
// commitOffsetEnable用来标记从内存中读取的消费进度是否大于0
if (MessageModel.CLUSTERING == this.defaultMQPushConsumer.getMessageModel()) {
    commitOffsetValue = this.offsetStore.readOffset(pullRequest.getMessageQueue(), ReadOffsetType.READ_FROM_MEMORY);
    if (commitOffsetValue > 0) {
        commitOffsetEnable = true;
    }
}

// 消息过滤机制表达式
String subExpression = null;
// 消息过滤机制是否为类过滤模式
boolean classFilter = false;
SubscriptionData sd = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
if (sd != null) {
    if (this.defaultMQPushConsumer.isPostSubscriptionWhenPull() && !sd.isClassFilterMode()) {
        subExpression = sd.getSubString();
    }
    classFilter = sd.isClassFilterMode();
}
int sysFlag = PullSysFlag.buildSysFlag(
    commitOffsetEnable, // commitOffset
    true, // suspend
    subExpression != null, // subscription
    classFilter // class filter
);
```
构建消息拉取系统标记
```
/**
* 通过pullKernelImpl()方法发起真实的消息拉取请求
*/
try {
    this.pullAPIWrapper.pullKernelImpl(
        pullRequest.getMessageQueue(),
        subExpression,
        subscriptionData.getExpressionType(),
        subscriptionData.getSubVersion(),
        pullRequest.getNextOffset(),
        this.defaultMQPushConsumer.getPullBatchSize(),
        sysFlag,
        commitOffsetValue,
        BROKER_SUSPEND_MAX_TIME_MILLIS,
        CONSUMER_TIMEOUT_MILLIS_WHEN_SUSPEND,
        CommunicationMode.ASYNC,
        pullCallback
    );
} catch (Exception e) {
    log.error("pullKernelImpl exception", e);
    this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
}
```
调用PullAPIWrapper.pullKernelImpl方法后与服务端交互。方法如下：
```
/**
* @param mq 从哪个消息消费队列拉取消息
* @param subExpression 消息过滤表达式
* @param expressionType 消息过滤表达式类型
* @param subVersion
* @param offset 消息拉取偏移量
* @param maxNums 本次拉取最大消息条数
* @param sysFlag 拉取系统标识
* @param commitOffset 当前MessageQueue的消费进度（内存中）
* @param brokerSuspendMaxTimeMillis 消息拉取过程只能够允许Broker挂起时间，默认15s
* @param timeoutMillis 消息拉取超时时间
* @param communicationMode 消息拉取模式，默认为异步拉取
* @param pullCallback 从Broker拉取到消息后的回调方法
*/
public PullResult pullKernelImpl(
    final MessageQueue mq,
    final String subExpression,
    final String expressionType,
    final long subVersion,
    final long offset,
    final int maxNums,
    final int sysFlag,
    final long commitOffset,
    final long brokerSuspendMaxTimeMillis,
    final long timeoutMillis,
    final CommunicationMode communicationMode,
    final PullCallback pullCallback
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    // 根据brokerName、BrokerId从mQClientFactory中获取broker的地址，在每次拉取消息后，会给出一个建议，下次拉取从主节点还是从节点拉取
    FindBrokerResult findBrokerResult = this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(), this.recalculatePullFromWhichNode(mq), false);
    // 若没有获取到broker地址
    if (null == findBrokerResult) {
        // 更新topic信息
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(mq.getTopic());
        // 然后再一次获取
        findBrokerResult = this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(), this.recalculatePullFromWhichNode(mq), false);
    }
    .....
}
```
根据brokerName，BrokerId从MQClientlnstance中获取Broker地址（即直接根据BrokerName从MQClientlnstance的brokerAddrTable中获取），在整个RocketMQ的Broker的部署结构中，相同名称的Broker构成主从结构，其BrokerId会不一样，在每次拉取消息后，会给出个建议，下次拉取从主节点还是从节点拉取。若没有获取到，则会调用mQClientFactory.updateTopicRouteInfoFromNameServer()方法，通过网络从broker端获取该topic最新的broker信息。
```
// PullAPIWrapper#pullKernelImpl
String brokerAddr = findBrokerResult.getBrokerAddr();
// 若消息过滤模式为类过滤，则需要根据主题、broker地址找到注册在Broker上的FilterServer地址，从FilterServer上拉取消息，否则从Broker上拉取消息
if (PullSysFlag.hasClassFilterFlag(sysFlagInner)) {
    brokerAddr = computPullFromWhichFilterServer(mq.getTopic(), brokerAddr);
}
```
如果消息过滤模式为类过滤，则需要根据主题名称、broker地址找到注册在Broker上的 FilterServer地址，从FilterServer上拉取消息，否则从Broker上拉取消息
#### broker组装并返回消息
根据消息拉取命令Code：RequestCode.PULL_MESSAGE，找到Broker端消息拉取的入口org.apache.rocketmq.broker.processor.PullMessageProcessor#processRequest。
```
# PullMessageProcessor#processReques，查找消息
final GetMessageResult getMessageResult = this.brokerController.getMessageStore().getMessage(requestHeader.getConsumerGroup(), requestHeader.getTopic(),requestHeader.getQueueId(), requestHeader.getQueueOffset(), requestHeader.getMaxMsgNums(), messageFilter);
```
调用MessageStore#getMessage()方法拉取消息，该方法的参数如下：

    group：消费者组名称
    topic：消息主题
    queueId：队列ID
    offset：待拉取偏移量，这个offset和数据的下标概念是一致的
    maxMsgNums：最大拉取消息条数
    messageFilter：消息过滤器

```
// 开始时间
long beginTime = this.getSystemClock().now();
// 拉取消息初始状态
GetMessageStatus status = GetMessageStatus.NO_MESSAGE_IN_QUEUE;
// 待查找的队列偏移量
long nextBeginOffset = offset;
// 当前消息队列最小偏移量
long minOffset = 0;
// 当前消息队列最大偏移量
long maxOffset = 0;
GetMessageResult getResult = new GetMessageResult();
// 当前commitLog文件最大偏移量
final long maxOffsetPy = this.commitLog.getMaxOffset();
// 通过topic和queueId获取对应的ConsumeQueue
ConsumeQueue consumeQueue = findConsumeQueue(topic, queueId);
```
```
public ConsumeQueue findConsumeQueue(String topic, int queueId) {
    // 每个topic对应的ConsumeQueue的Map
    ConcurrentMap<Integer, ConsumeQueue> map = consumeQueueTable.get(topic);
    if (null == map) {
        ConcurrentMap<Integer, ConsumeQueue> newMap = new ConcurrentHashMap<Integer, ConsumeQueue>(128);
        ConcurrentMap<Integer, ConsumeQueue> oldMap = consumeQueueTable.putIfAbsent(topic, newMap);
        if (oldMap != null) {
            map = oldMap;
        } else {
            map = newMap;
        }
    }

    // 从<queueId, ConsumeQueue>Map中获取对应的ConsumeQueue
    ConsumeQueue logic = map.get(queueId);

    // 若queueId对应的ConsumeQueue为null，则创建ConsumeQueue
    if (null == logic) {
        ConsumeQueue newLogic = new ConsumeQueue(topic,queueId,
StorePathConfigHelper.getStorePathConsumeQueue(this.messageStoreConfig.getStorePathRootDir()),this.getMessageStoreConfig().getMappedFileSizeConsumeQueue(),this);
        ConsumeQueue oldLogic = map.putIfAbsent(queueId, newLogic);
        if (oldLogic != null) {
            logic = oldLogic;
        } else {
            logic = newLogic;
        }
    }
    return logic;
}
```
根据主题名称与队列编号获取消息消费队列，首先通过topic从DefaultMessageStore#consumeQueueTable中获取该topic所有的消息队列信息，若没有，则创建。然后根据queueId从topic的消息队列信息中获取对应的消息队列ConsumeQueue，若没有，则创建并添加到缓存。
```
if (consumeQueue != null) {
    // 获取当前队列对应的最大偏移量和最小偏移量
    minOffset = consumeQueue.getMinOffsetInQueue();
    maxOffset = consumeQueue.getMaxOffsetInQueue();
    // 若当前队列最大偏移量为0，表明当前消费队列中没有消息，
    if (maxOffset == 0) {
        status = GetMessageStatus.NO_MESSAGE_IN_QUEUE;
        /**
        * 若当前队列是主节点或offsetCheckInSlave为false，下次拉取偏移量仍为offset
        * 若当前队列是从节点，offsetCheckInSlave为true，下次拉取偏移量为0
        */
        nextBeginOffset = nextOffsetCorrection(offset, 0);
    } else if (offset < minOffset) {
        // 若待拉取消息偏移量小于队列的起始偏移量
        status = GetMessageStatus.OFFSET_TOO_SMALL;
        /**
        * 若当前队列是主节点或offsetCheckInSlave为false，下次拉取偏移量仍为offset
        * 若当前队列是从节点，offsetCheckInSlave为true，下次拉取偏移量为minOffset
        */
        nextBeginOffset = nextOffsetCorrection(offset, minOffset);
    } else if (offset == maxOffset) {
        // 若待拉取消息偏移量等于当前队列的最大偏移量
        status = GetMessageStatus.OFFSET_OVERFLOW_ONE;
        /**
        * 下次拉取偏移量仍为offset
        */
        nextBeginOffset = nextOffsetCorrection(offset, offset);
    } else if (offset > maxOffset) {
        // 若待拉取消息偏移量大于当前队列的最大偏移量
        status = GetMessageStatus.OFFSET_OVERFLOW_BADLY;
        if (0 == minOffset) {
            nextBeginOffset = nextOffsetCorrection(offset, minOffset);
        } else {
            nextBeginOffset = nextOffsetCorrection(offset, maxOffset);
        }
    }
    ....
}
```
消息偏移量异常情况校对下一次拉取偏移量

    （1）maxOffset == 0，表示当前消费队列中没有消息，拉取结果为NO_MESSAGE_IN_QUEUE。
        若是主节点或offsetCheckInSlave为false，下次拉取偏移量仍为offset
        若是从节点或offsetCheckSlave为true，下次拉取偏移量为0
    （2）offset < minOffset，表示要拉取的偏移量小于队列最小的偏移量
        若是主节点或offsetCheckInSlave为false，设置下一次拉取的偏移量为offset
        若是从节点或offsetCheckInSlave为true，下次拉取偏移量为minOffset
    （3）offset == maxOffset，表示待拉取偏移量等于队列最大偏移量，下次拉取结果仍为offset
    （4）offset > maxOffset，表示偏移量越界返回状态：OFFSET_OVERFLOW_BADLY
        若minOffset == 0，若是主节点或offsetCheckInSlave为false，下次拉取偏移量仍为offset，若是从节点或offsetCheckSlave为true，下次拉取偏移量为minOffset
        否则，若是主节点或offsetCheckInSlave为false，下次拉取偏移量仍为offset，若是从节点或offsetCheckSlave为true，下次拉取偏移量为maxOffset

若offset大于minOffset并小于maxOffset，正常情况。则从offset处尝试拉取32条消息。则根据消息队列偏移量从commitlog文件中查找消息。代码如下：
```
// 首先通过offset定位到consumeQueue，然后返回该consumeQueue从当前offset到该consumeQueue中最大可读消息内存
SelectMappedBufferResult bufferConsumeQueue = consumeQueue.getIndexBuffer(offset);
if (bufferConsumeQueue != null) {
    try {
        status = GetMessageStatus.NO_MATCHED_MESSAGE;
        long nextPhyFileStartOffset = Long.MIN_VALUE;
        long maxPhyOffsetPulling = 0;
        int i = 0;
        // 最大过滤消息字节数
        final int maxFilterMessageCount = Math.max(16000, maxMsgNums * ConsumeQueue.CQ_STORE_UNIT_SIZE);
        final boolean diskFallRecorded = this.messageStoreConfig.isDiskFallRecorded();
        ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
        // 循环读取bufferConsumeQueue的数据
        for (; i < bufferConsumeQueue.getSize() && i < maxFilterMessageCount; i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
            // 在consumeQueue中解析一条消息，consumeQueue中信息存储到格式就是以下这个顺序。
            // commitlog偏移量
            long offsetPy = bufferConsumeQueue.getByteBuffer().getLong();
            // 消息总长度
            int sizePy = bufferConsumeQueue.getByteBuffer().getInt();
            // 消息tag hashcode
            long tagsCode = bufferConsumeQueue.getByteBuffer().getLong();
            // 当前拉取的物理偏移量
            maxPhyOffsetPulling = offsetPy;
            // 说明物理文件正在被删除
            if (nextPhyFileStartOffset != Long.MIN_VALUE) {
                // 如果拉取到的消息偏移量小于下一个要拉取的物理偏移量的话，直接跳过该条消息
                if (offsetPy < nextPhyFileStartOffset)
                    continue;
            }
            // 检查该offsetPy，拉取的偏移量是否在磁盘上
            // 消息存储在物理内存中占用一个最大比例，若大于这个比例，有一部分消息就会置换到物理磁盘上
            boolean isInDisk = checkInDiskByCommitOffset(offsetPy, maxOffsetPy);
            // 判断本次拉取任务是否完成，sizePy：当前消息的字节长度，maxMsgNums:本次拉取消息条数，bufferTotal:已拉取消息字节总长度，不包含当前消息，messageTotal：已拉取消息总条数，isInDisk当前消息是否存在于磁盘中
            if (this.isTheBatchFull(sizePy, maxMsgNums, getResult.getBufferTotalSize(), getResult.getMessageCount(),isInDisk)) {
                break;
            }
            // 执行消息过滤，如果符合过滤条件。则直接进行下一条的拉取，如果不符合过滤条件，则进入继续执行，并如果最终符合条件，则将该消息添加到拉取结果中
            boolean extRet = false, isTagsCodeLegal = true;
            if (consumeQueue.isExtAddr(tagsCode)) {
                extRet = consumeQueue.getExt(tagsCode, cqExtUnit);
                if (extRet) {
                    tagsCode = cqExtUnit.getTagsCode();
                } else {
                    isTagsCodeLegal = false;
                }
            }
            if (messageFilter != null && !messageFilter.isMatchedByConsumeQueue(isTagsCodeLegal ? tagsCode : null, extRet ? cqExtUnit : null)) {
                if (getResult.getBufferTotalSize() == 0) {
                    status = GetMessageStatus.NO_MATCHED_MESSAGE;
                }
                continue;
            }
            // 从commitlog文件中读取消息，根据偏移量与消息大小
            SelectMappedBufferResult selectResult = this.commitLog.getMessage(offsetPy, sizePy);
            if (null == selectResult) {
                if (getResult.getBufferTotalSize() == 0) {
                    status = GetMessageStatus.MESSAGE_WAS_REMOVING;
                }
                // 如果该偏移量没有找到正确的消息，则说明已经到文件末尾了，下一次切换到下一个 commitlog 文件读取
                nextPhyFileStartOffset = this.commitLog.rollNextFile(offsetPy);
                continue;
            }
            // 从commitlog（全量消息）再次过滤，consumeque 中只能处理 TAG 模式的过滤，SQL92 这种模式无法过滤，因为SQL92 需要依赖消息中的属性，故在这里再做一次过滤。如果消息符合条件，则加入到拉取结果中
            if (messageFilter != null && !messageFilter.isMatchedByCommitLog(selectResult.getByteBuffer().slice(), null)) {
                if (getResult.getBufferTotalSize() == 0) {
                    status = GetMessageStatus.NO_MATCHED_MESSAGE;
                }
                // release...
                selectResult.release();
                continue;
            }
            this.storeStatsService.getGetMessageTransferedMsgCount().incrementAndGet();
            // 将消息加入到拉取结果中
            getResult.addMessage(selectResult);
            status = GetMessageStatus.FOUND;
            nextPhyFileStartOffset = Long.MIN_VALUE;
        }
        if (diskFallRecorded) {
            long fallBehind = maxOffsetPy - maxPhyOffsetPulling;
            brokerStatsManager.recordDiskFallBehindSize(group, topic, queueId, fallBehind);
        }
        // 设置下一次拉取偏移量，然后返回拉取结果
        nextBeginOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
        long diff = maxOffsetPy - maxPhyOffsetPulling;
        long memory = (long) (StoreUtil.TOTAL_PHYSICAL_MEMORY_SIZE
        * (this.messageStoreConfig.getAccessMessageInMemoryMaxRatio() / 100.0));
        getResult.setSuggestPullingFromSlave(diff > memory);
    } finally {
        bufferConsumeQueue.release();
    }
} else {
    status = GetMessageStatus.OFFSET_FOUND_NULL;
    nextBeginOffset = nextOffsetCorrection(offset, consumeQueue.rollNextFile(offset));
}
```
具体处理逻辑：
如果bufferTotal和messageTotal都等于0，显然本次拉取任务才刚开始，本批拉取任务未完成，返回false。
如果maxMsgNums <= messageTotal，返回true，表示已拉取完毕。
若上述两种情况都不满足，接下来根据是否在磁盘中，会区分对待：

    （1）如果该消息存在于磁盘而不是内存中，如果已拉取消息字节数 + 待拉取消息的长度 > maxTransferBytesOnMessageInDisk(MessageStoreConfig)，默认64K，则不继续拉取该消息，返回拉取任务结束。
        如果已拉取消息条数 > maxTransferCountOnMessageInDisk (MessageStoreConfig)默认为8，也就是，如果消息存在于磁盘中，一次拉取任务最多拉取8条。
    （2）如果该消息存在于内存中，对应的参数为maxTransferBytesOnMessageInMemory、maxTransferCountOnMessageInMemory，其逻辑与上述一样。
    
```
// PullMessageProcessor#processRequest
response.setRemark(getMessageResult.getStatus().name());      responseHeader.setNextBeginOffset(getMessageResult.getNextBeginOffset());
responseHeader.setMinOffset(getMessageResult.getMinOffset());
responseHeader.setMaxOffset(getMessageResult.getMaxOffset());
// 根据主从同步延迟，若从节点数据包含下一次拉取偏移量，设置下一次拉取任务的brokerId
if (getMessageResult.isSuggestPullingFromSlave()) { 
    responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
} else {
    responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
}
```
根据主从同步延迟，若从节点数据包含下一次拉取到偏移量，设置下一次拉取任务的brokerId。

#### 消息拉取客户端处理消息
客户端网络交互端收到服务端broker上返回的数据后，NettyRemotingClient在收到服务端响应结构后会调用PullCallback的onSuccess或onException，PullCallBack对象在DefaultMQPushConsumerImpl#pullMessage中创建。下面首先分析拉取结果为PullStatus.FOUND（找到对应的消息）的情况。
```
# PullCallBack#onSuccess()
// 更新PullRequest的下一次拉取偏移量
long prevRequestOffset = pullRequest.getNextOffset();
pullRequest.setNextOffset(pullResult.getNextBeginOffset());
// 消息拉取的耗时
long pullRT = System.currentTimeMillis() - beginTimestamp;
DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),pullRequest.getMessageQueue().getTopic(), pullRT);
long firstMsgOffset = Long.MAX_VALUE;
// 若msgFoundList为空，则立即将PullRequest放入到PullMessageService的pullRequestQueue中，以便PullMessageService能及时唤醒并再次执行消息拉取
// 为什么msgFoundList会为空呢？因为在RocketMQ根据TAG消息过滤，在服务端只验证来TAG的hashcode，在客户端再次对消息进行过滤时，故可能出现msgFoundList为空的情况。
if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
}
``` 
更新PullRequest的下一次拉取偏移量（broker端返回），如果msgFoundList为空，则立即将PullRequest放入到PullMessageService的pullRequestQueue（这时候的PullRequest是更新了待拉取偏移量的），以便PullMessageService能及时唤醒并再次执行消息拉取。为什么msgFoundList会为空呢？因为在RocketMQ根据TAG消息过滤，在服务端只验证来TAG的hashcode，在客户端再次对消息进行过滤时，故可能出现msgFoundList为空的情况。
```
// 将拉取到的消息放入ProcessQueue
boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
// 然后将拉取到的消息提交到consumeMessageService中供消费者进行消费，该方法是一个异步方法，提交后会立即返回
DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(pullResult.getMsgFoundList(),processQueue,pullRequest.getMessageQueue(),dispatchToConsume);
```
将拉取到的消息存放到ProcessQueue，然后将拉取到的消息提交到ConsumeMessageService中供消费者消费，该方法是一个异步方法，提交后会立即返回。
```
// 若pullInterval>0，则等待pullInterval毫秒后将PullRequest对象放入到PullMessageService的pullRequestQueue中，
// 该消息队列到下次拉取即将被激活达到持续消息拉取，实现实时拉取消息的效果
if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
    DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
} else {
    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
}
```
将消费提交给消费者线程之后PullCallBack将立即返回，可以说本次消息拉取顺利完成，然后根据pullInterval参数，如果pullInterval > 0，则等待pullInterval毫秒后将PullRequest对象放入到PullMessageService的pullRequestQueue中，该消息队列的下次拉取即将被激活，达到持续消息拉取，实现准实时拉取消息的效果。下面分析消息拉取异常的处理，如何校对拉取偏移量。
```
case NO_NEW_MSG:
    pullRequest.setNextOffset(pullResult.getNextBeginOffset());
    DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
    break;
```
如果返回NO_NEW_MSG没有新消息，NO_MATCHED_MSG（没有匹配消息），则直接使用服务器端校正的偏移量进行下一次消息的拉取。再来看看服务端是如何校正Offset

    （1）NO_NEW_MSG对应GetMessageResult.OFFSET_FOUND_NULL，GetMessageResult.OFFSET_OVERFLOW_ONE
        OFFSET_OVERFLOW_ONE：待拉取offset等于消息队列最大的偏移量，如果有新的消息到达，此时会创建一个新的ConsumeQueue文件，按照上一个ConsumeQueue的最大偏移量就是下一个文件的起始偏移，所以如果按照该offset第二次拉取消息时能成功
        OFFSET _FOUND _NULL：根据ConsumeQueue的偏移量没有找到内容，将偏移定位到下一个ConsumeQueue，其实就是offset ＋（一个ConsumeQueue含多少个条目= MappedFileSize/20)

```
// 若是偏移量非法，首先将processQueue的dropped为true，表示丢弃该队列，意味着processQueue中拉取的消息将停止消费，
// 然后根据服务端下一次校对的偏移量尝试更新消息消费进度（内存中），然后尝试持久化消息消费进度，并将该消息队列从RebalacnImpl的处理队列中移除，意味着
// 暂停该消息队列的消息拉取，等待下一次消息队列重新负载。
case OFFSET_ILLEGAL:
    log.warn("the pull request offset illegal, {} {}",
    pullRequest.toString(), pullResult.toString());
    pullRequest.setNextOffset(pullResult.getNextBeginOffset());
    pullRequest.getProcessQueue().setDropped(true);
    DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {
        @Override
        public void run() {
            try {
                DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(),pullRequest.getNextOffset(), false);
                DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());
                DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());
                log.warn("fix the pull request offset, {}", pullRequest);
            } catch (Throwable e) {
                log.error("executeTaskLater Exception", e);
            }
        }
    }, 10000);
break;
```
如果拉取结果显示偏移量非法，首先将ProcessQueue的dropped为ture表示丢弃该消费队列， ProcessQueue中拉取的消息将停止消费，然后根据服务端下一次校对的偏移量尝试更新消息消费进度（内存中），然后尝试持久化消息消费进度，并将该消息队列从Rebalacnlmpl的处理队列中移除，意味着暂停该消息队列的消息拉取，等待下一次消息队列重新负载。

![](/images/RocketMQ/消息拉取.png)

    





