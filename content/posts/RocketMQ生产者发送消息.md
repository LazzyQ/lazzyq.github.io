---
title: "RocketMQ生产者发送消息"
date: 2020-11-27T22:31:12+08:00
draft: false
categories: 
  - "中间件"
tags: 
  - "RocketMQ"
---


前面的文章

* [RocketMQ通信的数据格式](https://lazzyq.github.io/2019/09/23/RocketMQ通信的数据格式/)介绍了RocketMQ各个组件之间通信，交换数据的格式

* [RocketMQ如何实现请求与响应](https://lazzyq.github.io/2019/09/24/RocketMQ如何实现请求与响应/)介绍了RocketMQ如何将通信数据发送出去并接受相应的响应
* [RocketMQ Namesrv启动流程](https://lazzyq.github.io/2019/09/23/RocketMQ-Namesrv启动流程/)介绍了RocketMQ中Namesrv组件的启动流程
* [RocketMQ Broker启动流程](https://lazzyq.github.io/2019/09/24/RocketMQ-Broker启动流程/)介绍了RocketMQ中Broker组件的启动流程

在了解了上面内容之后，我们就可以启动Producer向Broker发送消息，启动Consumer来消费响应的数据

这里Producer和Consumer都是RocketMQ的客户端，这篇文章先来介绍一下Producer的实现

<!--more-->

### Producer使用

首先看Producer，看源码之前，还得先知道怎么使用是吧

```java
public class SyncProducer {
    public static void main(String[] args) throws Exception {
        //使用ProducerGroup初始化Producer
        DefaultMQProducer producer = new
            DefaultMQProducer("please_rename_unique_group_name");
        // 指定namesrv
        producer.setNamesrvAddr("localhost:9876");
        producer.start();
        for (int i = 0; i < 100; i++) {
            //创建消息
            Message msg = new Message("TopicTest" /* Topic */,
                "TagA" /* Tag */,
                ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
            );
            //发送消息
            SendResult sendResult = producer.send(msg);
            System.out.printf("%s%n", sendResult);
        }
        //关闭Producer
        producer.shutdown();
    }
}
```

上面是同步发送消息的简单使用，主要是初始化DefaultMQProducer，调用start()方法和send()方法

### 初始化DefaultMQProducer

DefaultMQProducer的定义如下

```java
public class DefaultMQProducer extends ClientConfig implements MQProducer {
  ...
}
```

在MQProducer接口定义了Producer的一些操作，并且继承了MQAdmin，在MQAdmin主要是一些管理功能接口，MQProducer中定义了各种发送消息的结构，同步发送，异步发送，oneway发送，批量发送和事务消息的发送

<!--more-->

```java
public interface MQProducer extends MQAdmin {
    void start() throws MQClientException;

    void shutdown();

    List<MessageQueue> fetchPublishMessageQueues(final String topic) throws MQClientException;

    SendResult send(final Message msg) throws MQClientException, RemotingException, MQBrokerException,
        InterruptedException;

    // 省略一系列send方法

    // 失误相关的发送消息
    TransactionSendResult sendMessageInTransaction(final Message msg,
        final LocalTransactionExecuter tranExecuter, final Object arg) throws MQClientException;

    TransactionSendResult sendMessageInTransaction(final Message msg,
        final Object arg) throws MQClientException;

    //for batch
    SendResult send(final Collection<Message> msgs) throws MQClientException, RemotingException, MQBrokerException,
        InterruptedException;

    // 省略一系列的batch send的方法
}
```

```java
public interface MQAdmin {

    void createTopic(final String key, final String newTopic, final int queueNum)
        throws MQClientException;

    void createTopic(String key, String newTopic, int queueNum, int topicSysFlag)
        throws MQClientException;

    long searchOffset(final MessageQueue mq, final long timestamp) throws MQClientException;

    long maxOffset(final MessageQueue mq) throws MQClientException;

    long minOffset(final MessageQueue mq) throws MQClientException;

    long earliestMsgStoreTime(final MessageQueue mq) throws MQClientException;

    MessageExt viewMessage(final String offsetMsgId) throws RemotingException, MQBrokerException,
        InterruptedException, MQClientException;

    QueryResult queryMessage(final String topic, final String key, final int maxNum, final long begin,
        final long end) throws MQClientException, InterruptedException;
  
    MessageExt viewMessage(String topic,
        String msgId) throws RemotingException, MQBrokerException, InterruptedException, MQClientException;

}
```

然后是ClientConfig，这个类主要是一些配置数据，具体有哪些配置，每个配置是什么作用，用到了再细看，只需要知道有一些配置数据就行

```java
public class ClientConfig {
    public static final String SEND_MESSAGE_WITH_VIP_CHANNEL_PROPERTY = "com.rocketmq.sendMessageWithVIPChannel";
    private String namesrvAddr = NameServerAddressUtils.getNameServerAddresses();
    private String clientIP = RemotingUtil.getLocalAddress();
    private String instanceName = System.getProperty("rocketmq.client.name", "DEFAULT");
    private int clientCallbackExecutorThreads = Runtime.getRuntime().availableProcessors();
    protected String namespace;
    protected AccessChannel accessChannel = AccessChannel.LOCAL;

    private int pollNameServerInterval = 1000 * 30;
   
    private int heartbeatBrokerInterval = 1000 * 30;
   
    private int persistConsumerOffsetInterval = 1000 * 5;
    private boolean unitMode = false;
    private String unitName;
    private boolean vipChannelEnabled = Boolean.parseBoolean(System.getProperty(SEND_MESSAGE_WITH_VIP_CHANNEL_PROPERTY, "false"));

    private boolean useTLS = TlsSystemConfig.tlsEnable;

    private LanguageCode language = LanguageCode.JAVA;
}
```

大致过了一遍DefaultMQProducer的父类和实现了这些接口，DefaultMQProducer的主要功能就是发送消息，为了能够发送消息，再结合上面接口中定义的方法和配置，可以猜测一下DefaultMQProducer如何实现发送消息

1. 为了能够发送消息，首先我们得知道往哪里发送，所以DefaultMQProducer会从namesrv获取路由数据(就是可以向那些broker发送消息)
2. 如果知道了向那些broker发消息，接下来就是如何发?
   1. 由于broker是集群，所以需要做发送消息的均衡
   2. Producer需要向不同的broker机器发送消息，那么需要建立通信链路和管理这些连接
   3. 事务消息如何实现
   4. 除了发送消息，MQAdmin中还有一些管理功能的接口，不过这个比较简单，都是broker在干活，Producer只是通知broker干活而已

带着这些问题，继续深入DefaultMQProducer吧

DefaultMQProducer有很多的构造函数，不过完整构造就

```java
public DefaultMQProducer(final String namespace, final String producerGroup, RPCHook rpcHook,
    boolean enableMsgTrace, final String customizedTraceTopic) {
    this.namespace = namespace;
    this.producerGroup = producerGroup;
    defaultMQProducerImpl = new DefaultMQProducerImpl(this, rpcHook);
    //if client open the message trace feature
    if (enableMsgTrace) {
        try {
            AsyncTraceDispatcher dispatcher = new AsyncTraceDispatcher(customizedTraceTopic, rpcHook);
            dispatcher.setHostProducer(this.getDefaultMQProducerImpl());
            traceDispatcher = dispatcher;
            this.getDefaultMQProducerImpl().registerSendMessageHook(
                new SendMessageTraceHookImpl(traceDispatcher));
        } catch (Throwable e) {
            log.error("system mqtrace hook init failed ,maybe can't send msg trace data");
        }
    }
}
```

* namespace：这个现在不知道有啥用，初始化时也没用到，先忽略这个
* producerGroup：生产组，这个只是方便管理，给Producer进行归类的标识吧
* rpcHook：发送请求的rpcHook，这个在前面分析发送消息的时候有接触过，就是发送消息和结束后做一些处理的逻辑，可以和Spring生命周期中各种钩子类比，思想是一样的
* enableMsgTrace和customizedTraceTopic：都和消息轨迹有关，这个后面会单独分析，这里我们默认没有这个功能

初始化参数了解完后，再看一下初始化逻辑，又把功能丢给DefaultMQProducerImpl了，自己啥事也不干，那么我们继续探究DefaultMQProducerImpl

DefaultMQProducerImpl的构造函数初始化的东西也很多，这里就不贴代码了，反正就是有很多状态变量，Map，线程池之类的要初始化，至于都有啥用，用到了再说

### start()方法

DefaultMQProducer初始化完成后，会调用start()方法，其实主要逻辑都在start()里面

```java
public void start() throws MQClientException {
    this.setProducerGroup(withNamespace(this.producerGroup));
    this.defaultMQProducerImpl.start();
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
```

没错，它又把任务丢给defaultMQProducerImpl了，那我们看看defaultMQProducerImpl#start()方法

```java
public void start() throws MQClientException {
    this.start(true);
}
```

```java
public void start(final boolean startFactory) throws MQClientException {
    // 根据serviceState处理不同逻辑，初始化都是CREATE_JUST
    switch (this.serviceState) {
        case CREATE_JUST:
            this.serviceState = ServiceState.START_FAILED;
            // 检查配置信息，主要是ProducerGroup的，不重要
            this.checkConfig();
						// 判断ProducerGroup是否是CLIENT_INNER_PRODUCER_GROUP，如果是将 ClientConfig#instanceName 改为进程PID，什么情况下是CLIENT_INNER_PRODUCER_GROUP呢？接下来会接触到，由于我们都会指定自己的group，所以这里会进入到if语句里面修改ClientConfig#instanceName为进程PID
            if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                this.defaultMQProducer.changeInstanceNameToPID();
            }
						// 创建一个MQClientInstance，这个是重点，就是它完成了和broker的通信
            // 创建完成后MQClientManager会保存ClientId -> MQClientInstance的映射
            this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook);
						// 注册Producer，就是将 ProducerGroup -> DefaultMQProducerImpl 的关系保存到MQClientInstance中
            boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
            if (!registerOK) {
                // 如果注册失败，抛异常结束，终止启动
            }
						// topicPublishInfoTable 初始化一个topic的路由信息，这个CreateTopicKey就是MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC，这个topic和Broker的AutoCreateTopicEnable进行配置使用，后面会详细介绍
            this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());
						// startFactory这时是true，所以会调用MQClientInstance的start()方法
            if (startFactory) {
                mQClientFactory.start();
            }

            log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                this.defaultMQProducer.isSendMessageWithVIPChannel());
            this.serviceState = ServiceState.RUNNING;
            break;
        // 这里省略了一些 case 逻辑的处理，都是不重要的，逻辑都在CREATE_JUST这个case下面
    }
		// 发送心跳数据，这个后面再看吧
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
}
```

上面代码中的注释已经很详细了，如果跟着注释理解应该很容易吧，这里就不再啰嗦一遍了，当然也挖了很多坑

1. MixAll.CLIENT_INNER_PRODUCER_GROUP这个Group啥时候用？
2. 创建MQClientInstance，MQClientInstance是如何实现与其他主机通信的?
3. MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC和Broker的AutoCreateTopicEnable进行配置如何配合的？
4. MQClientInstance给谁发送心跳数据？怎么发送的？

接下来我们慢慢来填坑

#### **MixAll.CLIENT_INNER_PRODUCER_GROUP这个Group啥时候用？**

不要慌，MQClientInstance创建出来了，会调用它的start()方法，MixAll.CLIENT_INNER_PRODUCER_GROUP就在这里面

```java
public void start() throws MQClientException {

    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                // 如果没有指定namesrv的地址会发送一个Http请求从一个奇怪的地方获取，但是也不是给Producer用的，这里没看懂，但是不影响我们继续探索
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }
                // mQClientAPIImpl是帮助MQClientInstance完成与其他主机通信的功臣
                this.mQClientAPIImpl.start();
                this.startScheduledTask();
                this.pullMessageService.start();
                this.rebalanceService.start();
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                this.serviceState = ServiceState.RUNNING;
                break;
           // 这里省略了一些 case 逻辑的处理，都是不重要的，逻辑都在CREATE_JUST这个case下面
        }
    }
}
```

可以看到MQClientInstance#start()方法中又启动了一堆的东西，不过这里我们只关心2个: `MQClientAPIImpl`和`DefaultMQProducer`。`MQClientAPIImpl`其实是`NettyRemotingClient`的封装，后面会继续分析，这里主要看`DefaultMQProducer`

哇，这个就是我们一开始就介绍的那个`DefaultMQProducer`吗? No! No! No! 同样的类，但是这个`DefaultMQProducer`是MQClientInstance内部的，并不是初始化DefaultMQProducerImpl时`new DefaultMQProducerImpl(this, rpcHook)`，通过this传入的DefaultMQProducer。我当初读这段逻辑的时候饶了好久，才发现这个`DefaultMQProducer`不是同一个。我TM就要吐槽一句了，就不能换一个名字吗？哎，我们继续

初始化MQClientInstance时，就初始化了这个该死的`DefaultMQProducer`了，初始化的逻辑和我们自己的DefaultMQProducer一样，只不过这里的Group是我们期盼已久的MixAll.CLIENT_INNER_PRODUCER_GROUP了

```java
public MQClientInstance(ClientConfig clientConfig, int instanceIndex, String clientId, RPCHook rpcHook) {
  ...
	this.defaultMQProducer = new DefaultMQProducer(MixAll.CLIENT_INNER_PRODUCER_GROUP);
	...
}
```

然后这个MixAll.CLIENT_INNER_PRODUCER_GROUP的DefaultMQProducer调用了它内部的DefaultMQProducerImpl的start()方法，不过参数是false了，和我们自己的DefaultMQProducerImpl的start()参数是true不一样，又要走一遍上面的start()逻辑，但是不同的是，不再初始化MQClientInstance，MQClientManager在创建MQClientInstance后保存了一个ClientId -> MQClientInstance的映射，ClientId就是本机IP+PID，所以这次start()进去后会直接获取到前面创建的MQClientInstance，由于start()方法传入的参数是false，所以不会再调用MQClientInstance的start()方法，然后结束

上面一段是不是比较绕，我再画一张图来解释一下，其实就是有议程递归而已

![image-20191010015827771](/RocketMQ生产者发送消息/image-20191010015827771.png)

所以会初始化2个DefaultMQProducerImpl和DefaultMQProducer，但是他们都共用一个MQClientInstance，到这里start()流程就完了，但是前面我们挖的坑还没有填完，继续

#### **创建MQClientInstance，MQClientInstance是如何实现与其他主机通信的?**

前面我们谈到了很多MQClientInstance，但是都还没深入了解，现在就来肢解它吧

```java
public MQClientInstance(ClientConfig clientConfig, int instanceIndex, String clientId, RPCHook rpcHook) {
    this.clientConfig = clientConfig;
    this.instanceIndex = instanceIndex;
    this.nettyClientConfig = new NettyClientConfig();
    this.nettyClientConfig.setClientCallbackExecutorThreads(clientConfig.getClientCallbackExecutorThreads());
    this.nettyClientConfig.setUseTLS(clientConfig.isUseTLS());
    this.clientRemotingProcessor = new ClientRemotingProcessor(this);
    this.mQClientAPIImpl = new MQClientAPIImpl(this.nettyClientConfig, this.clientRemotingProcessor, rpcHook, clientConfig);

    if (this.clientConfig.getNamesrvAddr() != null) {
        this.mQClientAPIImpl.updateNameServerAddressList(this.clientConfig.getNamesrvAddr());
        log.info("user specified name server address: {}", this.clientConfig.getNamesrvAddr());
    }

    this.clientId = clientId;

    this.mQAdminImpl = new MQAdminImpl(this);

    this.pullMessageService = new PullMessageService(this);

    this.rebalanceService = new RebalanceService(this);

    this.defaultMQProducer = new DefaultMQProducer(MixAll.CLIENT_INNER_PRODUCER_GROUP);
    this.defaultMQProducer.resetClientConfig(clientConfig);

    this.consumerStatsManager = new ConsumerStatsManager(this.scheduledExecutorService);

    log.info("Created a new client Instance, InstanceIndex:{}, ClientID:{}, ClientConfig:{}, ClientVersion:{}, SerializerType:{}",
        this.instanceIndex,
        this.clientId,
        this.clientConfig,
        MQVersion.getVersionDesc(MQVersion.CURRENT_VERSION), RemotingCommand.getSerializeTypeConfigInThisServer());
}
```

构造函数还是比较简单，这里我们关注几个属性`NettyClientConfig`，`MQClientAPIImpl`和`MQAdminImpl`，这3个就是MQClientInstance实现通信功能的功臣

* NettyClientConfig：RocketMQ通信都是用Netty，那么要通信就得有相关的配置啊

* MQClientAPIImpl：它可是大功臣了，都是它在实现通信的，MQAdminImpl也得仰仗它

* MQAdminImpl：前面说了DefaultMQProducer实现了管理功能的接口，那么主要实现就在这里，其实还是要借助MQClientAPIImpl


既然MQClientAPIImpl这么NB，我们直接就深入敌方看看

在上面MQClientInstance的构造函数中对MQClientAPIImpl进行了初始化

```java
public MQClientAPIImpl(final NettyClientConfig nettyClientConfig,
    final ClientRemotingProcessor clientRemotingProcessor,
    RPCHook rpcHook, final ClientConfig clientConfig) {
    this.clientConfig = clientConfig;
    topAddressing = new TopAddressing(MixAll.getWSAddr(), clientConfig.getUnitName());
    this.remotingClient = new NettyRemotingClient(nettyClientConfig, null);
    this.clientRemotingProcessor = clientRemotingProcessor;

    this.remotingClient.registerRPCHook(rpcHook);
    this.remotingClient.registerProcessor(RequestCode.CHECK_TRANSACTION_STATE, this.clientRemotingProcessor, null);

    this.remotingClient.registerProcessor(RequestCode.NOTIFY_CONSUMER_IDS_CHANGED, this.clientRemotingProcessor, null);

    this.remotingClient.registerProcessor(RequestCode.RESET_CONSUMER_CLIENT_OFFSET, this.clientRemotingProcessor, null);

    this.remotingClient.registerProcessor(RequestCode.GET_CONSUMER_STATUS_FROM_CLIENT, this.clientRemotingProcessor, null);

    this.remotingClient.registerProcessor(RequestCode.GET_CONSUMER_RUNNING_INFO, this.clientRemotingProcessor, null);

    this.remotingClient.registerProcessor(RequestCode.CONSUME_MESSAGE_DIRECTLY, this.clientRemotingProcessor, null);
}
```

* NettyClientConfig：初始化Netty客户端的配置
* ClientConfig：也是一些配置
* ClientRemotingProcessor：请求结果响应处理
* RPCHook：请求时的钩子

可以看到MQClientAPIImpl初始化了NettyRemotingClient，通过NettyRemotingClient就能发送各种请求了，具体可以参考[RocketMQ如何实现请求与响应](https://lazzyq.github.io/2019/09/24/RocketMQ如何实现请求与响应/)，发送请求后，对方可能会返回一些响应数据，还得有处理这些响应数据的逻辑，他就是ClientRemotingProcessor了，在MQClientAPIImpl的构造函数中可以看到，ClientRemotingProcessor被注册到NettyRemotingClient，当了NettyRemotingClient小跟班，从注册的类型可以大概看出ClientRemotingProcessor会处理那些响应数据

* RequestCode.CHECK_TRANSACTION_STATE
* RequestCode.NOTIFY_CONSUMER_IDS_CHANGED
* RequestCode.RESET_CONSUMER_CLIENT_OFFSET
* RequestCode.GET_CONSUMER_STATUS_FROM_CLIENT
* RequestCode.GET_CONSUMER_RUNNING_INFO
* RequestCode.CONSUME_MESSAGE_DIRECTLY

这里我们大概有个印象，到时候分析各种请求的时候会细讲

既然知道了MQClientAPIImpl发送请求都是通过NettyRemotingClient，那么在MQClientAPIImpl中，我们随便找一个发送请求的方法看看

```java
public TopicList getSystemTopicListFromBroker(final String addr, final long timeoutMillis)
    throws RemotingException, MQClientException, InterruptedException {
    RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_BROKER, null);

    RemotingCommand response = this.remotingClient.invokeSync(MixAll.brokerVIPChannel(this.clientConfig.isVipChannelEnabled(), addr),
        request, timeoutMillis);
    assert response != null;
    switch (response.getCode()) {
        case ResponseCode.SUCCESS: {
            byte[] body = response.getBody();
            if (body != null) {
                TopicList topicList = TopicList.decode(body, TopicList.class);
                return topicList;
            }
        }
        default:
            break;
    }

    throw new MQClientException(response.getCode(), response.getRemark());
}
```

可以发现，MQClientAPIImpl中的请求都是先构造RemotingCommand，然后通过NettyRemotingClient发送请求，最后处理响应数据

这个就是MQClientInstance的秘密，知道了它的秘密以后它不管再添加啥请求，都是一样的

#### MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC和Broker的AutoCreateTopicEnable进行配置如何配合的？

这个我们在分析send()方法的时候会分析

### send()方法

前面说了那么多，到现在才到发送消息的方法

在DefaultMQProducer中有各种send()方法，我们只分析最简单的这种通过发送消息，根据前面的分析，最后干活的都是DefaultMQProducerImpl，这次也没有例外，还是让DefaultMQProducerImpl发送消息了，最后都会到这个方法去发送消息

```java
private SendResult sendDefaultImpl(
    Message msg,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final long timeout
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    // 检查状态和消息
    this.makeSureStateOK();
    Validators.checkMessage(msg, this.defaultMQProducer);

    final long invokeID = random.nextLong();
    long beginTimestampFirst = System.currentTimeMillis();
    long beginTimestampPrev = beginTimestampFirst;
    long endTimestamp = beginTimestampFirst;
    // 获取topic的路由数据，这里是重点，这里就是MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC和Broker的AutoCreateTopicEnable进行配置如何配合的？的答案了
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        boolean callTimeout = false;
        MessageQueue mq = null;
        Exception exception = null;
        SendResult sendResult = null;
        // 如果是同步发送，会有重试机制
        int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
        int times = 0;
        String[] brokersSent = new String[timesTotal];
        for (; times < timesTotal; times++) {
            String lastBrokerName = null == mq ? null : mq.getBrokerName();
            // 这里是发送消息负载均衡的关键
            MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
            if (mqSelected != null) {
                mq = mqSelected;
                brokersSent[times] = mq.getBrokerName();
                try {
                    beginTimestampPrev = System.currentTimeMillis();
                    if (times > 0) {
                        //Reset topic with namespace during resend.
                        msg.setTopic(this.defaultMQProducer.withNamespace(msg.getTopic()));
                    }
                    long costTime = beginTimestampPrev - beginTimestampFirst;
                    if (timeout < costTime) {
                        callTimeout = true;
                        break;
                    }
										// 真正干活的老弟在这
                    sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                    endTimestamp = System.currentTimeMillis();
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                    switch (communicationMode) {
                        case ASYNC:
                            return null;
                        case ONEWAY:
                            return null;
                        case SYNC:
                            if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                    continue;
                                }
                            }

                            return sendResult;
                        default:
                            break;
                    }
                } catch () {
                	// 这里是一对catch Exception，省略了
                }
            } else {
                break;
            }
        }

				// 如果发送成功，返回
        if (sendResult != null) {
            return sendResult;
        }

        // 这里还有一堆处理异常的逻辑，也省略了

        throw mqClientException;
    }

    // 没有路由信息的一些异常处理
}
```

首先看一下参数

* Message：这个就是我们要发送的消息数据，没啥说的
* CommunicationMode：就是发送消息的方式，同步？异步？还是Oneway？
* SendCallback：这个是异步发送用的，这个在[RocketMQ如何实现请求与响应](https://lazzyq.github.io/2019/09/24/RocketMQ如何实现请求与响应/)提过了
* timeout：超时时间，这个也没什么说的

这段代码大部分都是在处理异常，当然干(挖)货(坑)也是比较多的

1. 首先就是回答MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC和Broker的AutoCreateTopicEnable进行配置如何配合的？
2. 发送消息如何实现负载均衡
3. sendKernelImpl()这个干活的小老弟怎么发送消息的

注意到当时同步发送消息时，加入了重试机制，从代码中发现，重试时的超时时间并不是每次请求都重置了的，不管重试多少次，超时时间都是控制在我们传入的timeout之内的

好了，新坑又来了，继续填吧

#### MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC和Broker的AutoCreateTopicEnable进行配置如何配合的？

这段功能就是获取topic的路由信息

```java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    // 根据topic从Map中直接获取路由数据，Producer启动时肯定是啥都没有的
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        // 直接put进去一个空的TopicPublishInfo
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        // 从namesrv获取这个topic的路由信息
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }
		// 如果从namesrv中拿到路由数据了直接返回
    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else { // namesrv中也没有再从namesrv中获取？头铁?不撞南墙不回头？
        // 这个updateTopicRouteInfoFromNameServer和上面的那个方法是不同的逻辑了，并不是重试
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```

这里需要插播一点知识点了，Topic的创建有2种方式

1. 通过管理功能的接口创建(就是手动创建)
2. 自动创建

手动创建的方式有很多，可以通过Console管理页面创建，可以通过mqadmin创建，也可以直接通过Client包写代码自己创建，这里就不细说了，这里主要看自动创建，这个就是MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC的作用了

如果要使用自动创建，broker的autoCreateTopicEnable需要配置成true，不过官方建议线上不要开，线下用用就好了

![image-20191010115840768](/RocketMQ生产者发送消息/image-20191010115840768.png)

插播完了，我们回来继续分析。如果手动创建了topic，那么在第一次updateTopicRouteInfoFromNameServer(topic)就能获取到路由数据了

```java
this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
```

但是我们并没有手动创建，所以会走到updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);

```java
this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
```

注意这次的updateTopicRouteInfoFromNameServer的第二个参数，设置成了true

```java
public boolean updateTopicRouteInfoFromNameServer(final String topic, boolean isDefault,
    DefaultMQProducer defaultMQProducer) {
    try {
        if (this.lockNamesrv.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) {
            try {
                TopicRouteData topicRouteData;
                // 如果是默认的，调用mQClientAPIImpl#getDefaultTopicRouteInfoFromNameServer获取默认的路由信息
                // defaultMQProducer.getCreateTopicKey()返回的就是MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC这个Topic了
                if (isDefault && defaultMQProducer != null) {
                    topicRouteData = this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(defaultMQProducer.getCreateTopicKey(),
                        1000 * 3);
                    if (topicRouteData != null) {
                        for (QueueData data : topicRouteData.getQueueDatas()) {
                            int queueNums = Math.min(defaultMQProducer.getDefaultTopicQueueNums(), data.getReadQueueNums());
                            data.setReadQueueNums(queueNums);
                            data.setWriteQueueNums(queueNums);
                        }
                    }
                } else { // 如果不是默认的，updateTopicRouteInfoFromNameServer(topic)就会到这里来
                    topicRouteData = this.mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3);
                }
                // 获取到路由数据，这里可能是默认Topic或是我们自己的Topic的路由信息
                if (topicRouteData != null) {
                    TopicRouteData old = this.topicRouteTable.get(topic);
                    boolean changed = topicRouteDataIsChange(old, topicRouteData);
                    if (!changed) {
                        changed = this.isNeedUpdateTopicRouteInfo(topic);
                    } else {
                        log.info("the topic[{}] route info changed, old[{}] ,new[{}]", topic, old, topicRouteData);
                    }
										// 后面就是更新信息了
                    if (changed) {
                       ...
                    }
                } else {
                    log.warn("updateTopicRouteInfoFromNameServer, getTopicRouteInfoFromNameServer return null, Topic: {}", topic);
                }
            } catch (Exception e) {
                if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX) && !topic.equals(MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC)) {
                    log.warn("updateTopicRouteInfoFromNameServer Exception", e);
                }
            } finally {
                this.lockNamesrv.unlock();
            }
        } else {
            log.warn("updateTopicRouteInfoFromNameServer tryLock timeout {}ms", LOCK_TIMEOUT_MILLIS);
        }
    } catch (InterruptedException e) {
        log.warn("updateTopicRouteInfoFromNameServer Exception", e);
    }

    return false;
}
```

可以发现，这里做了一个“偷天换日”的操作，如果没有手动创建topic的话，会拿MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC这个Topic的路由数据当做自己的路由数据，怎么样，见识到了这种操作，是不是想说：秒(N)啊(B)！秒(厉)啊(害)！秒(S)啊(B)！

由此我们可以猜想，如果Broker将autoCreateTopicEnable需要配置成true，那么Broker同样会把自己的配置注册给namesrv并且Topic是MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC，不信我们看看

在Broker中的TopicConfigManager初始化的一段逻辑如下

```java
// MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC
if (this.brokerController.getBrokerConfig().isAutoCreateTopicEnable()) {
    String topic = MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC;
    TopicConfig topicConfig = new TopicConfig(topic);
    this.systemTopicList.add(topic);
    topicConfig.setReadQueueNums(this.brokerController.getBrokerConfig()
        .getDefaultTopicQueueNums());
    topicConfig.setWriteQueueNums(this.brokerController.getBrokerConfig()
        .getDefaultTopicQueueNums());
    int perm = PermName.PERM_INHERIT | PermName.PERM_READ | PermName.PERM_WRITE;
    topicConfig.setPerm(perm);
    this.topicConfigTable.put(topicConfig.getTopicName(), topicConfig);
}
```

TopicConfigManager会将MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC的配置添加到topicConfigTable，并定时上报给namesrv，完美。

这个坑我们就算填平了，下一坑位走起

#### 发送消息如何实现负载均衡？

获取到Topic的路由信息后，会调用selectOneMessageQueue()方法获取一个MessageQueue

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    return this.mqFaultStrategy.selectOneMessageQueue(tpInfo, lastBrokerName);
}
```

这个方式交给了MQFaultStrategy处理，继续深入

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    // Broker故障延迟机制，默认是false，不开启，后面再分析
    if (this.sendLatencyFaultEnable) {
        try {
            int index = tpInfo.getSendWhichQueue().getAndIncrement();
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                    if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                        return mq;
                }
            }

            final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
            int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
            if (writeQueueNums > 0) {
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
		// 没有开启Broker故障延迟机制会走这里
    return tpInfo.selectOneMessageQueue(lastBrokerName);
}
```

可以发现这里有sendLatencyFaultEnable配置，表示是否开启Broker故障延迟机制，待会我们再细说这个机制是啥，我们先看普通机制下的负载均衡

```java
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
    // lastBrokerName是上一次发送失败的broker name，第一次会直接进入
    if (lastBrokerName == null) {
        return selectOneMessageQueue();
    } else {
        // 如果上次选择的broker发送失败了
        // sendWhichQueue是一个通过ThreadLocal保存的Integer值
        // 通过这个自增的index和messageQueueList.size()求余进行负载均衡
        int index = this.sendWhichQueue.getAndIncrement();
        for (int i = 0; i < this.messageQueueList.size(); i++) {
            int pos = Math.abs(index++) % this.messageQueueList.size();
            if (pos < 0)
                pos = 0;
            MessageQueue mq = this.messageQueueList.get(pos);
            // 如果和上次发送失败的还是同一个broker，继续选择下一个
            if (!mq.getBrokerName().equals(lastBrokerName)) {
                return mq;
            }
        }
        // 到这里还是没找到，那么只能被迫选择一个了，不管是不是上次发送失败的broker
        return selectOneMessageQueue();
    }
}
```

```java
public MessageQueue selectOneMessageQueue() {
    int index = this.sendWhichQueue.getAndIncrement();
    int pos = Math.abs(index) % this.messageQueueList.size();
    if (pos < 0)
        pos = 0;
    return this.messageQueueList.get(pos);
}
```

可以发现，普通情况下的负载均衡很简单，就是通过自增的一个index和messageQueueList.size()求余实现的，由于同步发送有重试机制，所以重试时会判断是否是上次发送失败的broker，如果是会跳过这个有问题的broker

前面我们还提到了Broker故障延迟机制，这个是个什么机制，是不是很高级？

再看源码之前，我们可以思考一个问题，Broker是集群，虽然普通情况下同步发送会对上次发送失败的broker进行判断，但是这么做也只是减少一些情况的没必要调用，还是会存在一些情况导致没必要的调用，比如

1. 异步发送并没有对上次发送失败broker的判断，所以多次异步发送会选择到broker有问题的主机进行发送
2. 同步发送只记住了上一次发送失败的broker，那么多次发送消息后，还是会轮到这个有问题broker
3. 其他情况... 大家可以再想想还有没有情况导致一些没必要的调用

如果你明白了上述问题，那么怎么解决呢？rockerMQ告诉你用Broker故障延迟机制

#### Broker故障延迟机制

我们看看RocketMQ的Broker故障延迟机制是怎么减少前面提到的没必要的调用的

```java
if (this.sendLatencyFaultEnable) {
    try {
        int index = tpInfo.getSendWhichQueue().getAndIncrement();
        for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
            int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
            if (pos < 0)
                pos = 0;
            MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
            // latencyFaultTolerance这里就是实现Broker故障延迟机制的关键了
            if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                    return mq;
            }
        }
				// 到这里都还没找到合适的MessageQueue，那么也只能随便挑一个了，总得给上级一个交代是吧
        // pickOneAtLeast()就不再细说了，大家可以自行了解，反正大概就是从差的里面再挑一个好的出来
        final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
        int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
        if (writeQueueNums > 0) {
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
```

可以看到，进入if里面后，和普通机制的处理差不多，会选择一个MessageQueue，但是这里调用了LatencyFaultTolerance对选择的

MessageQueue的broker进行了判断，如果可用的话，才会继续使用，不可用的话继续选择下一个

那么我们看看LatencyFaultTolerance#isAvailable()怎么做的吧

```java
public boolean isAvailable(final String name) {
    final FaultItem faultItem = this.faultItemTable.get(name);
    if (faultItem != null) {
        return faultItem.isAvailable();
    }
    return true;
}
```

逻辑比较简单，从faultItemTable中根据broker name获取到FaultItem对象，如果FaultItem不存在说明这个broker是可用的，如果FaultItem不为空，根据FaultItem#isAvailable()判断是否可用

```java
private final ConcurrentHashMap<String, FaultItem> faultItemTable = new ConcurrentHashMap<String, FaultItem>(16);
```

faultItemTable是一个Map，那么这个faultItemTable是哪更新它的数据呢？原来更新这个faultItemTable都是在sendDefaultImpl()发送消息里面做的

如果消息发送成功会更新faultItemTable

```java
sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
endTimestamp = System.currentTimeMillis();
this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
```

发送失败了也会更新

```java
} catch (RemotingException e) {
    endTimestamp = System.currentTimeMillis();
    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
    ...
} catch (MQClientException e) {
    endTimestamp = System.currentTimeMillis();
    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
    ...
} catch (MQBrokerException e) {
    endTimestamp = System.currentTimeMillis();
    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
    ...
} catch (InterruptedException e) {
    endTimestamp = System.currentTimeMillis();
    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
    ...
}
```

可以发现，消息发送成功，第3个参数isolation是false，消息发送失败，第3个参数isolation是true

```java
public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
    this.mqFaultStrategy.updateFaultItem(brokerName, currentLatency, isolation);
}
```

```java
public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
    if (this.sendLatencyFaultEnable) {
        long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
        this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);
    }
}
```

```java
private long computeNotAvailableDuration(final long currentLatency) {
    for (int i = latencyMax.length - 1; i >= 0; i--) {
        if (currentLatency >= latencyMax[i])
            return this.notAvailableDuration[i];
    }

    return 0;
}
```

```java
private long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L};
private long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L};
```

在更新faultItemTable前，会根据当前发送消息的耗时(也就是第2个参数currentLatency)计算出一个duration，计算逻辑很简单，就是根据latencyMax和notAvailableDuration这2个数组计算的

* 如果发送消息的耗时currentLatency < 100ms，那么duration=0
* 发送消息的耗时currentLatency > 100ms时，根据不同的延时会得到不同的duration
* 如果消息发送失败了，duration直接等于30000

最后才是更新faultItemTable

```java
public void updateFaultItem(final String name, final long currentLatency, final long notAvailableDuration) {
    FaultItem old = this.faultItemTable.get(name);
    // 如果是新来的，创建FaultItem，并设置CurrentLatency和StartTimestamp
    // StartTimestamp就是当前时间+前面根据CurrentLatency计算得到的duration
    if (null == old) {
        final FaultItem faultItem = new FaultItem(name);
        faultItem.setCurrentLatency(currentLatency);
        faultItem.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
				// 这里不知道为什么还有old，是担心faultItemTable这个ConcurrentHashMap线程不够安全吗？
        old = this.faultItemTable.putIfAbsent(name, faultItem);
        if (old != null) {
            old.setCurrentLatency(currentLatency);
            old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
        }
    } else {
        // 如果是老油条了，直接更新就行
        old.setCurrentLatency(currentLatency);
        old.setStartTimestamp(System.currentTimeMillis() + notAvailableDuration);
    }
}
```

现在知道了FaultItem对象的属性是啥意思了，并且知道这些属性是怎么来的了，我们再回头看LatencyFaultTolerance#isAvailable()

```java
public boolean isAvailable(final String name) {
    final FaultItem faultItem = this.faultItemTable.get(name);
    if (faultItem != null) {
        return faultItem.isAvailable();
    }
    return true;
}
```

现在可以理解了，如果faultItemTable.get(name)没有获取到FaultItem，自然这个broker就是没问题的，如果获取到了FaultItem，还不能说这个broker有问题，因为正常发送消息成功也会更新faultItemTable，所以还得看faultItem#isAvailable()

```java
public boolean isAvailable() {
    return (System.currentTimeMillis() - startTimestamp) >= 0;
}
```

也很简单，就是判断当前时间是否大于startTimestamp，startTimestamp的含义前面已经说过了

最后总结一些，Broker故障延迟机制既不神秘也不复杂，其实就是根据当前发送消息的耗时，对相应的broker进行一段时间屏蔽，耗时越长，屏蔽的时间就越长，如果发送消息失败，屏蔽时间最长

#### sendKernelImpl()这个干活的小老弟怎么发送消息的

前面的坑基本填完了，最后看看真正发送消息的sendKernelImpl()方法吧

```java
private SendResult sendKernelImpl(final Message msg,
                                  final MessageQueue mq,
                                  final CommunicationMode communicationMode,
                                  final SendCallback sendCallback,
                                  final TopicPublishInfo topicPublishInfo,
                                  final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    long beginStartTime = System.currentTimeMillis();
    // 根据broker name获取master的地址
    String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    if (null == brokerAddr) {
        // 如果没有获取到master的地址，刷新路由信息
        tryToFindTopicPublishInfo(mq.getTopic());
        brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    }

    SendMessageContext context = null;
    if (brokerAddr != null) {
        // 判断是不是VIP，如果配置了是VIP发送消息，name会向broker的另外一个端口发送消息
        // VIP走小红帽通道
        brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);

        byte[] prevBody = msg.getBody();
        try {
            // 省略了现在我们不关心的内容
            SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
            requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
            requestHeader.setTopic(msg.getTopic());
            requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
            requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
            requestHeader.setQueueId(mq.getQueueId());
            requestHeader.setSysFlag(sysFlag);
            requestHeader.setBornTimestamp(System.currentTimeMillis());
            requestHeader.setFlag(msg.getFlag());
            requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
            requestHeader.setReconsumeTimes(0);
            requestHeader.setUnitMode(this.isUnitMode());
            requestHeader.setBatch(msg instanceof MessageBatch);
           
            // 省略了现在我们不关心的内容
            SendResult sendResult = null;
            switch (communicationMode) {
                case ASYNC:
                    Message tmpMessage = msg;
                    
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                        brokerAddr,
                        mq.getBrokerName(),
                        tmpMessage,
                        requestHeader,
                        timeout - costTimeAsync,
                        communicationMode,
                        sendCallback,
                        topicPublishInfo,
                        this.mQClientFactory,
                        this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(),
                        context,
                        this);
                    break;
                case ONEWAY:
                case SYNC:
                    long costTimeSync = System.currentTimeMillis() - beginStartTime;
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
                        brokerAddr,
                        mq.getBrokerName(),
                        msg,
                        requestHeader,
                        timeout - costTimeSync,
                        communicationMode,
                        context,
                        this);
                    break;
                default:
                    assert false;
                    break;
            }

            return sendResult;
        } catch () {
            // 省略了现在我们不关心的内容
        } finally {
            msg.setBody(prevBody);
            msg.setTopic(NamespaceUtil.withoutNamespace(msg.getTopic(), this.defaultMQProducer.getNamespace()));
        }
    }

    throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
}
```

还是比较简单的，就是构造SendMessageRequestHeader，然后交个MQClientAPIImpl去发送消息

需要注意的是，这里有个VIP走了小红帽通道，从代码中看，就是修改了broker的端口

```
brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);
```

```java
public static String brokerVIPChannel(final boolean isChange, final String brokerAddr) {
    if (isChange) {
        String[] ipAndPort = brokerAddr.split(":");
        String brokerAddrNew = ipAndPort[0] + ":" + (Integer.parseInt(ipAndPort[1]) - 2);
        return brokerAddrNew;
    } else {
        return brokerAddr;
    }
}
```

还有就是，发送消息给broker都是给master发的

到这里，发送消息基本就完成了，但仍然有很多没有说到，比如事务发送消息，批量发送消息等等。一步一个脚印，我们慢慢来，这篇文章的内容个人感觉已经很多了，Producer的其他功能后面再单独写吧，以上。