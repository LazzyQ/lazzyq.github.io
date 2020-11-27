---
title: "RocketMQ Broker启动流程"
date: 2020-11-27T22:38:54+08:00
draft: false
original: true
categories: 
  - "中间件"
tags: 
  - "RocketMQ"
---


Broker的启动类是`BrokerStartup`

```java
public static void main(String[] args) {
    start(createBrokerController(args));
}
```

和Namesrv启动差不多，首先创建一个`BrokerController`，然后调用start()方法

<!--more-->

```java
public static BrokerController createBrokerController(String[] args) {
    // 省略了从环境变量中读取配置设置到NettySystemConfig

    try {
        // 和Namesrv差不多，使用命令行工具包commons-cli根据命令行参数解析成CommandLine对象
        Options options = ServerUtil.buildCommandlineOptions(new Options());
        commandLine = ServerUtil.parseCmdLine("mqbroker", args, buildCommandlineOptions(options),
            new PosixParser());
        if (null == commandLine) {
            System.exit(-1);
        }
        // 这里有3个配置类，相比Namesrv只有NamesrvConfig，NettyServerConfig2个配置类
        final BrokerConfig brokerConfig = new BrokerConfig();
        final NettyServerConfig nettyServerConfig = new NettyServerConfig();
        final NettyClientConfig nettyClientConfig = new NettyClientConfig();

        // TLS相关 
        nettyClientConfig.setUseTLS(Boolean.parseBoolean(System.getProperty(TLS_ENABLE,
            String.valueOf(TlsSystemConfig.tlsMode == TlsMode.ENFORCING))));
        // Broker监听端口10911
        nettyServerConfig.setListenPort(10911);
        final MessageStoreConfig messageStoreConfig = new MessageStoreConfig();

        // BrokerRole如果是SLAVE设置accessMessageInMemoryMaxRatio，MessageStoreConfig中有一堆关于message相关的配置，后面总结配置的时候一起写
        if (BrokerRole.SLAVE == messageStoreConfig.getBrokerRole()) {
            int ratio = messageStoreConfig.getAccessMessageInMemoryMaxRatio() - 10;
            messageStoreConfig.setAccessMessageInMemoryMaxRatio(ratio);
        }
        // 注意了，-c 选项又来了，在Namesrv启动时 -c 指定的配置文件被反复蹂躏，那么在Broker启动的时候，她的命运是否还是如此呢？
        if (commandLine.hasOption('c')) {
            String file = commandLine.getOptionValue('c');
            if (file != null) {
                configFile = file;
                InputStream in = new BufferedInputStream(new FileInputStream(file));
                properties = new Properties();
                properties.load(in);
                // 从配置文件中读取了2个配置设置到了系统属性中
                properties2SystemEnv(properties);
                // 下面就很熟悉了，将配置文件中的配置拆分到BrokerConfig,NettyClientConfig,NettyServerConfig,MessageStoreConfig
                MixAll.properties2Object(properties, brokerConfig);
                MixAll.properties2Object(properties, nettyServerConfig);
                MixAll.properties2Object(properties, nettyClientConfig);
                MixAll.properties2Object(properties, messageStoreConfig);
                // 记录配置文件
                BrokerPathConfigHelper.setBrokerConfigPath(file);
                in.close();
            }
        }
        // 命令行中指定的配置设置到BrokerConfig
        MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), brokerConfig);
        // 检查rocketmqHome是否指定，这里和Namesrv差不多
        // 在BrokerConfig中，可以看到使用系统属性rocketmq.home.dir，环境变量ROCKETMQ_HOME和前面的-c指定的配置文件设置RocketMQHome
        // 在mqnamesrv启动脚本中会自定探测RockerMQ并export ROCKETMQ_HOME
        if (null == brokerConfig.getRocketmqHome()) {
            System.out.printf("Please set the %s variable in your environment to match the location of the RocketMQ installation", MixAll.ROCKETMQ_HOME_ENV);
            System.exit(-2);
        }
        // 检查配置的NamesrvAddr格式是否正确，检查方式就是根据Addr能够构造出InetSocketAddress对象
        String namesrvAddr = brokerConfig.getNamesrvAddr();
        if (null != namesrvAddr) {
            try {
                String[] addrArray = namesrvAddr.split(";");
                for (String addr : addrArray) {
                    RemotingUtil.string2SocketAddress(addr);
                }
            } catch (Exception e) {
                System.out.printf(
                    "The Name Server Address[%s] illegal, please set it as follows, \"127.0.0.1:9876;192.168.0.1:9876\"%n",
                    namesrvAddr);
                System.exit(-3);
            }
        }
				// 根据BrokerRole设置BrokerId，不过这里的BrokerRole都没有赋值，都是默认值ASYNC_MASTER
        switch (messageStoreConfig.getBrokerRole()) {
            case ASYNC_MASTER:
            case SYNC_MASTER:
                brokerConfig.setBrokerId(MixAll.MASTER_ID);
                break;
            case SLAVE:
                if (brokerConfig.getBrokerId() <= 0) {
                    System.out.printf("Slave's brokerId must be > 0");
                    System.exit(-3);
                }

                break;
            default:
                break;
        }
				// 又来一个端口
        messageStoreConfig.setHaListenPort(nettyServerConfig.getListenPort() + 1);

        // 加载logback，根据命令行选项-p和-m在控制台打印打印配置信息
        if (commandLine.hasOption('p')) {
            InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.BROKER_CONSOLE_NAME);
            MixAll.printObjectProperties(console, brokerConfig);
            MixAll.printObjectProperties(console, nettyServerConfig);
            MixAll.printObjectProperties(console, nettyClientConfig);
            MixAll.printObjectProperties(console, messageStoreConfig);
            System.exit(0);
        } else if (commandLine.hasOption('m')) {
            InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.BROKER_CONSOLE_NAME);
            MixAll.printObjectProperties(console, brokerConfig, true);
            MixAll.printObjectProperties(console, nettyServerConfig, true);
            MixAll.printObjectProperties(console, nettyClientConfig, true);
            MixAll.printObjectProperties(console, messageStoreConfig, true);
            System.exit(0);
        }
				// 还是打印配置，只是打印在logback配置的文件中
        log = InternalLoggerFactory.getLogger(LoggerName.BROKER_LOGGER_NAME);
        MixAll.printObjectProperties(log, brokerConfig);
        MixAll.printObjectProperties(log, nettyServerConfig);
        MixAll.printObjectProperties(log, nettyClientConfig);
        MixAll.printObjectProperties(log, messageStoreConfig);
				// BrokerController终于初始化了
        final BrokerController controller = new BrokerController(
            brokerConfig,
            nettyServerConfig,
            nettyClientConfig,
            messageStoreConfig);
        // 仍然在蹂躏可怜的配置文件
        controller.getConfiguration().registerConfig(properties);
				// 这里直接调用了initialize()方法
        boolean initResult = controller.initialize();
        if (!initResult) {
            controller.shutdown();
            System.exit(-3);
        }
				// shutdown的钩子
        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            private volatile boolean hasShutdown = false;
            private AtomicInteger shutdownTimes = new AtomicInteger(0);

            @Override
            public void run() {
                synchronized (this) {
                    log.info("Shutdown hook was invoked, {}", this.shutdownTimes.incrementAndGet());
                    if (!this.hasShutdown) {
                        this.hasShutdown = true;
                        long beginTime = System.currentTimeMillis();
                        controller.shutdown();
                        long consumingTimeTotal = System.currentTimeMillis() - beginTime;
                        log.info("Shutdown hook over, consuming total time(ms): {}", consumingTimeTotal);
                    }
                }
            }
        }, "ShutdownHook"));

        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }

    return null;
}
```

