---
title: "RocketMQ如何实现请求与响应"
date: 2020-11-27T22:35:02+08:00
draft: false
original: true
categories: 
  - "中间件"
tags: 
  - "RocketMQ"
---


前面我们分析了RocketMQ通信的数据格式，那么RocketMQ怎么将数据发送出去呢？

我们假设已经完成了对RemotingCommand的初始化，这篇文章只分析发送数据部分

在`RemotingClient`和`RemotingServer`中都定义了invoke方法，我们有理由相信client和server端发送请求会存在不同

首先看一下`RemotingClient`中的定义

```java
public interface RemotingClient extends RemotingService {

    RemotingCommand invokeSync(final String addr, final RemotingCommand request,
        final long timeoutMillis) throws InterruptedException, RemotingConnectException,
        RemotingSendRequestException, RemotingTimeoutException;

    void invokeAsync(final String addr, final RemotingCommand request, final long timeoutMillis,
        final InvokeCallback invokeCallback) throws InterruptedException, RemotingConnectException,
        RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException;

    void invokeOneway(final String addr, final RemotingCommand request, final long timeoutMillis)
        throws InterruptedException, RemotingConnectException, RemotingTooMuchRequestException,
        RemotingTimeoutException, RemotingSendRequestException;

}
```

<!--more-->

再看一下`RemotingServer`中的定义

```java
public interface RemotingServer extends RemotingService {

    RemotingCommand invokeSync(final Channel channel, final RemotingCommand request,
        final long timeoutMillis) throws InterruptedException, RemotingSendRequestException,
        RemotingTimeoutException;

    void invokeAsync(final Channel channel, final RemotingCommand request, final long timeoutMillis,
        final InvokeCallback invokeCallback) throws InterruptedException,
        RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException;

    void invokeOneway(final Channel channel, final RemotingCommand request, final long timeoutMillis)
        throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException,
        RemotingSendRequestException;

}
```

可以看到，正好和RocketMQ官方文档中支持的3种调用方式相对应，即同步调用，异步调用和oneway调用

再者还能发现，client和server中定义的3种调用方法不同就只在第一个参数，`RemotingClient`的参数是`addr`，而`RemotingServer`的参数是`channel`

接下来在看看`RemotingClient`的实现类`NettyRemotingClient`和`RemotingServer`的实现类`NettyRemotingServer`的区别

`NettyRemotingServer`的实现很简单，都是直接调用抽象类`NettyRemotingAbstract`的Impl方法

```java
@Override
public RemotingCommand invokeSync(final Channel channel, final RemotingCommand request, final long timeoutMillis)
    throws InterruptedException, RemotingSendRequestException, RemotingTimeoutException {
    return this.invokeSyncImpl(channel, request, timeoutMillis);
}

@Override
public void invokeAsync(Channel channel, RemotingCommand request, long timeoutMillis, InvokeCallback invokeCallback)
    throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
    this.invokeAsyncImpl(channel, request, timeoutMillis, invokeCallback);
}

@Override
public void invokeOneway(Channel channel, RemotingCommand request, long timeoutMillis) throws InterruptedException,
    RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
    this.invokeOnewayImpl(channel, request, timeoutMillis);
}
```

再看看`NettyRemotingClient`的实现，可以发现比`NettyRemotingServer`稍微复杂了一点，主要区别就是

1. 根据`addr`获取或创建响应的连接，得到channel对象
2. 支持RpcHook

不算是`NettyRemotingClient`还是`NettyRemotingServer`最后都是调用抽象类`NettyRemotingAbstract`的Impl方法，那么我们就来看一下这些Impl方法是如何实现的

