---
title: "RocketMQ Namesrv启动流程"
date: 2020-11-27T22:36:12+08:00
draft: false
original: true
categories: 
  - "中间件"
tags: 
  - "RocketMQ"
---


`NamesrvStartup`启动入口

```java
public static NamesrvController main0(String[] args) {

    try {
        // 创建NamesrvController
        NamesrvController controller = createNamesrvController(args);
        // 启动NamesrvController
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

<!--more-->

创建`NamesrvController`

```java
public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {

    // 根据命令行参数，使用commons-cli命令行工具包解析生成CommandLine对象
    // 在parseCmdLine中，如果命令行中有-h选项，执行打印帮助文档的逻辑，然后退出，不再继续
    Options options = ServerUtil.buildCommandlineOptions(new Options());
    commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());
    if (null == commandLine) {
        System.exit(-1);
        return null;
    }

    // 初始化了2个配置 NamesrvConfig，NettyServerConfig，其中NettyServerConfig监听9876是硬编码的
    // 然后通过命令行参数 -c 指定一个配置文件，然后将配置文件中的内容解析成NamesrvConfig，NettyServerConfig的配置
    // 设置NamesrvConfig，NettyServerConfig的逻辑是看类中的set方法，如果set方法后的名字和配置文件中的key匹配，就会设置对应的值
    final NamesrvConfig namesrvConfig = new NamesrvConfig();
    final NettyServerConfig nettyServerConfig = new NettyServerConfig();
    nettyServerConfig.setListenPort(9876);
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

    //如果指定了 -p 选项，会在控制台打印配置信息，然后退出，不再继续执行
    if (commandLine.hasOption('p')) {
        InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
        MixAll.printObjectProperties(console, namesrvConfig);
        MixAll.printObjectProperties(console, nettyServerConfig);
        System.exit(0);
    }

    //将启动命令行的参数配置设置到NamesrvConfig中
    MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);

    // 检查必须设置RocketMQHome
    // 在NamesrvConfig中，可以看到使用系统属性rocketmq.home.dir，环境变量ROCKETMQ_HOME和前面的-c指定的配置文件设置RocketMQHome
    // 在mqnamesrv启动脚本中会自定探测RockerMQ并export ROCKETMQ_HOME
    if (null == namesrvConfig.getRocketmqHome()) {
        System.out.printf("Please set the %s variable in your environment to match the location of the RocketMQ installation%n", MixAll.ROCKETMQ_HOME_ENV);
        System.exit(-2);
    }

    // 加载logback.xml
    LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
    JoranConfigurator configurator = new JoranConfigurator();
    configurator.setContext(lc);
    lc.reset();
    configurator.doConfigure(namesrvConfig.getRocketmqHome() + "/conf/logback_namesrv.xml");

    log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);

    // 使用logback打印NamesrvConfig，NettyServerConfig配置信息
    MixAll.printObjectProperties(log, namesrvConfig);
    MixAll.printObjectProperties(log, nettyServerConfig);

    final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

    // 最后还把-c指定的文件的配置在保存到Configruation中
    controller.getConfiguration().registerConfig(properties);

    return controller;
}
```

可以看到`NamesrvStartup`只是一个启动类，主要逻辑都在处理命令行和配置，主要功能都是在`NamesrvController`中，而且我们可以看到，在处理处理配置的时候，真的是对配置文件进行反复鞭尸呀

首先通过-c指定配置文件，使用MixAll.properties2Object将配置设置到NamesrvConfig，NettyServerConfig

```java
MixAll.properties2Object(properties, namesrvConfig);
MixAll.properties2Object(properties, nettyServerConfig);
```

然后通过命令行参数设置到NamesrvConfig

```java
MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);
```

然后初始化NamesrvController，NamesrvController中会初始化一个Configuration类，Configuration类中又会把NamesrvConfig，NettyServerConfig都merge到allConfigs中

```java
public NamesrvController(NamesrvConfig namesrvConfig, NettyServerConfig nettyServerConfig) {
    this.namesrvConfig = namesrvConfig;
    this.nettyServerConfig = nettyServerConfig;
    this.kvConfigManager = new KVConfigManager(this);
    this.routeInfoManager = new RouteInfoManager();
    this.brokerHousekeepingService = new BrokerHousekeepingService(this);
    // 初始化Configuration
    this.configuration = new Configuration(
        log,
        this.namesrvConfig, this.nettyServerConfig
    );
    this.configuration.setStorePathFromConfig(this.namesrvConfig, "configStorePath");
}
```

```java
public Configuration(InternalLogger log, Object... configObjects) {
    this.log = log;
    if (configObjects == null || configObjects.length == 0) {
        return;
    }
    // 将NamesrvConfig，NettyServerConfig注册
    for (Object configObject : configObjects) {
        registerConfig(configObject);
    }
}
```

```java
public Configuration registerConfig(Object configObject) {
    try {
        readWriteLock.writeLock().lockInterruptibly();

        try {

            Properties registerProps = MixAll.object2Properties(configObject);
						// 将NamesrvConfig，NettyServerConfig合并到allConfigs
            merge(registerProps, this.allConfigs);

            configObjectList.add(configObject);
        } finally {
            readWriteLock.writeLock().unlock();
        }
    } catch (InterruptedException e) {
        log.error("registerConfig lock error");
    }
    return this;
}
```

最后还要把-c指定的配置文件和allConfigs进行合并

```java
// 最后还把-c指定的文件的配置在保存到Configruation中
controller.getConfiguration().registerConfig(properties);
```

可以看到-c指定的配置文件读取进来后被拆分为NamesrvConfig，NettyServerConfig，然后又和Configuration中的allConfigs合并，最后还要再合并一次，你说这个-c指定的配置文件惨不惨

`NamesrvController`初始化完成后，就调用start(controller)，才真正的开始

```java
// 启动NamesrvController
start(controller);
```

在`start(controller)`方法中最关键的就是下面2个方法

```java
controller.initialize();
controller.start();
```

`NamesrvController`初始化

```java
public boolean initialize() {
    
    // 从NamesrvConfig#KvConfigPath指定的文件中反序列化数据到KVConfigManager#configTable中
    this.kvConfigManager.load();
    // 启动网络通信的Netty服务
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

    // 初始化一下负责处理Netty网络交互数据的线程池，
    this.remotingExecutor =
        Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

    // 注册一个默认负责处理Netty网络交互数据的DefaultRequestProcessor，这个Processor会使用remotingExecutor执行
    // *划重点，后面这里会再次提到*
    this.registerProcessor();

    // 每10s扫描一下失效的Broker
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 5, 10, TimeUnit.SECONDS);

    // 每10min打印一下前面被反复蹂躏的配置
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.kvConfigManager.printAllPeriodically();
        }
    }, 1, 10, TimeUnit.MINUTES);


    // 设置TLS，这块不太了解，所以省略了，以后用空了再研究一下TLS吧
    if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
       ...
    }

    return true;
}
```

启动`NamesrvController`

```java
public void start() throws Exception {
    this.remotingServer.start();

    if (this.fileWatchService != null) {
        this.fileWatchService.start();
    }
}
```

可以看到逻辑还算比较清晰，关键功能在KVConfigManager，RouteInfoManager和NettyRemotingServer实现

我们先来看看NettyRemotingServer

```java
public NettyRemotingServer(final NettyServerConfig nettyServerConfig,
    final ChannelEventListener channelEventListener) {
    // 初始化2个Semaphore，一个是one-way请求的并发数，一个是asynchronous请求的并发数，可以简单理解成对2种请求做了限流，至于什么是one-way请求，什么是asynchronous请求，分析到了再说吧
    super(nettyServerConfig.getServerOnewaySemaphoreValue(), nettyServerConfig.getServerAsyncSemaphoreValue());
    this.serverBootstrap = new ServerBootstrap();
    this.nettyServerConfig = nettyServerConfig;
    this.channelEventListener = channelEventListener;

    int publicThreadNums = nettyServerConfig.getServerCallbackExecutorThreads();
    if (publicThreadNums <= 0) {
        publicThreadNums = 4;
    }
		
  	 // 初始化一个公用的线程池，什么情况下用这个公用的线程池？看后面的分析
    this.publicExecutor = Executors.newFixedThreadPool(publicThreadNums, new ThreadFactory() {
        private AtomicInteger threadIndex = new AtomicInteger(0);

        @Override
        public Thread newThread(Runnable r) {
            return new Thread(r, "NettyServerPublicExecutor_" + this.threadIndex.incrementAndGet());
        }
    });

    // 下面这个就是判断系统是否支持epoll来初始化相应的EventLoopGroup，如果不是很了解Netty的同学可以先去学学Netty
    if (useEpoll()) {
        this.eventLoopGroupBoss = new EpollEventLoopGroup();

        this.eventLoopGroupSelector = new EpollEventLoopGroup();
    } else {
        this.eventLoopGroupBoss = new NioEventLoopGroup();

        this.eventLoopGroupSelector = new NioEventLoopGroup();
    }
		// 这个也是TLS相关的忽略分析
    loadSslContext();
}
```

这里可以看一下怎么判断系统是否支持epoll的

```java
private static boolean isLinuxPlatform = false;
private static boolean isWindowsPlatform = false;