```java
public static BrokerController start(BrokerController controller) {
    try {
        controller.start();
				// 省略了打印日志      
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }

    return null;
}
```

可以看到，Broker和Namesrv的启动都差不多，首先根据命令的选项和配置初始化一些配置类，然后用这些配置类创建一个Controller，Controller根据配置在initialize()方法中初始化一些信息，然后在start()中才真正的启动服务监听端口

不同点呢，就是Broker比Namesrv配置类多，逻辑可能就相对更加复杂一点，不用怕，我们慢慢啃

扯回来，启动主要逻辑都在BrokerController.initialize()和BrokerController.start()方法中

首先看看BrokerController中初始化了啥东西，WTF，咋这么多东西，先不管了，就认为初始化了很多东西

```java
public BrokerController(
    final BrokerConfig brokerConfig,
    final NettyServerConfig nettyServerConfig,
    final NettyClientConfig nettyClientConfig,
    final MessageStoreConfig messageStoreConfig
) {
    this.brokerConfig = brokerConfig;
    this.nettyServerConfig = nettyServerConfig;
    this.nettyClientConfig = nettyClientConfig;
    this.messageStoreConfig = messageStoreConfig;
    this.consumerOffsetManager = new ConsumerOffsetManager(this);
    this.topicConfigManager = new TopicConfigManager(this);
    this.pullMessageProcessor = new PullMessageProcessor(this);
    this.pullRequestHoldService = new PullRequestHoldService(this);
    this.messageArrivingListener = new NotifyMessageArrivingListener(this.pullRequestHoldService);
    this.consumerIdsChangeListener = new DefaultConsumerIdsChangeListener(this);
    this.consumerManager = new ConsumerManager(this.consumerIdsChangeListener);
    this.consumerFilterManager = new ConsumerFilterManager(this);
    this.producerManager = new ProducerManager();
    this.clientHousekeepingService = new ClientHousekeepingService(this);
    this.broker2Client = new Broker2Client(this);
    this.subscriptionGroupManager = new SubscriptionGroupManager(this);
    this.brokerOuterAPI = new BrokerOuterAPI(nettyClientConfig);
    this.filterServerManager = new FilterServerManager(this);

    this.slaveSynchronize = new SlaveSynchronize(this);

    this.sendThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getSendThreadPoolQueueCapacity());
    this.pullThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getPullThreadPoolQueueCapacity());
    this.queryThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getQueryThreadPoolQueueCapacity());
    this.clientManagerThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getClientManagerThreadPoolQueueCapacity());
    this.consumerManagerThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getConsumerManagerThreadPoolQueueCapacity());
    this.heartbeatThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getHeartbeatThreadPoolQueueCapacity());
    this.endTransactionThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getEndTransactionPoolQueueCapacity());

    this.brokerStatsManager = new BrokerStatsManager(this.brokerConfig.getBrokerClusterName());
    this.setStoreHost(new InetSocketAddress(this.getBrokerConfig().getBrokerIP1(), this.getNettyServerConfig().getListenPort()));

    this.brokerFastFailure = new BrokerFastFailure(this);
    this.configuration = new Configuration(
        log,
        BrokerPathConfigHelper.getBrokerConfigPath(),
        this.brokerConfig, this.nettyServerConfig, this.nettyClientConfig, this.messageStoreConfig
    );
}
```

