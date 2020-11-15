# RocketMQ批量消息发送
批量消息发送方式就是将同一主题的多条消息一起打包发送到消息服务端，减少网络调用次数，提高网络传输效率。当然，并不是在同一批次中发送的消息数量越多性能就越好，其判断依据是单条消息的长度，如果单条消息内容比较长，则打包多条消息发送会影响其他线程发送消息的响应时间，并且单批次消息发送总长度不能超过DefaultMQProducer#maxMessageSize。批量消息发送要解决的是如何将这些消息编码以便服务端能够正确解码出每条消息的消息内容。那RocketMQ如何编码多条消息呢？可以先看一下RocketMQ网络请求命令设计。消息RemotingCommand的属性如下：

    （1）code：请求命令编码，请求命令类型。
    （2）version：版本号。
    （3）opaque：客户端请求序号。
    （4）flag：标记，倒数第一位表示请求类型，O：请求；1：返回。倒数第二位，值为1表示oneway。
    （5）remark：描述。
    （6）extFields：扩展属性。
    （7）customeHeader：每个请求对应的请求头信息。
    （8）byte[] body：消息体内容。

单条消息发送时，消息体的内容将保存在body中。批量消息发送，需要将多条消息体
的内容存储在body中。如何存储方便服务端正确解析出每条消息呢？RocketMQ采取的方式是，对单条消息内容使用固定格式进行存储，格式如下：

![](/images/RocketMQ/RocketMQ批量消息封装格式.png)