static {
    if (OS_NAME != null && OS_NAME.toLowerCase().contains("linux")) {
        isLinuxPlatform = true;
    }

    if (OS_NAME != null && OS_NAME.toLowerCase().contains("windows")) {
        isWindowsPlatform = true;
    }
}

-----

private boolean useEpoll() {
    return RemotingUtil.isLinuxPlatform()
        && nettyServerConfig.isUseEpollNativeSelector()
        && Epoll.isAvailable();
}
```

还记得在`NamesrvController`初始化时注册一个默认负责处理Netty网络交互数据的DefaultRequestProcessor，DefaultRequestProcessor使用了一个专用的remotingExecutor线程池，我们也可以注册其他的Processor，如果我们注册Processor时没有指定线程池就会使用公共的线程池publicExecutor

```java
 public void registerProcessor(int requestCode, NettyRequestProcessor processor, ExecutorService executor) {
    ExecutorService executorThis = executor;
    if (null == executor) {
        executorThis = this.publicExecutor;
    }

    Pair<NettyRequestProcessor, ExecutorService> pair = new Pair<NettyRequestProcessor, ExecutorService>(processor, executorThis);
    this.processorTable.put(requestCode, pair);
}
```

上面只是将Netty的EventLoopGroup进行了初始化，却没有真正的启动Netty，真正的启动还得调用remotingServer.start();

```java
public void start() throws Exception {
    this.remotingServer.start();
		// fileWatchService和TLS有关，大概就是会监听TLS相关文件的改变，也不仔细分析了
    if (this.fileWatchService != null) {
        this.fileWatchService.start();
    }
}
```

接下来看看NettyRemotingServer的start()方法干了啥

```java
public void start() {
    // 初始化一个线程池，用于执行共享的Handler
    this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(
        nettyServerConfig.getServerWorkerThreads(),
        new ThreadFactory() {

            private AtomicInteger threadIndex = new AtomicInteger(0);

            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, "NettyServerCodecThread_" + this.threadIndex.incrementAndGet());
            }
        });
		// 初始化一些共享的Handler，HandshakeHandler，NettyEncoder，NettyConnectManageHandler，NettyServerHandler
    prepareSharableHandlers();
		// 后面就是一些Netty的设置，具体看Netty
    ServerBootstrap childHandler =
        this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
            .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
            .option(ChannelOption.SO_BACKLOG, 1024)
            .option(ChannelOption.SO_REUSEADDR, true)
            .option(ChannelOption.SO_KEEPALIVE, false)
            .childOption(ChannelOption.TCP_NODELAY, true)
            .childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSndBufSize())
            .childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketRcvBufSize())
            .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))
      			// 我们只需要关心这里设置了哪些handler
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline()
                        .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME, handshakeHandler)
                        .addLast(defaultEventExecutorGroup,
                            encoder,
                            new NettyDecoder(),
                            new IdleStateHandler(0, 0, 			nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                            connectionManageHandler,
                            serverHandler
                        );
                }
            });

    if (nettyServerConfig.isServerPooledByteBufAllocatorEnable()) {
        childHandler.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
    }

    try {
        ChannelFuture sync = this.serverBootstrap.bind().sync();
        InetSocketAddress addr = (InetSocketAddress) sync.channel().localAddress();
        this.port = addr.getPort();
    } catch (InterruptedException e1) {
        throw new RuntimeException("this.serverBootstrap.bind().sync() InterruptedException", e1);
    }
		// 这里有一个channelEventListener，在NamesrvController中channelEventListener就是BrokerHousekeepingService，BrokerHousekeepingService负责在broker断开连接的时候，移除RouteInfoManager中的路由信息
    // NettyEventExecutor会维护一个NettyEvent的队列，NettyConnectManageHandler会向NettyEvent的队列中添加Event，然后由channelEventListener进行消费
    if (this.channelEventListener != null) {
        this.nettyEventExecutor.start();
    }
		// 定时扫描responseTable，执行超时请求的callback
    // 这里有2个疑问，是谁向responseTable中put数据？为什么这里只执行超时请求的callback，正常结束的请求在哪处理的？
    this.timer.scheduleAtFixedRate(new TimerTask() {

        @Override
        public void run() {
            try {
                NettyRemotingServer.this.scanResponseTable();
            } catch (Throwable e) {
                log.error("scanResponseTable exception", e);
            }
        }
    }, 1000 * 3, 1000);
}
```

到这里RocketMQ Namesrv的启动流程就结束了，下一篇在来分析具体怎么处理请求数据的吧