接着看initialize()方法，我去这也不少啊，不过还是得一点一点啃

```java
public boolean initialize() throws CloneNotSupportedException {
    // 初始化4个Manager，从名字可以看出这4个Manager的指责
    boolean result = this.topicConfigManager.load();

    result = result && this.consumerOffsetManager.load();
    result = result && this.subscriptionGroupManager.load();
    result = result && this.consumerFilterManager.load();
    // 如果前面4个manager的配置加载成功了，会初始化一个MessageStore，并且用MessageStorePluginContext对MessageStore进行包装，具体初始化了哪些东西我现在也不知道，我们先看整个启动，以后再详细看每个小组件的功能，这里只需要知道初始化了一个MessageStore，这个MessageStore和Broker存储消息是有关系的
    if (result) {
        try {
            this.messageStore =
                new DefaultMessageStore(this.messageStoreConfig, this.brokerStatsManager, this.messageArrivingListener,
                    this.brokerConfig);
            if (messageStoreConfig.isEnableDLegerCommitLog()) {
                DLedgerRoleChangeHandler roleChangeHandler = new DLedgerRoleChangeHandler(this, (DefaultMessageStore) messageStore);
                ((DLedgerCommitLog)((DefaultMessageStore) messageStore).getCommitLog()).getdLedgerServer().getdLedgerLeaderElector().addRoleChangeHandler(roleChangeHandler);
            }
            this.brokerStats = new BrokerStats((DefaultMessageStore) this.messageStore);
            //load plugin
            MessageStorePluginContext context = new MessageStorePluginContext(messageStoreConfig, brokerStatsManager, messageArrivingListener, brokerConfig);
            this.messageStore = MessageStoreFactory.build(context, this.messageStore);
            this.messageStore.getDispatcherList().addFirst(new CommitLogDispatcherCalcBitMap(this.brokerConfig, this.consumerFilterManager));
        } catch (IOException e) {
            result = false;
            log.error("Failed to initialize", e);
        }
    }
		// 如果配置和MessageStore都load成功了，才继续进行
    result = result && this.messageStore.load();

    if (result) {
        // 初始化Netty了
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.clientHousekeepingService);
        NettyServerConfig fastConfig = (NettyServerConfig) this.nettyServerConfig.clone();
        // 又出现了一个端口
        fastConfig.setListenPort(nettyServerConfig.getListenPort() - 2);
        // 有起来一个Netty？
        this.fastRemotingServer = new NettyRemotingServer(fastConfig, this.clientHousekeepingService);
        // 初始化了一堆线程池
        this.sendMessageExecutor = ...;
        this.pullMessageExecutor = ...;

        this.queryMessageExecutor = ...;

        this.clientManageExecutor =...;
      
        this.heartbeatExecutor = ...;

        this.endTransactionExecutor = ...;
        this.consumerManageExecutor = ...;
				// 注册处理器，这个我们可以详细看一下
        this.registerProcessor();

        // 这里启动一些周期执行任务
        this.scheduledExecutorService.scheduleAtFixedRate(...);
        // 更新Broker中Namesrv的地址列表
        if (this.brokerConfig.getNamesrvAddr() != null) {
            this.brokerOuterAPI.updateNameServerAddressList(this.brokerConfig.getNamesrvAddr());
            log.info("Set user specified name server address: {}", this.brokerConfig.getNamesrvAddr());
        } else if (this.brokerConfig.isFetchNamesrvAddrByAddressServer()) {
            // 这里支持从一个地址获取Namesrv的地址列表，就是通过Http#get从一个配置的网站地址获取，不再深入探讨
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

                @Override
                public void run() {
                    try {
                        BrokerController.this.brokerOuterAPI.fetchNameServerAddr();
                    } catch (Throwable e) {
                        log.error("ScheduledTask fetchNameServerAddr exception", e);
                    }
                }
            }, 1000 * 10, 1000 * 60 * 2, TimeUnit.MILLISECONDS);
        }
				// 基于 Raft 协议的 commitlog 存储库 DLedger，可以参考 https://www.infoq.cn/article/7xeJrpDZBa9v*GDZOFS6
        // 默认是不支持DLeger的，所以这里会进入到if里面，关于DLedger后面写到再研究吧
        if (!messageStoreConfig.isEnableDLegerCommitLog()) {
            // 如果是SLAVE，根据条件判断是否周期的更新MasterHaServer的地址
            if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
                if (this.messageStoreConfig.getHaMasterAddress() != null && this.messageStoreConfig.getHaMasterAddress().length() >= 6) {
                    this.messageStore.updateHaMasterAddress(this.messageStoreConfig.getHaMasterAddress());
                    this.updateMasterHAServerAddrPeriodically = false;
                } else {
                    this.updateMasterHAServerAddrPeriodically = true;
                }
            } else {
                // 如果不是SLAVE，周期性打印slave落后于master的diff
                this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            BrokerController.this.printMasterAndSlaveDiff();
                        } catch (Throwable e) {
                            log.error("schedule printMasterAndSlaveDiff error.", e);
                        }
                    }
                }, 1000 * 10, 1000 * 60, TimeUnit.MILLISECONDS);
            }
        }
				// TLS相关的忽略
        if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
            ...
        }
        // 下面这三个方法又是RocketMQ的一些特性的组件初始化，通过方法名字就能知道组件和什么有关，这里不想详细分析，注意这里是通过SPI机制去初始化这3个组件的
        initialTransaction();
        initialAcl();
        initialRpcHooks();
    }
    return result;
}
```

