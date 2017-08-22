title: 【RocketMQ源码学习】3-Remoting模块
date: 2017-08-21 12:01:09
tags:
- Java
- RocketMQ
---
rocketmq-remoting 模块是 RocketMQ 中负责网络通信的模块，被其他所有需要网络通信的模块依赖。它是基于 Netty 实现的，避免了网络编程很多 tricky 的问题。

![](/images/【RocketMQ源码学习】3-Remoting模块_1.png)
<!--more-->

首先来看下 RocketMQ NettyServer 的 Reactor 线程模型，一个 Reactor 主线程负责监听 TCP 连接请求，建立好连接后丢给 Reactor 线程池，它负责将建立好连接的 socket 注册到 selector 上去（这里有两种方式，NIO和Epoll，可配置），然后监听真正的网络数据。拿到网络数据后，再丢给 Worker 线程池。
Worker 拿到网络数据后，就交给 Pipeline，从 Head 到 Tail 一个个 Handler 的走下去，这些 Handler 是在创建 Server 的时候指定的。NettyEncoder 和 NettyDecoder 负责网络数据和 RemotingCommand 之间的编解码。NettyServerHandler 拿到解码得到的 RemotingCommand 后，根据 RemotingCommand.type 来判断是 request 还是 response，如果是 request, 就根据 RomotingCommand 的 code(code用来标识不同类型的请求) 去 processorTable 找到对应的 processor,然后封装成 task 后，丢给对应的 processor 线程池, 如果是 response 就根据 RemotingCommand.opaque 去 responseTable 中拿到对应的 ResponseFuture,把结果 set 给它。

对于 Client，经过 Pipeline 的顺序是从 Tail 到 Head。不管是 Server 和 Client，并不是每次数据流转都得经过所有的 Handler，而是会根据 Context 中的一些信息去判断。

整个数据流转过程中还有很多hook, 比如处理 command 前，处理 command 后，发送数据前，发送数据后等。


关于 Netty 的一些关键知识点，[Netty学习笔记](https://fdx321.github.io/2016/11/07/Netty%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/) 中做了些总结。

##### -
** 以上所有扯淡都是基于源码 https://github.com/apache/incubator-rocketmq (*tag:rocketmq-all-4.1.0-incubating*)  ** 所贴代码有所删减。

<style>
img[title="300"] {
  width:300px;
  width:300px;
  display: block;
}
img[title="600"] {
  width:700px;
  height:600px;
  display: block;
}
</style>