接下来介绍一个批量消息发送的流程，批量消息发送的入口是DefaultProducer#send()方法，代码如下：
```
DefaultProducer#send
public SendResult send(
    Collection<Message> msgs) throws MQClientException, RemotingException, 
        MQBrokerException, InterruptedException {
    return this.defaultMQProducerImpl.send(batch(msgs));
}

public SendResult send(Collection<Message> msgs,
    long timeout) throws MQClientException, RemotingException,
         MQBrokerException, InterruptedException {
    return this.defaultMQProducerImpl.send(batch(msgs), timeout);
}
```
首先在消息发送端，调用batch方法，将一批消息封装成MessageBatch对象。MessageBatch继承自Message对象，MessageBatch内部持有List<Message> messages。这样的话，批量消息发送与单条消息发送的处理流程完全一样。MessageBatch 只需要将该集合中的每条消息的消息体body聚合成一个bytep[]数组，在消息服务端能够从该byte[]数组中正确解析出消息即可。batch()方法如下：
```
private MessageBatch batch(Collection<Message> msgs) throws MQClientException {
    MessageBatch msgBatch;
    try {
        // 从Message的list生成批量消息体MessageBatch
        msgBatch = MessageBatch.generateFromList(msgs);
        for (Message message : msgBatch) {
            Validators.checkMessage(message, this);
            MessageClientIDSetter.setUniqID(message);
            message.setTopic(withNamespace(message.getTopic()));
        }
        // 设置消息体，此时的消息体已经是处理过后的批量消息体
        // 这里对MessageBatch进行消息编码处理，通过调用MessageBatch的encode方法实现
        msgBatch.setBody(msgBatch.encode());
    } catch (Exception e) {
        throw new MQClientException("Failed to initiate the MessageBatch", e);
    }
    // 设置topic
    msgBatch.setTopic(withNamespace(msgBatch.getTopic()));
    return msgBatch;
}
```
## generateFromList()方法
```
public static MessageBatch generateFromList(Collection<Message> messages) {
    assert messages != null;
    assert messages.size() > 0;
    // 首先实例化一个Message的list
    List<Message> messageList = new ArrayList<Message>(messages.size());
    Message first = null;

    // 对messages集合进行遍历
    for (Message message : messages) {
        // 判断延时级别，如果大于0抛出异常，原因为：批量消息发送不支持延时
        if (message.getDelayTimeLevel() > 0) {
            throw new UnsupportedOperationException("TimeDelayLevel is not supported for batching");
        }

        // 判断topic是否以 **"%RETRY%"** 开头，如果是，
        // 则抛出异常，原因为：批量发送消息不支持消息重试
        if (message.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
            throw new UnsupportedOperationException("Retry Group is not supported for batching");
        }

        // 判断集合中的每个Message的topic与批量发送topic是否一致，
        // 如果不一致则抛出异常，原因为：
        // 批量消息中的每个消息实体的Topic要和批量消息整体的topic保持一致。
        if (first == null) {
            first = message;
        } else {
            if (!first.getTopic().equals(message.getTopic())) {
                throw new UnsupportedOperationException("The topic of the messages in one batch should be the same");
            }

            // 判断批量消息的首个Message与其他的每个Message实体的等待消息存储状态是否相同，
            // 如果不同则报错，原因为：批量消息中每个消息的waitStoreMsgOK状态均应该相同。
            if (first.isWaitStoreMsgOK() != message.isWaitStoreMsgOK()) {
                throw new UnsupportedOperationException("The waitStoreMsgOK of the messages in one batch should the same");
            }
        }
        // 校验通过后，将message实体添加到messageList中
        messageList.add(message);
    }
    // 将处理完成的messageList作为构造方法，
    // 初始化MessageBatch实体，并设置topic以及isWaitStoreMsgOK状态。
    MessageBatch messageBatch = new MessageBatch(messageList);

    messageBatch.setTopic(first.getTopic());
    messageBatch.setWaitStoreMsgOK(first.isWaitStoreMsgOK());
    return messageBatch;
}
```
## encode()方法
对MessageBatch进行消息编码处理，通过调用MessageBatch的encode方法实现，代码逻辑如下：
```
public byte[] encode() {
    return MessageDecoder.encodeMessages(messages);
}

public static byte[] encodeMessages(List<Message> messages) {
    List<byte[]> encodedMessages = new ArrayList<byte[]>(messages.size());
    int allSize = 0;
    for (Message message : messages) {
        // 遍历messages集合，分别对每个Message实体进行编码操作，转换为byte[]
        byte[] tmp = encodeMessage(message);
        // 将转换后的单个Message的byte[]设置到encodedMessages中
        encodedMessages.add(tmp);
        // 批量消息的二进制数据长度随实际消息体递增
        allSize += tmp.length;
    }
    byte[] allBytes = new byte[allSize];
    int pos = 0;
    for (byte[] bytes : encodedMessages) {
        // 遍历encodedMessages，按序复制每个Message的二进制格式消息体
        System.arraycopy(bytes, 0, allBytes, pos, bytes.length);
        pos += bytes.length;
    }
    // 返回批量消息整体的消息体二进制数组
    return allBytes;
}

public static byte[] encodeMessage(Message message) {
        
    byte[] body = message.getBody();
    int bodyLen = body.length;
    String properties = messageProperties2String(message.getProperties());
    byte[] propertiesBytes = properties.getBytes(CHARSET_UTF8);
    //note properties length must not more than Short.MAX
    short propertiesLength = (short) propertiesBytes.length;
    int sysFlag = message.getFlag();
    int storeSize = 4 // 1 TOTALSIZE
        + 4 // 2 MAGICCOD
        + 4 // 3 BODYCRC
        + 4 // 4 FLAG
        + 4 + bodyLen // 4 BODY
        + 2 + propertiesLength;
    ByteBuffer byteBuffer = ByteBuffer.allocate(storeSize);
    // 1 TOTALSIZE
    byteBuffer.putInt(storeSize);

    // 2 MAGICCODE
    byteBuffer.putInt(0);

    // 3 BODYCRC
    byteBuffer.putInt(0);

    // 4 FLAG
    int flag = message.getFlag();
    byteBuffer.putInt(flag);

    // 5 BODY
    byteBuffer.putInt(bodyLen);
    byteBuffer.put(body);

    // 6 properties
    byteBuffer.putShort(propertiesLength);
    byteBuffer.put(propertiesBytes);

    return byteBuffer.array();
}
```
通过编码，最终将多条消息编码为固定格式，最终会被Broker端进行处理并持久化。除来上述调用bath()方法的逻辑之外，批量消息发送逻辑与单条消息发送没有任何差别。