注册了Processor，NettyRemotingServer接收到Request时，对根据请求的类型查找Processor，让查找的Processor来处理这些请求，如果没找到对应的Processor，那么就会使用在NettyRemotingServer注册的DefaultProcessor来处理，不同的server注册的DefaultProcessor时不一样的，比如Broker注册的是AdminBrokerProcessor，Namesrv注册的是DefaultRequestProcessor

```java
public void registerProcessor() {
    /**
     * SendMessageProcessor
     */
    SendMessageProcessor sendProcessor = new SendMessageProcessor(this);
    sendProcessor.registerSendMessageHook(sendMessageHookList);
    sendProcessor.registerConsumeMessageHook(consumeMessageHookList);

    this.remotingServer.registerProcessor(RequestCode.SEND_MESSAGE, sendProcessor, this.sendMessageExecutor);
    this.remotingServer.registerProcessor(RequestCode.SEND_MESSAGE_V2, sendProcessor, this.sendMessageExecutor);
    this.remotingServer.registerProcessor(RequestCode.SEND_BATCH_MESSAGE, sendProcessor, this.sendMessageExecutor);
    this.remotingServer.registerProcessor(RequestCode.CONSUMER_SEND_MSG_BACK, sendProcessor, this.sendMessageExecutor);
    this.fastRemotingServer.registerProcessor(RequestCode.SEND_MESSAGE, sendProcessor, this.sendMessageExecutor);
    this.fastRemotingServer.registerProcessor(RequestCode.SEND_MESSAGE_V2, sendProcessor, this.sendMessageExecutor);
    this.fastRemotingServer.registerProcessor(RequestCode.SEND_BATCH_MESSAGE, sendProcessor, this.sendMessageExecutor);
    this.fastRemotingServer.registerProcessor(RequestCode.CONSUMER_SEND_MSG_BACK, sendProcessor, this.sendMessageExecutor);
    /**
     * PullMessageProcessor
     */
    this.remotingServer.registerProcessor(RequestCode.PULL_MESSAGE, this.pullMessageProcessor, this.pullMessageExecutor);
    this.pullMessageProcessor.registerConsumeMessageHook(consumeMessageHookList);

    /**
     * QueryMessageProcessor
     */
    NettyRequestProcessor queryProcessor = new QueryMessageProcessor(this);
    this.remotingServer.registerProcessor(RequestCode.QUERY_MESSAGE, queryProcessor, this.queryMessageExecutor);
    this.remotingServer.registerProcessor(RequestCode.VIEW_MESSAGE_BY_ID, queryProcessor, this.queryMessageExecutor);

    this.fastRemotingServer.registerProcessor(RequestCode.QUERY_MESSAGE, queryProcessor, this.queryMessageExecutor);
    this.fastRemotingServer.registerProcessor(RequestCode.VIEW_MESSAGE_BY_ID, queryProcessor, this.queryMessageExecutor);

    /**
     * ClientManageProcessor
     */
    ClientManageProcessor clientProcessor = new ClientManageProcessor(this);
    this.remotingServer.registerProcessor(RequestCode.HEART_BEAT, clientProcessor, this.heartbeatExecutor);
    this.remotingServer.registerProcessor(RequestCode.UNREGISTER_CLIENT, clientProcessor, this.clientManageExecutor);
    this.remotingServer.registerProcessor(RequestCode.CHECK_CLIENT_CONFIG, clientProcessor, this.clientManageExecutor);

    this.fastRemotingServer.registerProcessor(RequestCode.HEART_BEAT, clientProcessor, this.heartbeatExecutor);
    this.fastRemotingServer.registerProcessor(RequestCode.UNREGISTER_CLIENT, clientProcessor, this.clientManageExecutor);
    this.fastRemotingServer.registerProcessor(RequestCode.CHECK_CLIENT_CONFIG, clientProcessor, this.clientManageExecutor);

    /**
     * ConsumerManageProcessor
     */
    ConsumerManageProcessor consumerManageProcessor = new ConsumerManageProcessor(this);
    this.remotingServer.registerProcessor(RequestCode.GET_CONSUMER_LIST_BY_GROUP, consumerManageProcessor, this.consumerManageExecutor);
    this.remotingServer.registerProcessor(RequestCode.UPDATE_CONSUMER_OFFSET, consumerManageProcessor, this.consumerManageExecutor);
    this.remotingServer.registerProcessor(RequestCode.QUERY_CONSUMER_OFFSET, consumerManageProcessor, this.consumerManageExecutor);

    this.fastRemotingServer.registerProcessor(RequestCode.GET_CONSUMER_LIST_BY_GROUP, consumerManageProcessor, this.consumerManageExecutor);
    this.fastRemotingServer.registerProcessor(RequestCode.UPDATE_CONSUMER_OFFSET, consumerManageProcessor, this.consumerManageExecutor);
    this.fastRemotingServer.registerProcessor(RequestCode.QUERY_CONSUMER_OFFSET, consumerManageProcessor, this.consumerManageExecutor);

    /**
     * EndTransactionProcessor
     */
    this.remotingServer.registerProcessor(RequestCode.END_TRANSACTION, new EndTransactionProcessor(this), this.endTransactionExecutor);
    this.fastRemotingServer.registerProcessor(RequestCode.END_TRANSACTION, new EndTransactionProcessor(this), this.endTransactionExecutor);

    /**
     * Default
     */
    AdminBrokerProcessor adminProcessor = new AdminBrokerProcessor(this);
    this.remotingServer.registerDefaultProcessor(adminProcessor, this.adminBrokerExecutor);
    this.fastRemotingServer.registerDefaultProcessor(adminProcessor, this.adminBrokerExecutor);
}
```

