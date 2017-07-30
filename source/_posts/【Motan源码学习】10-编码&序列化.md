title: 【Motan源码学习】10-编码&序列化
date: 2017-07-29 11:23:53
tags:
- Java
- Motan
---
编解码是通过在Netty中添加Pipeline来实现的。
```java
//com.weibo.api.motan.transport.netty.NettyServer#initServerBootstrap
bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
    public ChannelPipeline getPipeline() {
        ChannelPipeline pipeline = Channels.pipeline();
		pipeline.addLast("channel_manage", channelManage);
		pipeline.addLast("decoder", new NettyDecoder(codec, NettyServer.this, maxContentLength));
		pipeline.addLast("encoder", new NettyEncoder(codec, NettyServer.this));
		pipeline.addLast("handler", handler);
		return pipeline;
    }
});
```
```java
//com.weibo.api.motan.codec.Codec
@Spi(scope=Scope.PROTOTYPE)
public interface Codec {
	byte[] encode(Channel channel, Object message) throws IOException;
	Object decode(Channel channel, String remoteIp, byte[] buffer) throws IOException;
}
```
Codec是一个扩展点，提供encode/decode两个方法。motan-core 实现了 DefaultRpcCodec 和 CompressRpcCodec， 后者和前者差不多，就是编码后又用GZIP压缩了一下。motan-extensios 里还实现了处理 protobuf 的 Codec. 下面以DefaultRpcCodec为例了解下Motan的编码机制，解码机制类似，就是反过来，就不细说了。 <!--more-->

编码后的数据分为两部分，可以理解成消息头(长度128位)和消息体，消息头就是MAGIC NUMBER、Id、Version等信息，消息体就是真正的请求数据，消息体的主要数据是用serialize模块序列化过的，还有一些其他meta data是没有经过编码处理的。motan-core 里实现了 hessian2 和 fastjson 两种序列化方式，motan-extensios 里还实现了protobuf的序列化。编码后的消息格式如下：

Bit位 | 数据内容 
- | :-: 
0 - 15  | Magic Number, Motan用的是0xF0F0
16 - 23 | Version
24 - 31 | Flag，
32 - 95 | RequestId, 64位Long型
96 - 127| body content length
128 - XXX| 消息体

Flag用于标识消息的类型，目前定义了这么些 
```java
FLAG_REQUEST = 0x00;	
FLAG_RESPONSE = 0x01;
FLAG_RESPONSE_VOID = 0x03;
FLAG_RESPONSE_EXCEPTION = 0x05;
FLAG_RESPONSE_ATTACHMENT = 0x07;
FLAG_OTHER = (byte) 0xFF;
```
上面表格描述的消息格式对应代码就是：
```java
//com.weibo.api.motan.protocol.rpc.DefaultRpcCodec#encode(byte[], byte, long)
private byte[] encode(byte[] body, byte flag, long requestId) throws IOException {
    byte[] header = new byte[RpcProtocolVersion.VERSION_1.getHeaderLength()];
    int offset = 0;
    // 0 - 15 bit : magic
    ByteUtil.short2bytes(MAGIC, header, offset);
    offset += 2;
    // 16 - 23 bit : version
    header[offset++] = RpcProtocolVersion.VERSION_1.getVersion();
    // 24 - 31 bit : extend flag
    header[offset++] = flag;
    // 32 - 95 bit : requestId
    ByteUtil.long2bytes(requestId, header, offset);
    offset += 8;
    // 96 - 127 bit : body content length
    ByteUtil.int2bytes(body.length, header, offset);
    byte[] data = new byte[header.length + body.length];
    System.arraycopy(header, 0, data, 0, header.length);
    System.arraycopy(body, 0, data, header.length, body.length);
    return data;
}
```
接下来看下消息体(body)那部分数据是如何被序列化的，这里以Request的encode为例，Response的类似。消息体的构建分为三部分：

**1. Interface、Method、PatametesDesc**
直接 writeUTF，解码的时候按顺序执行三次 readUTF 就能读取到这三个信息了。关于 writeUTF, *writeUTF Writes two bytes of length information to the output stream, followed by the modified UTF-8 representation of  every character in the string.*
**2. arguments**
通过ExtensionLoader获得序列化器，然后将所有参数依次序列化。
**3. attachments**
attachments 是一个 Map 的列表。先在输出流里写入attachments的数量，然后用 writeUTF 依次写入各个Map的Key/value. 那么，什么时候需要用到attachment呢，attachment用来传递那些不影响服务调用、但可以做一些辅助的事情的数据。比如填一些信息用来在Filter里记录日志，用来做trace等。

```java
private byte[] encodeRequest(Channel channel, Request request) throws IOException {
    ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
    ObjectOutput output = createOutput(outputStream);

	//1.
    output.writeUTF(request.getInterfaceName());
    output.writeUTF(request.getMethodName());
    output.writeUTF(request.getParamtersDesc());

	//2.
    Serialization serialization =
            ExtensionLoader.getExtensionLoader(Serialization.class).getExtension(
                    channel.getUrl().getParameter(URLParamType.serialize.getName(), URLParamType.serialize.getValue()));
    if (request.getArguments() != null && request.getArguments().length > 0) {
        for (Object obj : request.getArguments()) {
            serialize(output, obj, serialization);
        }
    }
	//3.
    if (request.getAttachments() == null || request.getAttachments().isEmpty()) {
        output.writeInt(0);
    } else {
        output.writeInt(request.getAttachments().size());
        for (Map.Entry<String, String> entry : request.getAttachments().entrySet()) {
            output.writeUTF(entry.getKey());
            output.writeUTF(entry.getValue());
        }
    }

    output.flush();
    byte[] body = outputStream.toByteArray();
    byte flag = MotanConstants.FLAG_REQUEST;
    output.close();
    return encode(body, flag, request.getRequestId());
}
```

##### .
** 以上皆是阅读源码 https://github.com/weibocom/motan （tag 0.3.1）所得，文中贴的代码或配置有所删减 **


