# RocketMQ消息发送总结
## Producer如何获取topic路由信息？如何自动创建topic？

![](/images/RocketMQ/Producer缓存的路由信息.jpg)

producer侧的路由同步有两种途径，途径一是在消息发送过程去同步获取路由信息；途径二是通过定时任务（每30s）同步获取路由信息。

    （1）第一次发送消息时，本地没有缓存topic的路由信息，查询NameServer尝试获取路由信息。
    （2）若路由信息从NameServer没有查询到，那么会默认拉取broker启动时默认创建好名为“TBW102”的Topic。在broker启动时，会调用TopicConfigManager的构造方法，autoCreateTopicEnable打开后，会将“TBW102”保存到topicConfigTable中。
        但是若autoCreateTopicEnable没有打开，从NameServer没有查询到路由信息时将抛出无法找到topic路由异常。
    （3）将“TBW102”topic路由信息构建TopicPublishInfo，并将用topic为key，TopicPublishInfo为value更新本地缓存。
    （4）然后将消息发送到“TBW102”topic，Broker在接受到消息之后，若该Broker中没有该消息原本到topic，那么就会直接创建。
    
    
    （1）在发送完第一次消息之后，再发送消息时，首先从本地缓存获取topic的路由信息。
    （2）若本地缓存没有该topic的路由信息，则尝试从NameServer获取路由信息。
    （3）若从NameServer获取到路由信息，并且与本地路由信息不一致（定时任务获取时会出现这种情况），更新本地缓存的路由信息。
        其实就是重新创建一个TopicPublishInfo对象，然后构建该对象的信息，主要是对应topic的队列信息。只有topic的队列在master上，并且可写，才会在producer端创建队列信息。
   
### TopicPublishInfo举例 
```  
 {
	"TBW102": [{
		"brokerName": "broker-a",
		"perm": 7,
		"readQueueNums": 8,
		"topicSynFlag": 0,
		"writeQueueNums": 8
	}, {
		"brokerName": "broker-b",
		"perm": 7,
		"readQueueNums": 8,
		"topicSynFlag": 0,
		"writeQueueNums": 8
	}]
}
```
topic（名为TBW102）在broker-a和broker-b上存在队列信息。

    （1）首先按照broker-a、broker-b的顺序针对broker信息进行排序。
    （2）针对broker-a会生成8个MessageQueue对象，MessageQueue的topic为TBW102，brokerName为broker-a，queueId分别是0-7。
    （3）针对broker-b会生成8个MessageQueue对象，MessageQueue的topic为TBW102，brokerName为broker-b，queueId分别是0-7。
    （4）topic（名为TBW102）的TopicPublishInfo整体包含16个MessageQueue对象，其中有8个broker-a的MessageQueue，有8个broker-b的MessageQueue。
MessageQueue是以topic+brokerName+queueId作为维度的Queue对象。

## 当集群新增master节点后，如何保证队列负载？
通过源码可知，RocketMQ目前只能是通过手动配置topic到新增的master节点。但是如果集群中有成百上千个topic呢？全部手动配置那么工作量将很大，并且流量突然增大时，再手动扩容topic时效性也差。那如何来解决这个问题呢？

可以按业务分集群，把topic归类到不同的集群中，这样每个集群添加broker后，需要重新分配的topic就大大减少了。

更好的解决方案是添加一个复制功能，新增的broker自动从nameserver拉取需要复制到新broker的topic配置。期待以后的版本迭代中如愿增加这个功能吧。