再看看AdminBrokerProcessor中可以处理那些类型的请求

```java
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx,
    RemotingCommand request) throws RemotingCommandException {
    switch (request.getCode()) {
        case RequestCode.UPDATE_AND_CREATE_TOPIC:
            return this.updateAndCreateTopic(ctx, request);
        case RequestCode.DELETE_TOPIC_IN_BROKER:
            return this.deleteTopic(ctx, request);
        case RequestCode.GET_ALL_TOPIC_CONFIG:
            return this.getAllTopicConfig(ctx, request);
        case RequestCode.UPDATE_BROKER_CONFIG:
            return this.updateBrokerConfig(ctx, request);
        case RequestCode.GET_BROKER_CONFIG:
            return this.getBrokerConfig(ctx, request);
        case RequestCode.SEARCH_OFFSET_BY_TIMESTAMP:
            return this.searchOffsetByTimestamp(ctx, request);
        case RequestCode.GET_MAX_OFFSET:
            return this.getMaxOffset(ctx, request);
        case RequestCode.GET_MIN_OFFSET:
            return this.getMinOffset(ctx, request);
        case RequestCode.GET_EARLIEST_MSG_STORETIME:
            return this.getEarliestMsgStoretime(ctx, request);
        case RequestCode.GET_BROKER_RUNTIME_INFO:
            return this.getBrokerRuntimeInfo(ctx, request);
        case RequestCode.LOCK_BATCH_MQ:
            return this.lockBatchMQ(ctx, request);
        case RequestCode.UNLOCK_BATCH_MQ:
            return this.unlockBatchMQ(ctx, request);
        case RequestCode.UPDATE_AND_CREATE_SUBSCRIPTIONGROUP:
            return this.updateAndCreateSubscriptionGroup(ctx, request);
        case RequestCode.GET_ALL_SUBSCRIPTIONGROUP_CONFIG:
            return this.getAllSubscriptionGroup(ctx, request);
        case RequestCode.DELETE_SUBSCRIPTIONGROUP:
            return this.deleteSubscriptionGroup(ctx, request);
        case RequestCode.GET_TOPIC_STATS_INFO:
            return this.getTopicStatsInfo(ctx, request);
        case RequestCode.GET_CONSUMER_CONNECTION_LIST:
            return this.getConsumerConnectionList(ctx, request);
        case RequestCode.GET_PRODUCER_CONNECTION_LIST:
            return this.getProducerConnectionList(ctx, request);
        case RequestCode.GET_CONSUME_STATS:
            return this.getConsumeStats(ctx, request);
        case RequestCode.GET_ALL_CONSUMER_OFFSET:
            return this.getAllConsumerOffset(ctx, request);
        case RequestCode.GET_ALL_DELAY_OFFSET:
            return this.getAllDelayOffset(ctx, request);
        case RequestCode.INVOKE_BROKER_TO_RESET_OFFSET:
            return this.resetOffset(ctx, request);
        case RequestCode.INVOKE_BROKER_TO_GET_CONSUMER_STATUS:
            return this.getConsumerStatus(ctx, request);
        case RequestCode.QUERY_TOPIC_CONSUME_BY_WHO:
            return this.queryTopicConsumeByWho(ctx, request);
        case RequestCode.REGISTER_FILTER_SERVER:
            return this.registerFilterServer(ctx, request);
        case RequestCode.QUERY_CONSUME_TIME_SPAN:
            return this.queryConsumeTimeSpan(ctx, request);
        case RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_BROKER:
            return this.getSystemTopicListFromBroker(ctx, request);
        case RequestCode.CLEAN_EXPIRED_CONSUMEQUEUE:
            return this.cleanExpiredConsumeQueue();
        case RequestCode.CLEAN_UNUSED_TOPIC:
            return this.cleanUnusedTopic();
        case RequestCode.GET_CONSUMER_RUNNING_INFO:
            return this.getConsumerRunningInfo(ctx, request);
        case RequestCode.QUERY_CORRECTION_OFFSET:
            return this.queryCorrectionOffset(ctx, request);
        case RequestCode.CONSUME_MESSAGE_DIRECTLY:
            return this.consumeMessageDirectly(ctx, request);
        case RequestCode.CLONE_GROUP_OFFSET:
            return this.cloneGroupOffset(ctx, request);
        case RequestCode.VIEW_BROKER_STATS_DATA:
            return ViewBrokerStatsData(ctx, request);
        case RequestCode.GET_BROKER_CONSUME_STATS:
            return fetchAllConsumeStatsInBroker(ctx, request);
        case RequestCode.QUERY_CONSUME_QUEUE:
            return queryConsumeQueue(ctx, request);
        case RequestCode.UPDATE_AND_CREATE_ACL_CONFIG:
            return updateAndCreateAccessConfig(ctx, request);
        case RequestCode.DELETE_ACL_CONFIG:
            return deleteAccessConfig(ctx, request);
        case RequestCode.GET_BROKER_CLUSTER_ACL_INFO:
            return getBrokerAclConfigVersion(ctx, request);
        case RequestCode.UPDATE_GLOBAL_WHITE_ADDRS_CONFIG:
            return updateGlobalWhiteAddrsConfig(ctx, request);
        case RequestCode.RESUME_CHECK_HALF_MESSAGE:
            return resumeCheckHalfMessage(ctx, request);
        default:
            break;
    }

    return null;
}
```

