title: 【RocketMQ源码学习】1-整体结构
date: 2017-08-16 18:51:10
tags:
- Java
- RocketMQ
---
### **1.为什么是RocketMQ**
为什么是 RocketMQ，而不是 ActiveMQ/RabbitMQ/Kafka 呢？这不是技术选型，我只是想找一个业界比较好的、开源的 MQ 系统，学习一下 MQ 的工作原理。所以首选 Java 的(虽然语言对我来说不是问题，然还是有点学习成本的)，这就只剩下 RocketMQ 和 ActiveMQ 了，这两个那就肯定 选RocketMQ 了，毕竟人家是这么吹牛逼的： “万亿级数据洪峰下的分布式消息引擎”。
<!--more-->
### **2.项目结构**
![300](/images/【RocketMQ源码学习】1-总体架构_1.png)
* benchmark, 一些 sh 脚本，调用 example 模块中的 benchmark Producer/Consumer 来做一些测试
* broker, 消息代理，起到串联 Producer/Consumer 和 Store 的作用
* client，包含 Producer 和 Consumer，负责消息的发和收
* common，一些公共代码，供其他模块依赖
* distribution, 一些 sh 脚本和 配置，主要是在部署的时候用的
* example，使用样例，包括各种使用方法，Pull模式、Push模式、广播模式、有序消息、事务消息等等
* filter，过滤器，用于服务端 SQL92 的过滤方式
* filtersrv, 过滤器 server, 用于消费端过滤，主要负责动态的加载和实例化过滤器
* logappender，日志相关
* namesrv，可以理解成注册中心，每个 broker 都会在这里注册，client 也会从这里获取 broker 的相关信息
* openmessaging, 讲真，还没去了解，应该是类似于 opentracing 这种，按照一套标准(或者叫API)，封装了一下RocketMQ原来的接口，让其可以通用一点。我猜的！
* remoting，基于 Netty 实现的网络通信模块，包括 Server 和 Client, client、broker、filtersrv 等模块对它都有依赖
* srvutil, 里面只有两个类，一个是来处理命令行的，一个是来配置 shutdownHook 的
* store，负责消息的存储和读取
* style，代码模板，为了统一代码风格
* test, 测试用例
* tools, 一些工具类，基于它们可以写一些 sh 工具来管理、查看MQ系统的一些信息

### **3.我的重点关注**
这么多模块，我并不是每一个都一行行代码的去读，像 distribution、test、tools 这些我就大概扫了一眼。那么哪些是我重点关注的呢？主要是 broker、client、common、namesrv、store、remoting，这几个是核心模块，值得认真研读，其它模块不要它们也能跑起来。

### **4.RocketMQ逻辑部署结构**
![](/images/【RocketMQ源码学习】1-总体架构_2.png)
这是 RocketMQ 的逻辑部署结构(参考《RocketMQ原理简介 v3.1.1》)，包括 producer/broker/namesrv/consumer 四大部分。namesrv 起到注册中心的作用，部署的时候会用到 rocketmq-namesrv/rocketmq-common/rocketmq-remoting 三个模块的代码；broker 部署的时候会用到 rocketmq-broker/rocketmq-store/rocketmq-common/rocketmq-remoting  四个模块的代码；producer 和 consumer 会用到 rocketmq-client/rocketmq-common/rocketmq-remoting 三个模块的代码，这里虽然将它们分开画了，但实际上一个应用往往既是producer又是consumer。

Consumer 和 Broker 之间其实还可以加一个 filtersrv，用来做消费端的消息过滤。

这里面还有几个概念：
* Producer Group, 一类 Producer 的集合的名称
* Consumer Group, 一类 Consumer 的集合的名称
* Topic, 消息的主题，用来标识一类消息，还有 Tag 可以用来区分一个 Topic 下的多种消息。

Producer Group、Consumer Group 和 Topic 之间并没有强制的某种关系，一个 Producer Group 可以发多个 Topic 的消息，一个 Consumer Group 也可以消费多个 Topic 的消息。一个Consumer Group 可以消费来自多个 Producer Group的消息，一个 Producer Group 的消息也可以被多个 Consumer Group 消费。


### **Reference**
* [RocketMQ 原理简介](http://alibaba.github.io/RocketMQ-docs/document/design/RocketMQ_design.pdf)
* [分布式开放消息系统(RocketMQ)的原理与实践](http://www.jianshu.com/p/453c6e7ff81c)


##### -
** 以上所有扯淡都是基于源码 https://github.com/apache/incubator-rocketmq (*tag:rocketmq-all-4.1.0-incubating*)  **

<style>
img[title="300"] {
  width:300px;
  width:300px;
  display: block;
}
img[title="500"] {
  width:500px;
  height:500px;
  display: block;
}
</style>