## 选择消息队列？消息发送异常机制  
如果找到路由信息会根据它选择消息队列，返回的消息队列会按照broker、序号排序。例如topicA在broker-a，broker-b上分别创建了4个队列，那么返回的消息队列：［｛“ brokerName”:”broker-a”,”queueId”:0}, {“brokerName”:”broker-a”,”queueId”:1},{“brokerName”: ”broker-a”, “queueId”:2},｛“BrokerName”：“broker-a”，”queueId” ：3，
{“brokerName”：”broker-b”,”queueId”:0}, {“brokerName”：”broker-b”，“queueId”：1}, {“brokerName”:”broker-b”,”queueId”:2}, {“brokerName”:"broker-b”，”queueId” ：3 }]，那RocketMQ如何选择消息队列呢？

首先消息发送端采用重试机制，由retryTimesWhenSendFailed参数指定同步方式重试次数，异步重试机制在收到消息发送结构后执行回调之前进行重试。由retryTimesWhenSendAsyncFailed指定，接下来就是循环执行，选择消息队列、发送消息，发送成功则返回，收到异常则重试。不管是第一次发送，还是重试发送，选择消息队列都是selectOneMessageQueue()方法一个入口，但是该方法内存在两种选择消息队列的方式。

    （1）sendLatencyFaultEnable=false，默认不启用Broker故障延迟机制。
    （2）sendLatencyFaultEnable=true，启用Broker故障延迟机制。

### 默认机制（不启用故障延迟）
首先在一次消息发送过程中，可能会多次执行选择消息队列这个方法，lastBrokerName就是上一次选择的执行发送消息失败的Broker。第一次执行消息队列选择时，lastBrokerName为null，此时直接用sendWhichQueue自增再获取值，与当前路由表中消息队列个数取模，再根据取模结果，取出MessageQueue列表中的某个Queue。

若第一次发送消息失败，还是按上述方式采用sendWhichQueue自增再获取值，与当前路由表中消息队列个数取模，再根据取模结果，取出MessageQueue列表中的某个Queue。但是这个时候需要判断本次获取的Queue所属Broker是否是lastBrokerName，若是，则进行规避；然后再进行自增取模、规避的流程，直到取到的Queue所属Broker不是lastBrokerName。若一直没有满足条件的Queue，则按照第一次发送时的方式获取Queue。

该算法在一次消息发送过程中能成功规避故障的Broker，但如果Broker若机，由于路由算法中的消息队列是按Broker排序的，如果上一次根据路由算法选择的是宕机的Broker的第一个队列，那么随后的下次选择的是宕机Broker的第二个队列，消息发送很有可能会失败，再次引发重试，带来不必要的性能损耗。那么有什么方法在一次消息发送失败后，暂时将该Broker排除在消息队列选择范围外呢?
还有，Broker不可用后路由信息中为什么还会包含该Broker的路由信息呢？这是因为，首先 NameServer检测Broker是否可用是有延迟的，最短为一次心跳检测间隔(1Os)；其次，NameServer不会检测到Broker宕机后马上推送消息给消息生产者，而是消息生产者每隔30s更新一次路由信息，所以消息生产者最快感知Broker最新的路由信息也需要30s。如果能引人一种机制，在Broker宕机期间，如果一次消息发送失败后，可以将该Broker暂时排除在消息队列的选择范围中。

### Broker故障延迟机制
Broker故障延迟机制主要需要以下信息，FaultItem：失败条目（规避规则条目），LatencyFaultToleranceImpl内部类，该类的属性如下：
```
private final String name; // 条目唯一键，这里是brokerName
private volatile long currentLatency; // 本次消息发送延迟
private volatile long startTimestamp; // 故障规避开始时间
```
MQFaultStrategy：消息失败策略，延迟实现的门面类，该类下的几个重要属性：
```
// 根据currentLatency为发送消息的执行时间，从latencyMax尾部向前找到第一个比currentLatency小的值的索引index，如果没有找到，则返回0。然后根据这个索引从notAvailableDuration数组中取出对应的时间，这个时长内，Broker将设置为不可用。
private long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L};
private long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L};
```
broker容错机制的思路及实现步骤如下：

    （1）在消息发送完成之后（不论失败还是成功），mq根据消息发送耗时来预测该broker不可用的时长，并将brokerName及”预计恢复时长“，存储于ConcurrentHashMap<String, FaultItem> faultItemTable中。
        预计恢复时长是通过latencyMax、notAvailableDuration来计算的。
    （2）在开启Broker延迟故障机制后，首先通过自增取模获取MessageQueue列表中的某个Queue。然后会根据系统当前时间与FaultItem中该broker（取模选中的Queue所属的broker）的预计恢复时间做比较，若(System.currentTimeMillis() - startTimestamp) >= 0，则预计该broker恢复正常，选择该broker的消息队列
    （3）若所有的broker都预计不可用，随机选择一个不可用的broker，从路由信息中选择下一个消息队列，重置其brokerName，queueId，进行消息发送

## 如何进行批量消息发送
RocketMQ支持同一个主题下的多条消息一次性发送到消息服务端。单条消息发送时，消息体的内容将保存在body中。批量消息发送，需要将多条消息体的内容存储在body中。如何存储方便服务端正确解析出每条消息呢？RocketMQ采取的方式是，对单条消息内容使用固定格式进行存储，格式如下：

![](/images/RocketMQ/RocketMQ批量消息封装格式.png)

将多条消息体内容编码到body中后，后续的发送流程和单条消息发送是完全一致的。

## RocketMQ有哪几种发送方式？
具有同步发送、异步发送、单向消息方式

    （1）同步发送：同步阻塞等待发送结果
    （2）异步发送：消息发送之后不阻塞等待结果，而是在MQ返回发送结果时回调SendCallback函数，若发送成功了就回调onSuccess函数，若发送失败了就回调onExceptino函数
    （3）单向消息：Broker根本不会返回发送结果。
    
其中方式1和2比较常见，具体使用哪一种方式需要根据业务情况来判断。而方式3，适合大数据场景，允许有一定消息丢失的场景。

思考：调研思考一下Kafka和RabbitMQ在使用的时候，有几种消息发送模式？有几种消息消费模式？系统若使用了MQ技术的话，那么平时使用的哪种消息发送模式？使用的是哪种消息消费模式？为什么？
几种消息发送模式下，在什么场景应该选用什么消息发送模式？几种消息消费模式下，在什么场景下应该选用什么消息消费模式？

## 生产者发送消息，怎么保证消息的可靠性？
生产者丢失消息的可能点在于程序发送失败抛异常了没有重试处理，或者发送的过程成功但是过程中网络闪断MQ没收到，消息就丢失了。由于同步发送的一般不会出现这样使用方式，所以就不考虑同步发送的问题，基于异步发送的场景来说。

异步发送分为两个方式：异步有回调和异步无回调，无回调的方式，生产者发送完后不管结果可能就会造成消息丢失，而通过异步发送+回调通知+本地消息表的形式就可以做出一个解决方案。以下单的场景举例。

    （1）下单后先保存本地数据和MQ消息表，这时候消息的状态是发送中，如果本地事务失败，那么下单失败，事务回滚。
    （2）下单成功，直接返回客户端成功，异步发送MQ消息
    （3）MQ回调通知消息发送结果，对应更新数据库MQ发送状态
    （4）JOB轮询超过一定时间（时间根据业务配置）还未发送成功的消息去重试
    （5）在监控平台配置或者JOB程序处理超过一定次数一直发送不成功的消息，告警，人工介入。
一般而言，对于大部分场景来说异步回调的形式就可以了，只有那种需要完全保证不能丢失消息的场景我们做一套完整的解决方案。

## producer与NameServer之间的长连接


思考：
（1）Kafka、RabbitMQ他们有类似的数据分片机制吗？
（2）他们是如何把一个逻辑上的数据集合概念（比如一个Topic）给在物理上拆分为多个数据分片的？
（3）拆分后的多个数据分片又是如何在物理的多台机器上分布式存储的
（4）为什么一定要让MQ实现数据分片的机制？
（5）如果不实现数据分片机制，来设计MQ中一个数据集合的分布式存储，你觉得好设计吗？