```java
@Override
public RemotingCommand invokeSync(String addr, final RemotingCommand request, long timeoutMillis)
    throws InterruptedException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException {
    long beginStartTime = System.currentTimeMillis();
    final Channel channel = this.getAndCreateChannel(addr);
    if (channel != null && channel.isActive()) {
        try {
            doBeforeRpcHooks(addr, request);
            long costTime = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTime) {
                throw new RemotingTimeoutException("invokeSync call timeout");
            }
            RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis - costTime);
            doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(channel), request, response);
            return response;
        } catch (RemotingSendRequestException e) {
            log.warn("invokeSync: send request exception, so close the channel[{}]", addr);
            this.closeChannel(addr, channel);
            throw e;
        } catch (RemotingTimeoutException e) {
            if (nettyClientConfig.isClientCloseSocketIfTimeout()) {
                this.closeChannel(addr, channel);
                log.warn("invokeSync: close socket because of timeout, {}ms, {}", timeoutMillis, addr);
            }
            log.warn("invokeSync: wait response timeout exception, the channel[{}]", addr);
            throw e;
        }
    } else {
        this.closeChannel(addr, channel);
        throw new RemotingConnectException(addr);
    }
}

@Override
public void invokeAsync(String addr, RemotingCommand request, long timeoutMillis, InvokeCallback invokeCallback)
    throws InterruptedException, RemotingConnectException, RemotingTooMuchRequestException, RemotingTimeoutException,
    RemotingSendRequestException {
    long beginStartTime = System.currentTimeMillis();
    final Channel channel = this.getAndCreateChannel(addr);
    if (channel != null && channel.isActive()) {
        try {
            doBeforeRpcHooks(addr, request);
            long costTime = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTime) {
                throw new RemotingTooMuchRequestException("invokeAsync call timeout");
            }
            this.invokeAsyncImpl(channel, request, timeoutMillis - costTime, invokeCallback);
        } catch (RemotingSendRequestException e) {
            log.warn("invokeAsync: send request exception, so close the channel[{}]", addr);
            this.closeChannel(addr, channel);
            throw e;
        }
    } else {
        this.closeChannel(addr, channel);
        throw new RemotingConnectException(addr);
    }
}

@Override
public void invokeOneway(String addr, RemotingCommand request, long timeoutMillis) throws InterruptedException,
    RemotingConnectException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
    final Channel channel = this.getAndCreateChannel(addr);
    if (channel != null && channel.isActive()) {
        try {
            doBeforeRpcHooks(addr, request);
            this.invokeOnewayImpl(channel, request, timeoutMillis);
        } catch (RemotingSendRequestException e) {
            log.warn("invokeOneway: send request exception, so close the channel[{}]", addr);
            this.closeChannel(addr, channel);
            throw e;
        }
    } else {
        this.closeChannel(addr, channel);
        throw new RemotingConnectException(addr);
    }
}
```

先看看同步调用的实现

```java
public RemotingCommand invokeSyncImpl(final Channel channel, final RemotingCommand request,
    final long timeoutMillis)
    throws InterruptedException, RemotingSendRequestException, RemotingTimeoutException {
    // opaque可以理解是RemotingCommand的编号，在初始化RemotingCommand对象的时候通过AtomicInteger自增得到的
    final int opaque = request.getOpaque();

    try {
        // 初始化ResponseFuture，这里可以和JDK中的Future进行对比，其实就是对将要返回数据的一种表示
        final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis, null, null);  
        // 然后会将这个ResponseFuture放入到responseTable这个ConcurrentHashMap中
        this.responseTable.put(opaque, responseFuture);
        final SocketAddress addr = channel.remoteAddress();
        // 发送RemotingCommand
        channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture f) throws Exception {
                if (f.isSuccess()) {
                    responseFuture.setSendRequestOK(true);
                    return;
                } else {
                    responseFuture.setSendRequestOK(false);
                }
								// 如果发送失败，设置Cause并把Response设置为null
                responseTable.remove(opaque);
                responseFuture.setCause(f.cause());
                responseFuture.putResponse(null);
                log.warn("send a request command to channel <" + addr + "> failed.");
            }
        });
				// 由于是同步调用，Netty所有的操作都不是同步操作，所以这里需要根据设置的超时时间做一定时间等待来实现同步的效果
        RemotingCommand responseCommand = responseFuture.waitResponse(timeoutMillis);
        // 如果等待时间到了，会返回响应数据
        if (null == responseCommand) {
            // 如果responseCommand == null，并且请求正常发送了，那么就是超时了
            if (responseFuture.isSendRequestOK()) {
                throw new RemotingTimeoutException(RemotingHelper.parseSocketAddressAddr(addr), timeoutMillis,
                    responseFuture.getCause());
            } else { // 如果发送请求失败抛出RemotingSendRequestException异常
                throw new RemotingSendRequestException(RemotingHelper.parseSocketAddressAddr(addr), responseFuture.getCause());
            }
        }
				// 获取到了响应结果返回
        return responseCommand;
    } finally {
        // 移除responseTable中对应的这个请求
        this.responseTable.remove(opaque);
    }
}
```

