title: 【RocketMQ源码学习】4-消息发送
date: 2017-08-21 13:01:45
tags:
- Java
- RocketMQ
---
### **1. Client端，三种发送方式**
RocketMQ 支持常见的三种发送方式，
* SYNC
```java
producer.send(msg)
```
同步的发送方式，会等待发送结果后才返回。可以用 send(msg, timeout) 的方式指定等待时间，如果不指定，就是默认的 3000ms. 这个timeout 最终会被设置到 ResponseFuture 里，再发送完消息后，用 countDownLatch 去 await timeout的时间，如果过期，就会抛出异常。
* ASYNC
<!--more-->
```java
producer.send(msg, new SendCallback() {
    @Override
    public void onSuccess(SendResult sendResult) {
        System.out.printf("%-10d OK %s %n", index, sendResult.getMsgId());
    }
    @Override
    public void onException(Throwable e) {
        System.out.printf("%-10d Exception %s %n", index, e);
        e.printStackTrace();
    }
});
```
异步的发送方式，发送完后，立刻返回。Client 在拿到 Broker 的响应结果后，会回调指定的 callback.  这个 API 也可以指定 Timeout，不指定也是默认的 3000ms. 
* ONEWAY
```java
producer.sendOneway(msg);
```
比较简单，发出去后，什么都不管直接返回。

对于每种方式，Producer 还提供了可以指定 MessageQueue， MessageQueueSelector的API，这属于稍微高端一点的玩法，一般用它提供的默认的策略选择 MessageQueue 就可以了。

### **2. Client端发送过程**
下面以 SYNC 方式为例，看下整个消息的发送过程，其他方式略有差异，总体流程类似。

**1\.  根据 Topic 找到指定的 TopicPublishInfo**
先去本地 map 找，如果没有，就去 Namesrv fetch, 如果 Namesrv 里也没有，则用默认的 Topic 再去 fetch TopicRouteData. 对用用默认 Topic 的这种情况，Client 拿到数据后，会去构建 TopicPublishInfo, 然后用当前的 Topic 作为 key 放到本地 map 里。Broker 在接收到消息的时候，会去更新它本地的配置，然后在 registerBroker 的时候会去更新 namesrv 中的 TopicRouteData 信息，这样 Namesrv 中就会有这样一份配置了。当然，也可以事先在 Namesrv 增加该配置，很多公司内部都有这样定制的平台来管理MQ的接入配置。

```java
public class TopicPublishInfo {
    private boolean orderTopic = false;
    private boolean haveTopicRouterInfo = false;
    private List<MessageQueue> messageQueueList = new ArrayList<MessageQueue>();
    private volatile ThreadLocalIndex sendWhichQueue = new ThreadLocalIndex();
    private TopicRouteData topicRouteData;
}

public class TopicRouteData {
    private String orderTopicConf;
    private List<QueueData> queueDatas;
    private List<BrokerData> brokerDatas;
    private HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
}
```
QueueData 定义了这个 read 和 write 的 queue的数量，Client 在拿到 TopicRouteData 后，会根据这里配的数量去构建响应数目的messageQueue，即 messageQueueList. brokerDatas 保存了各个 broker 的相关信息。

**2\. 从 messageQueueList 中选择一个 MessageQueue**
如果没有 enable latencyFaultTolerance，就用递增取模的方式选择。如果 enable 了，在递增取模的基础上，再过滤掉 not available 的。这里所谓的 latencyFaultTolerance, 是指对之前失败的，按一定的时间做退避：
```java
long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L};
long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L};
```
举个例子，如果上次请求的 latency 超过 550L ms, 就退避 3000L ms；超过 1000L，就退避 60000L.
以上就是 Producer 到 Broker 的简单的负载均衡。

**3\. 发送消息**
到这一步，我们已经拿到了这些关键数据：
* Message， 要发送的消息
* MessageQueue，这里面包括 topic/brokerName/queueId
* CommunicationMode, 发送方式, SYNC/ASYNC/ONEWAY
* TopicPublishInfo

