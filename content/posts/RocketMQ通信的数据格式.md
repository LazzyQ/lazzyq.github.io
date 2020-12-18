---
title: "RocketMQ通信的数据格式"
date: 2020-11-27T22:17:01+08:00
draft: false
categories: 
  - 中间件
tags: 
  - RocketMQ
  - 消息队列
---


直接上图，下图就是RocketMQ的数据格式

![RocketMQ通信的数据格式](/RocketMQ通信的数据格式/image-20190923182812710.png)

* length: 表示序列化类型+header length + header data + body data的字节长度
* 序列化类型: 在RockerMQ源码的SerializeType枚举中可以看到，支持只JSON和RocketMQ自定义的2种格式
* Header length: 表示header data部分的字节长度
* header data: header的数据
* body length: body的数据

RocketMQ支持的序列化类型 

```java
public enum SerializeType {
    JSON((byte) 0),
    ROCKETMQ((byte) 1);
}
```

对整体结构有认识后，接下来再结合一下源码在巩固一下，同时如果哪天你也需要写底层网络通讯那块内容，也可以借鉴一下RocketMQ的代码实现(个人觉得这块写得不怎么样)

首先看一下如何编码，RocketMQ将传输的数据抽象成了RemotingCommand

```java
public class NettyEncoder extends MessageToByteEncoder<RemotingCommand> {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(RemotingHelper.ROCKETMQ_REMOTING);

    @Override
    public void encode(ChannelHandlerContext ctx, RemotingCommand remotingCommand, ByteBuf out)
        throws Exception {
        try {
            // 编码Header
            ByteBuffer header = remotingCommand.encodeHeader();
            // 向ByteBuf中写入header数据
            out.writeBytes(header);
            // 向ByteBuf中写入body数据
            byte[] body = remotingCommand.getBody();
            if (body != null) {
                out.writeBytes(body);
            }
        } catch (Exception e) {
            log.error("encode exception, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()), e);
            if (remotingCommand != null) {
                log.error(remotingCommand.toString());
            }
            RemotingUtil.closeChannel(ctx.channel());
        }
    }
}
```

这里只看到写入header和body，上面图中的length和序列化类型在哪呢？不要慌，点进去看看header里面时什么数据

```JAVA
public ByteBuffer encodeHeader() {
    return encodeHeader(this.body != null ? this.body.length : 0);
}
```

```JAVA
public ByteBuffer encodeHeader(final int bodyLength) {
    // 1> header length size
    int length = 4;

    // 2> header data length
    byte[] headerData;
    // 将header使用设置的SerializeType进行序列化，RockerMQ默认使用JSON进行序列化
    headerData = this.headerEncode();

    length += headerData.length;

    // 3> body data length
    length += bodyLength;
		// 在此之前 length = 4 +  headerData.length + bodyLength，所以这个result的ByteBuffer大小是 4 + 4 +  headerData.length，其中第一个4是消息的代表写入length，第二个4是markProtocolType()方法写入，最后是headerData
    ByteBuffer result = ByteBuffer.allocate(4 + length - bodyLength);

    // 写入length
    result.putInt(length);

    // 写入header length
    result.put(markProtocolType(headerData.length, serializeTypeCurrentRPC));

    // header data
    result.put(headerData);

    result.flip();

    return result;
}
```

```java
public static byte[] markProtocolType(int source, SerializeType type) {
    byte[] result = new byte[4];

    result[0] = type.getCode();
    result[1] = (byte) ((source >> 16) & 0xFF);
    result[2] = (byte) ((source >> 8) & 0xFF);
    result[3] = (byte) (source & 0xFF);
    return result;
}
```

现在可以理解 length 是 序列化类型 + header length + header data + body data 的字节长度了吧

其中markProtocolType方法其实是header length和SerializeType，第1个字节表示SerializeType，后面3个字节才是表示header length

到这里Encode基本就完了，可以说还是比较简单的，接下来我们再看看Decode

```java
public class NettyDecoder extends LengthFieldBasedFrameDecoder {

    public NettyDecoder() {
        super(FRAME_MAX_LENGTH, 0, 4, 0, 4);
    }

    @Override
    public Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        ByteBuf frame = null;
        try {
            frame = (ByteBuf) super.decode(ctx, in);
            if (null == frame) {
                return null;
            }

            ByteBuffer byteBuffer = frame.nioBuffer();

            return RemotingCommand.decode(byteBuffer);
        } catch (Exception e) {
            log.error("decode exception, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()), e);
            RemotingUtil.closeChannel(ctx.channel());
        } finally {
            if (null != frame) {
                frame.release();
            }
        }

        return null;
    }
}
```

RocketMQ解码实现是通过`NettyDecoder`这个类，它继承了Netty的`LengthFieldBasedFrameDecoder`，调用了`LengthFieldBasedFrameDecoder`的构造函数

```java
super(FRAME_MAX_LENGTH, 0, 4, 0, 4);
```

这个参数设置就是解析是跳过最开始的4个字节，上面编码的时候最开始的4个字节表示的就是最开始图中的length部分，那么经过

```java
frame = (ByteBuf) super.decode(ctx, in);
```

得到的frame其实就是 **序列化类型 + header length + header data + body data **

然后调用**RemotingCommand.decode(byteBuffer);**进行解码

```java
public static RemotingCommand decode(final ByteBuffer byteBuffer) {
    int length = byteBuffer.limit();
    // 读取一个int，这里就是 序列化类型 + header length
    int oriHeaderLen = byteBuffer.getInt();
    // 从oriHeaderLen解析出header length
    int headerLength = getHeaderLength(oriHeaderLen);

    byte[] headerData = new byte[headerLength];
    byteBuffer.get(headerData);
		// getProtocolType从oriHeaderLen解析出 序列化类型
    // headerDecode 就是根据 序列化类型 进行反序列化，这里没什么说的
    RemotingCommand cmd = headerDecode(headerData, getProtocolType(oriHeaderLen));

    int bodyLength = length - 4 - headerLength;
    byte[] bodyData = null;
    if (bodyLength > 0) {
        bodyData = new byte[bodyLength];
        byteBuffer.get(bodyData);
    }
    cmd.body = bodyData;

    return cmd;
}
```

```java
public static int getHeaderLength(int length) {
    return length & 0xFFFFFF;
}
```

```java
public static SerializeType getProtocolType(int source) {
    return SerializeType.valueOf((byte) ((source >> 24) & 0xFF));
}
```

上面代码的注解把decode的过程描述得还算比较清楚吧