请求流程都在注释中写了，这里说一下RocketMQ怎么实现同步请求，并且实现了

由于Netty的 channel.writeAndFlush方法不是同步的，所以必须通过其他方式实现同步，RocketMQ是通过`CountDownLatch`实现的超时等待以实现同步调用，代码体现在`responseFuture.waitResponse(timeoutMillis)`

```java
public RemotingCommand waitResponse(final long timeoutMillis) throws InterruptedException {
    this.countDownLatch.await(timeoutMillis, TimeUnit.MILLISECONDS);
    return this.responseCommand;
}
```

这个countDownLatch在初始化ResponseFuture时就设置了，并且是new CountDownLatch(1)

```java
public class ResponseFuture {
    private final int opaque;
    private final Channel processChannel;
    private final long timeoutMillis;
    private final InvokeCallback invokeCallback;
    private final long beginTimestamp = System.currentTimeMillis();
    private final CountDownLatch countDownLatch = new CountDownLatch(1);
}
```

 在调用`responseFuture.putResponse(response)`时会调用countDownLatch\#countDown()方法

```java
public void putResponse(final RemotingCommand responseCommand) {
    this.responseCommand = responseCommand;
    this.countDownLatch.countDown();
}
```

在上面同步调用实现中，我们只看到发送请求失败的时候调用了`responseFuture.putResponse(null)`，这样如果发送请求失败，执行到`responseFuture.waitResponse(timeoutMillis)`时就不会等待了，而是直接返回了

那么正常获取到了响应数据的流程怎么执行的呢？

首先如果请求发送成功，但响应数据还没有返回时，同步调用会阻塞在`responseFuture.waitResponse(timeoutMillis)`，直到响应数据返回调用了`responseFuture.putResponse(response)`或者超时时间到了自己醒过来

那么正常获取到数据的请求时谁调用了``responseFuture.putResponse(response)`，是怎么调用到的呢？

这个就得回到前面介绍Namesrv启动流程时，向ChannelPipeline中注册了一堆Handler，这里需要说明不光是Namesrv，只要是涉及到使用Netty进行通信的组件都会向ChannelPipeline中注册了一堆Handler，只是不同的组件注册的Handler实现逻辑不同

那么我们看一下在`NettyRemotingClient`和`NettyRemotingServer`中注册的handler

先看一下`NettyRemotingServer`

```java
.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline()
            .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME, handshakeHandler)
            .addLast(defaultEventExecutorGroup,
                encoder,
                new NettyDecoder(),
                new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                connectionManageHandler,
                serverHandler
            );
    }
});
```

再看一下`NettyRemotingClient`

```java
pipeline.addLast(
    defaultEventExecutorGroup,
    new NettyEncoder(),
    new NettyDecoder(),
    new IdleStateHandler(0, 0, nettyClientConfig.getClientChannelMaxIdleTimeSeconds()),
    new NettyConnectManageHandler(),
    new NettyClientHandler());
