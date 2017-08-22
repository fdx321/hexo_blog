title: 【RocketMQ源码学习】2-Namesrv
date: 2017-08-17 10:41:52
tags:
- Java
- RocketMQ
---
### **1. Namesrv 简介**
Namesrv 可以理解为一个注册中心, 整个Namesrv的代码非常简单，主要包含两块功能：
* 管理一些 KV 的配置
* 管理一些 Topic、Broker的注册信息
<!--more-->

![600](/images/【RocketMQ源码学习】2-Namesrv_1.png)

### **2. Namesrv 启动过程**
启动过程主要涉及 NamesrvStartup/NamesrvController 两个类， NamesrvStartup 负责解析命令行的一些参数到各种 Config 对象中（NamesrvConfig/NettyServerConfig等），如果命令行参数中带有配置文件的路径，也会从配置文件中读取配置到各种 Config 对象中，然后初始化 NamesrvController，配置shutdownHook, 启动  NamesrvController。 NamesrvController 会去初始化和启动各个组件，主要是:
* 创建NettyServer，注册 requestProcessor，用于处理不同的网络请求
* 启动 NettyServer
* 启动各种 scheduled task.

不仅仅 Namesrv 是这样，其他模块在启动过程中也都是 startup/controller/config 一起完成这样的套路。


### **3. Namesrv 主要组件**
* Processor 线程池，nettyServer 接收到请求后，封装成任务提交到该线程池。
remoting 模块维护了这样一个 processorTable:
```java
  HashMap<Integer/* request code */, Pair<NettyRequestProcessor, ExecutorService>> processorTable
```
**一个 processor 可以处理多个 request code, 多个 processor 也可以共用一个线程池**。对于 Namesrv, 只有一个 processor 线程池，给两个 Processor 共享。

* DefaultRequestProcessor（Namesrv 还有一个 ClusterTestRequestProcessor 继承了该 Processor，在 clusterTest enable的情* 况下使用它来 getRouteInfoByTopic），用来处理 namesrv 接收到的所有 RequestCode， Processor 内部会根据不同的RequestCode 调用不同的方法。
* KVConfigManager, 维护了一些KV方式的配置数据，可以根据请求，执行添加、删除、查询等操作
* RouteInfoManager, 维护了topic/broker/cluster/filter这些东西的路由信息，同样支持增删改查的操作
* schedued 线程，按一定的频率做两个事情，扫描不活跃的broker；打印所有KV配置信息

### **4. 以broker注册为例看下Namesrv的工作过程 **
1. DefaultRequestProcessor 处理来自 NettyServer的 [RemotingCommand] request， 如果 request.getCode 是 RequestCode.REGISTER_BROKER, 就去注册。这里会根据request.version来判断，从V3_0_11 开始支持了FilterServer。

2. 从 request 解码得到 RegisterBrokerRequestHeader， 包含以下字段：
  * brokerName, // 默认是BrokerConfig里的获得的locakHostName
  * brokerAddr, //brokerConfig.getBrokerIP1() + ":" + nettyServerConfig.getListenPort()
  * clusterName, //默认是BrokerConfig的"DefaultCluster"
  * haServerAddr, //brokerConfig.getBrokerIP2() + ":" + messageStoreConfig.getHaListenPort()
  * brokerId, //如果是MASTER，就是MixAll.MASTER_ID(也就0)，否则就是其他

3. 从 request.body 解码得到 RegisterBrokerBody, RegisterBrokerBody 包含以下内容，用JSON的方式来描述吧:
```js
{
  "topicConfigSerializeWrapper": {
      "topicConfigTable":{
         "topic_xxx":{
           "defaultReadQueueNums":"16",
          "defaultWriteQueueNums":"16",
          "topicName":"xxx",
          "readQueueNums":"",
          "writeQueueNums":"",
          "perm":"",
          "topicFilterType":"",
          "topicSysFlag":"",
          "order":""
         },
      },
      "dataVersion":{
         "timestamp":"xxxx",
         "counter":"xxxx"
      }
   },
  "filterServerList":[
     "",//filterServerAddr
  ]
}
```

4. 在 clusterAddrTable 中新增一条记录
5. 在 brokerAddrTable 中新增一条记录，这里会构建一个BrokerData
```js
{
  "cluster":"xxx",
  "brokerName":"xxx",
  "brokerAddrs":{
     "brokerId_xx":"broker address xxx"
   }
}
```
6. 如果是第一次注册或者topicConfig发生了变更，会去更新topicQueueTable
7. 在brokerLiveTable新增该broker
8. 在filterServerTable新增这些filterServer的地址列表

### **5.其他**
**以上内容看下来，namesrv 是一个无状态的应用，可以水平任意扩展**。**每一个 broker 都会和所有的 namesrv 保持长连接**(有个scheduled task会按一定频率给所有namesrv做register broker的操作)，所以 namesrv 之间没有主从关系，也不需要复制数据。**client(producer/consumer) 随机选一个 namesrv 连接**。client 中的 namesrv 地址列表是怎么来的呢，有两种方式：
1. 通过命令行或配置文件在启动的时候获得的
2. 通过 Scheduled task，按一定的频率从一个 web 服务 fetch的(web服务可以自建)，如果有变更，就更新这个 namesrv 地址列表。
client 选择 namesrv的过程如下, index递增取模，然并不是每次都这么干，取到后会缓存起来。
```java
if (addrList != null && !addrList.isEmpty()) {
    for (int i = 0; i < addrList.size(); i++) {
        int index = this.namesrvIndex.incrementAndGet();
        index = Math.abs(index);
        index = index % addrList.size();
        String newAddr = addrList.get(index);
        this.namesrvAddrChoosed.set(newAddr);
        Channel channelNew = this.createChannel(newAddr);
        if (channelNew != null)
            return channelNew;
    }
}
```
** 看到这里我产生了疑问，那岂不是每个 client 启动的时候都取的是第一个 namesrv，它不会压力很大吗，后来发现 namesrvIndex 的初始值是随机的。**


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