有了这些数据，就可以构建 RequestHeader 了，大部分字段意思都很明显（当然，前提是对RocketMQ的源码有所熟悉），个别字段见注释。
```java
requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
requestHeader.setTopic(msg.getTopic());
requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
requestHeader.setQueueId(mq.getQueueId());
//系统Flag, 用于判断走什么逻辑。标识是否压缩，事务的不同TYPE(prepare/rollback/commit/not transaction) 等
requestHeader.setSysFlag(sysFlag); 
requestHeader.setBornTimestamp(System.currentTimeMillis());
//消息Flag, 最终会落地
requestHeader.setFlag(msg.getFlag());
requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
requestHeader.setReconsumeTimes(0);
//TODO，暂不知道这个字段是干嘛用的
requestHeader.setUnitMode(this.isUnitMode());
requestHeader.setBatch(msg instanceof MessageBatch);
```

最后用这些 header 字段，以及 message body 构建 RemotingCommand，通过 remoting 模块发给 broker.

**4\. 处理结果**
 * 发送成功：直接返回发送结果
 * 发送失败：如果 enable retryAnotherBrokerWhenNotStoreOK，就会重试，默认重试两次(retryTimesWhenSendFailed)。否则直接返回结果
 * 发送异常：Producer 对异常做了很好的区分，如果是 Remoting 和 Client 模块的异常，就重试，如果是 Broker 模块的异常，根据不同的 response code 做不同的处理，有的重试，有的抛出异常，有的返回结果。

### **3. Broker端，消息的处理和落地**
![](/images/【RocketMQ源码学习】4_消息发送_1.png)
如图，Broker 有很多 Processor 用来处理不同类型的请求，有些 Processor 会共用一个 Processor 线程池。对于消息发送，Broker 的 remoting 模块在接收到请求后，根据request code，最终会交给 SendMessageProcessor 来处理。SendMessageProcessor 会依次做以下处理：
* 做一些校验，包括但不限于
   * broker 是否可写
   * topic 配置是否存在，如果不存在就新建一个（createTopicInSendMessageMethod）
   * 校验 queueId 是否超过指定大小
* 构建 MessageExtBrokerInner
* 将 MessageExtBrokerInner 交给 Store 处理
* 处理 Store 返回的结果，BrokerStatsManager 做一些统计更新，设置 Response 中的一些字段并返回。

Store 收到消息后，会先做一些校验，然后交给 commitLog 去 put，然后做些统计并返回。Store 存储消息的过程比较复杂，后面会单独分析。

### **4. 其他**
**1\. 顺序消息**
很多应用并不关注消息顺序，而且消息没有顺序并不代表消息内容没有顺序，合理的系统设计可以避免顺序问题。MQ 要保证消息顺序必然会损失性能、增加系统实现复杂度。具体的分析可以看 [分布式开放消息系统(RocketMQ)的原理与实践](http://www.jianshu.com/p/453c6e7ff81c)。

在 RocketMQ 里, 在发送消息的时候可以自己定义 MessageQueueSelector，对于同一个订单ID（或其他ID）的不同消息，可以让它走同一个 MessageQueue，这样就可以按顺序发给同一个 Broker 了。

**2\. Batch Message**
Producer 的 API 还支持一次发多个消息。
```java
List<Message> messages = new ArrayList<>();
messages.add(new Message(topic, "Tag", "OrderID001", "Hello world 0".getBytes()));
messages.add(new Message(topic, "Tag", "OrderID002", "Hello world 1".getBytes()));

producer.send(messages);
```
Client 模块会将 Message List 封装成 MessageBatch，且会标记 requestHeader 的 batch 标志位为 true. Broker 在接收到消息后就可以根据这个标志位去做不同的处理。


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
img[title="600"] {
  width:700px;
  height:600px;
  display: block;
}
</style>