```

可以发现`NettyRemotingClient`中注册了`NettyClientHandler`，`NettyRemotingServer`中注册了`NettyServerHandler`

那么`responseFuture.putResponse(response)`肯定就是在这2个Handler中调用的，我们一个一个来看

先看`NettyClientHandler`，直接调用的是抽象类`NettyRemotingAbstract`的`processMessageReceived(ctx, msg)`方法

```java
class NettyClientHandler extends SimpleChannelInboundHandler<RemotingCommand> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
        processMessageReceived(ctx, msg);
    }
}
```

再看一下`NettyServerHandler`，也是直接调用的是抽象类`NettyRemotingAbstract`的`processMessageReceived(ctx, msg)`方法

```java
@ChannelHandler.Sharable
class NettyServerHandler extends SimpleChannelInboundHandler<RemotingCommand> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
        processMessageReceived(ctx, msg);
    }
}
```

看看这TM写得什么代码，2个Handler实现一样，你写成2个类干啥，处理逻辑还不是都到`NettyRemotingAbstract`实现的

行吧不吐槽了，继续看`processMessageReceived(ctx, msg)`方法吧

```java
public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
    final RemotingCommand cmd = msg;
    if (cmd != null) {
        switch (cmd.getType()) {
            case REQUEST_COMMAND:
                processRequestCommand(ctx, cmd);
                break;
            case RESPONSE_COMMAND:
                processResponseCommand(ctx, cmd);
                break;
            default:
                break;
        }
    }
}
```

可以看到，根据RemotingCommand#getType()方法，判断是请求还是响应分别调用不同的方法进行处理

这里需要说明一下，Client不光只会处理响应数据，Server只会处理请求响应数据，不管是Client还是Server，都可能处理请求或响应，因为TCP是双工的，双方都有可能向对方发送请求

好了扯了这么远，现在回到上面同步消息发送出去了，对方会返回给我们数据，我们收到响应时应该会到`processResponseCommand(ctx, cmd)`中，需要提醒一下，这个时候`cmd`已经是对方返回的**响应数据**了，而不是请求数据了，这个可能会因为cmd这个命名混淆

```java
public void processResponseCommand(ChannelHandlerContext ctx, RemotingCommand cmd) {
    final int opaque = cmd.getOpaque();
    final ResponseFuture responseFuture = responseTable.get(opaque);
    if (responseFuture != null) {
        responseFuture.setResponseCommand(cmd);

        responseTable.remove(opaque);

        if (responseFuture.getInvokeCallback() != null) {
            executeInvokeCallback(responseFuture);
        } else {
            responseFuture.putResponse(cmd);
            responseFuture.release();
        }
    } else {
        log.warn("receive response, but not matched any request, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()));
        log.warn(cmd.toString());
    }
}
```

看到了吧，一进来就是从responseTable中根据opaque获取ResponseFuture，然后就是我们前面说的会调用`responseFuture.setResponseCommand(cmd)`方法，然后从responseTable中根据opaque移除这个ResponseFuture，如果有回调方法的话，会执行这个回调方法，这个在异步调用的时候再说吧

其实执行了`responseFuture.setResponseCommand(cmd)`方法，将countDownLatch#countDown()之后，阻塞在`responseFuture.waitResponse(timeoutMillis)`的已经往下执行了。可以发现如果请求正常的话，`responseTable.remove(opaque)`会被执行2次

到这里，同步调用就算是结束了，知道了同步调用的套路，我们再看其他的调用应该就相对轻松了

```java
public void invokeAsyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis,
    final InvokeCallback invokeCallback)
    throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
    long beginStartTime = System.currentTimeMillis();
    final int opaque = request.getOpaque();
    // 限流
    boolean acquired = this.semaphoreAsync.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
    if (acquired) {
        final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreAsync);
        long costTime = System.currentTimeMillis() - beginStartTime;
        if (timeoutMillis < costTime) {
            once.release();
            throw new RemotingTimeoutException("invokeAsyncImpl call timeout");
        }

        final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis - costTime, invokeCallback, once);
        // 注意这里后面会提到，大家留意一下
        this.responseTable.put(opaque, responseFuture);
        try {
            channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    if (f.isSuccess()) {
                        responseFuture.setSendRequestOK(true);
                        return;
                    }
                    requestFail(opaque);
                    log.warn("send a request command to channel <{}> failed.", RemotingHelper.parseChannelRemoteAddr(channel));
                }
            });
        } catch (Exception e) {
            responseFuture.release();
            log.warn("send a request command to channel <" + RemotingHelper.parseChannelRemoteAddr(channel) + "> Exception", e);
            throw new RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e);
        }
    } else {
        if (timeoutMillis <= 0) {
            throw new RemotingTooMuchRequestException("invokeAsyncImpl invoke too fast");
        } else {
            String info =
                String.format("invokeAsyncImpl tryAcquire semaphore timeout, %dms, waiting thread nums: %d semaphoreAsyncValue: %d",
                    timeoutMillis,
                    this.semaphoreAsync.getQueueLength(),
                    this.semaphoreAsync.availablePermits()
                );
            log.warn(info);
            throw new RemotingTimeoutException(info);
        }
    }
}
```

分析完同步调用流程后再看一步调用的代码就比较容易理解了所以就没写注释了，大家可以自己看看，如果你仔细看了同步调用的流程分析，这段代码应该很简单了

这里分析一下和同步调用不同的地方

1. 异步调用有限流的逻辑
2. 异步调用没有同步调用的超时机制
3. 异步调用是通过回调处理返回值的

限流的逻辑很简单，通过信号量`Semaphore`进行控制，这里就不详细分析了，不了解的同学可以看一下JUC包里面的工具类，这里面可都是并发编程的好东西

2，3点其实可以合在一起，但是我还是把他们分开了，我认为这2个还是不一样的东西，先来看看回调，在同步调用的时候我们提到但是没有详细分析，现在我们仔细看看

```java
public void processResponseCommand(ChannelHandlerContext ctx, RemotingCommand cmd) {
    final int opaque = cmd.getOpaque();
    final ResponseFuture responseFuture = responseTable.get(opaque);
    if (responseFuture != null) {
        responseFuture.setResponseCommand(cmd);

        responseTable.remove(opaque);

        if (responseFuture.getInvokeCallback() != null) {
            // 执行会掉
            executeInvokeCallback(responseFuture);
        } else {
            responseFuture.putResponse(cmd);
            responseFuture.release();
        }
    } else {
        log.warn("receive response, but not matched any request, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()));
        log.warn(cmd.toString());
    }
}
```

上面是同步，异步都会走到的逻辑，只不过同步调用responseFuture.getInvokeCallback() == null，而异步调用responseFuture.getInvokeCallback() != null

```java
private void executeInvokeCallback(final ResponseFuture responseFuture) {
    boolean runInThisThread = false;
    ExecutorService executor = this.getCallbackExecutor();
    if (executor != null) {
        try {
            executor.submit(new Runnable() {
                @Override
                public void run() {
                    try {
                        responseFuture.executeInvokeCallback();
                    } catch (Throwable e) {
                        log.warn("execute callback in executor exception, and callback throw", e);
                    } finally {
                        responseFuture.release();
                    }
                }
            });
        } catch (Exception e) {
            runInThisThread = true;
            log.warn("execute callback in executor exception, maybe executor busy", e);
        }
    } else {
        runInThisThread = true;
    }

    if (runInThisThread) {
        try {
            responseFuture.executeInvokeCallback();
        } catch (Throwable e) {
            log.warn("executeInvokeCallback Exception", e);
        } finally {
            responseFuture.release();
        }
    }
}
```

这段代码写这么多，其实就是获取执行回调的线程池，如果有线程池就使用线程池执行回调，如果没有就在当前线程执行回调

```java
public void executeInvokeCallback() {
    if (invokeCallback != null) {
        if (this.executeCallbackOnlyOnce.compareAndSet(false, true)) {
            invokeCallback.operationComplete(this);
        }
    }
}
```

这段代码很简单，就是直接执行传入的回调函数执行

到这里异步就算完了？怎么可能，为啥我上面会把2，3分成2点，不合在一起呢？这就来解释，刚才我们分析回调的时候都是在正常请求，有请求响应的时候，那么如果请求发出去了，对方没有返回数据，但是我们已经把ResponseFuture放到responseTable中了，不记得了向上捯饬捯饬`invokeAsyncImpl`方法中我标记了一下

```java
responseTable.put(opaque, responseFuture);
```

这个时候怎么办？

这又要回到上一篇Namesrv启动流程文章中，在NettyRemotingServer#start()方法的最后，启动了一个Timer，每隔1s会去执行scanResponseTable()方法，这里只是拿NettyRemotingServer说明，其实NettyRemotingClient#start()方法中也有这段逻辑，接下来我们看看scanResponseTable()这个方法做了什么

```java
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
```

逻辑很简单就是将responseTable中超时的ResponseFuture移除，并且执行这些超时ResponseFuture的回调

```java
public void scanResponseTable() {
    final List<ResponseFuture> rfList = new LinkedList<ResponseFuture>();
    Iterator<Entry<Integer, ResponseFuture>> it = this.responseTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<Integer, ResponseFuture> next = it.next();
        ResponseFuture rep = next.getValue();

        if ((rep.getBeginTimestamp() + rep.getTimeoutMillis() + 1000) <= System.currentTimeMillis()) {
            rep.release();
            it.remove();
            rfList.add(rep);
            log.warn("remove timeout request, " + rep);
        }
    }

    for (ResponseFuture rf : rfList) {
        try {
            executeInvokeCallback(rf);
        } catch (Throwable e) {
            log.warn("scanResponseTable, operationComplete Exception", e);
        }
    }
}
```

好，到这里才算讲一步调用的流程走完了

接下来是Oneway调用，咋一看都没听过，同步和异步至少通过，这个Oneway是个什么高级玩意儿，不要慌，看一下就晓得了

```java
public void invokeOnewayImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis)
    throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
    request.markOnewayRPC();
    boolean acquired = this.semaphoreOneway.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
    if (acquired) {
        final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreOneway);
        try {
            channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    once.release();
                    if (!f.isSuccess()) {
                        log.warn("send a request command to channel <" + channel.remoteAddress() + "> failed.");
                    }
                }
            });
        } catch (Exception e) {
            once.release();
            log.warn("write send a request command to channel <" + channel.remoteAddress() + "> failed.");
            throw new RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e);
        }
    } else {
        if (timeoutMillis <= 0) {
            throw new RemotingTooMuchRequestException("invokeOnewayImpl invoke too fast");
        } else {
            String info = String.format(
                "invokeOnewayImpl tryAcquire semaphore timeout, %dms, waiting thread nums: %d semaphoreAsyncValue: %d",
                timeoutMillis,
                this.semaphoreOneway.getQueueLength(),
                this.semaphoreOneway.availablePermits()
            );
            log.warn(info);
            throw new RemotingTimeoutException(info);
        }
    }
}
```

有了同步和异步调用的分析流程，再一看这个oneway，原来就是有请求没有响应的情况呀，是不是更简单了

和异步调用一样，oneway调用也有限流控制，然后发送请求，完成，即不出处理响应也没有同步方法的超时控制

### 总结

最后还是假装总结一下，根据上面的分析，同步调用和异步调用都需要记录请求和响应之间的对应关系，所有每个请求都会有opaque，这就相当于请求的id，然后通过responseTable来保存，等响应数据回来的时候，通过这个id找到对应的请求

同步不像异步那样，通过回调来处理结果，同步必须等待响应结果回来，所以必须有个超时机制，RocketMQ这里是通过CountDownLatch来实现的，而异步是通过回调来处理结果，所以对超时机制不太关注，但是因为通过回调处理响应结果，会导致responseTable中没有响应的请求一直驻留在内存，所以需要一个Timer定时去清理

其实，底层通信在分布式系统中都会有，大家可以再想想RPC框架，是不是也和这个差不多，可能比RocketMQ这个通信更加复杂，比如RPC框架还需要实现异步调用的超时机制，而RocketMQ异步调用是没有严谨的异步超时控制的，以上