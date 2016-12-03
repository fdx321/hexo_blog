title: Netty学习笔记
date: 2016-11-07 16:29:27
tags:
- Netty
- Java
---
最近看了一些源码(netty-4.1.0.Final)，结合李林锋的[文章](http://www.infoq.com/cn/author/李林锋)学习了一下Netty.
先看几个基本概念，BIO | NIO | AIO

## 1. IO
#### BIO, 同步阻塞式IO
**一个连接一个线程**，数据的读写是阻塞的，可以通过线程池来避免频繁的创建和销毁线程。

#### NIO, 同步非阻塞式IO
所有的连接都会注册到一个多路复用器(selector)上，然后有个线程不停的轮询这个selector，有数据可读时才创建或使用线程池中的线程去处理该数据。

NIO基于Reactor，当socket有流可读或可写入socket时，操作系统会相应的通知引用程序进行处理，**应用再将流读取到缓冲区或写入操作系统**。这个时候，已经不是一个连接对应一个线程了，而是 **一个有效的请求对应一个线程**，当连接没有数据时，是没有工作线程来处理的。<!--more-->

#### AIO, 异步非阻塞式IO
与NIO不同，当进行读写操作时，只须直接调用API的read或write方法即可。这两种方法均为异步的，对于读操作而言，当有流可读取时，操作系统会将可读的流传入read方法的缓冲区，并通知应用程序；对于写操作而言，当操作系统将write方法传递的流写入完毕时，操作系统主动通知应用程序。即可以理解为，read/write方法都是异步的，完成后会主动调用回调函数，而不用像NIO一样主动去轮询。
在JDK1.7中，这部分内容被称作NIO.2，主要在java.nio.channels包下增加了下面四个异步通道：
```
AsynchronousSocketChannel
AsynchronousServerSocketChannel
AsynchronousFileChannel
AsynchronousDatagramChannel
```
其中的read/write方法，会返回一个带回调函数的对象，当执行完读取/写入操作后，直接调用回调函数。



先从宏观上看一下几种线程模型。
## 2. 线程模型
#### 同步阻塞
![](/images/Netty学习笔记_1.png)

来一个连接就创建一个线程，accept, read, write操作都是阻塞的。而且，即使一个连接中没有数据在传输，该连接对应的线程是idle的，不能做其他事情。缺点不说了，显而易见。

#### 伪异步
![](/images/Netty学习笔记_2.png)

本质上和同步阻塞模型一样，也是一个连接对应一个线程，只不过用了线程池，避免了线程频繁的创建和销毁。性能提升有限

#### Reactor 单线程
![](/images/Netty学习笔记_3.png)
一个线程在selector上轮询各种事件(SelectionKey.OP_READ/OP_WRITE/OP_CONNECT/OP_ACCPET)，然后进行处理。单线程，性能有限。
#### Reactor 多线程
![](/images/Netty学习笔记_4.png)
一个线程专门处理连接请求(非阻塞的accept, 轮询selector上的OP_ACCPET key)，连接建立后，将新创建的SocketChannel丢给线程池中的线程去处理（read， write操作也是非阻塞的，全部都是主动轮询）。
#### Reactor 主从多线程
![](/images/Netty学习笔记_5.png)
一个线程处理accept请求，可能存在贫瘠，所以相比 *Reactor 多线程模型*，accept 请求也丢给线程池处理。

这几种 Reactor 模型，在用Netty创建Server时，通过合理的配置参数都是可以实现的。下一节会结合代码具体分析。

## 3. 源码分析
#### Reactor 单线程
```java
public class EchoServeHandler extends SimpleChannelInboundHandler<ByteBuf> {
    protected void messageReceived(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
        System.out.println("Server Received: " + msg.toString(CharsetUtil.UTF_8));
        ctx.writeAndFlush(Unpooled.copiedBuffer(msg));
    }
}

public class EchoServer {

    public static void main(String[] args) throws InterruptedException {
        int port = 19999;

        NioEventLoopGroup group = new NioEventLoopGroup(1);
        NioEventLoopGroup childGroup = new NioEventLoopGroup(1);

        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(group, childGroup)
                .channel(NioServerSocketChannel.class)
                .localAddress(new InetSocketAddress(port))
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new EchoServeHandler());
                    }
                });

        try {
            ChannelFuture future = bootstrap.bind().sync();
            System.out.println(EchoServer.class.getName() + "started and listen on " + future.channel().localAddress());
            future.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            group.shutdownGracefully().sync();
        }

    }
}
```
这个样例Server的功能就是将从client收到的数据echo回client.
先来看下Server的启动过程：
![](/images/Netty学习笔记_6.png)



## Reference
* [Netty系列之Netty线程模型](http://www.infoq.com/cn/articles/netty-threading-model)