registerProcessor和AdminBrokerProcessor就可以知道Broker可以接受哪些类型的请求了

到这里BrokerController#initialize()方法就算结束了，接下来就是执行BrokerController#start()方法

```java
public void start() throws Exception {
    if (this.messageStore != null) {
        this.messageStore.start();
    }

    if (this.remotingServer != null) {
        this.remotingServer.start();
    }

    if (this.fastRemotingServer != null) {
        this.fastRemotingServer.start();
    }

    if (this.fileWatchService != null) {
        this.fileWatchService.start();
    }

    if (this.brokerOuterAPI != null) {
        this.brokerOuterAPI.start();
    }

    if (this.pullRequestHoldService != null) {
        this.pullRequestHoldService.start();
    }

    if (this.clientHousekeepingService != null) {
        this.clientHousekeepingService.start();
    }

    if (this.filterServerManager != null) {
        this.filterServerManager.start();
    }

    if (!messageStoreConfig.isEnableDLegerCommitLog()) {
        startProcessorByHa(messageStoreConfig.getBrokerRole());
        handleSlaveSynchronize(messageStoreConfig.getBrokerRole());
    }
    // 向Namesrv注册Broker，并且周期的向amesrv注册
    this.registerBrokerAll(true, false, true);

    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
            } catch (Throwable e) {
                log.error("registerBrokerAll Exception", e);
            }
        }
    }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);

    if (this.brokerStatsManager != null) {
        this.brokerStatsManager.start();
    }

    if (this.brokerFastFailure != null) {
        this.brokerFastFailure.start();
    }
}
```

可以看到start()方法其实就是调用Broker中各种service和manager的start()方法，并且向Namesrv周期性的注册自己

到这里，Broker启动的流程基本就完了，但是还有很多坑没有填，因为Broker涉及到的组件比较多，接下来会分一系列文章来分析，这里只是从整体上理解Broker启动的流程