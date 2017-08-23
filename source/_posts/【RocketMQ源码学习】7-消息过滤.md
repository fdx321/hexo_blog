title: 【RocketMQ源码学习】7-消息过滤
date: 2017-08-23 07:02:03
tags:
- Java
- RocketMQ
---
### **Broker端的过滤**
Broker端的过滤支持 TAG 和 SQL92 两种方式：
**1\. TAG方式**
消息除了指定Topic还可以指定TAG，如果一个消息有多个TAG，可以用||分隔。Consumer 端在订阅消息的时候，可以指定消费哪种TAG的消息，比如：
```java
// 表示订阅TAG为AAA和BBB的消息，如果传 * ,表示订阅该Topic的所有消息
consumer.subscribe("TopicTest", "AAA||BBB");
```
Consumer 会将这个订阅请求构建成一个 SubscriptionData，并告知 Broker.
<!--more-->
```java
public class SubscriptionData{
    public final static String SUB_ALL = "*";
    private String topic;
    private String subString; //就是上面样例的 AAA||BBB
    private Set<String> tagsSet = new HashSet<String>(); // AAA和BBB
    private Set<Integer> codeSet = new HashSet<Integer>();//hash(AAA)和hash(BBB)
    private long subVersion = System.currentTimeMillis();
    private String expressionType; //TAG或SQL92, 这里是TAG
    // 下面这两个是 Consumer 端过滤相关的，后面再讨论
    private boolean classFilterMode = false;
    private String filterClassSource;
}
```

Broker 在去 Store 拿数据之前，会用这些数据先构建一个 MessageFilter，然后传给 Store. Store 从 ConsumeQueue拿到一条记录后，会用它记录的消息tag hash值去做过滤。
```java
...
long tagsCode = bufferConsumeQueue.getByteBuffer().getLong();
// isMatchedByConsumeQueue 会用 tagsCode 和  subscriptionData 中的 tag 做比较，看是否匹配
if (messageFilter != null && !messageFilter.isMatchedByConsumeQueue(tagsCode, extRet ? cqExtUnit : null)) {
    if (getResult.getBufferTotalSize() == 0) {
        status = GetMessageStatus.NO_MATCHED_MESSAGE;
    }
    ...
}
...
// ConsumeQueue 的过滤通过了，用CommitLog中的数据进行进一步的过滤，针对TAG方式，这里面直接返回true
if (messageFilter != null && !messageFilter.isMatchedByCommitLog(selectResult.getByteBuffer().slice(), null){
    if (getResult.getBufferTotalSize() == 0) {
        status = GetMessageStatus.NO_MATCHED_MESSAGE;
    }
    ...
}
...

```
用 hash code 来过滤是为了效率高，不能保证准确性（因为hash冲突），所以 Consumer 端在拿到消息后，会做进一步精确的过滤。
```java
if (msg.getTags() != null) {
    if (subscriptionData.getTagsSet().contains(msg.getTags())) {
        msgListFilterAgain.add(msg);
    }
}
```


**1\. SQL92方式**
SQL92方式支持一些更高级的过滤，举个例子：
```java
// 构建消息并发送
Message msg = new Message("TopicTest", tag, ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
msg.putUserProperty("a", String.valueOf(i));
SendResult sendResult = producer.send(msg);

//订阅消息
consumer.subscribe("TopicTest", MessageSelector.bySql("a between 0 and 3");
```

Producer 构建并发送了一个消息，增加了属性 a. Consumer 用 "a between 0 and 3" 来表示只订阅 a 的值在 0和3 之间的数据。
"a between 0 and 3" 是一种 SQL 语法，更多过滤语法可以参考官方文档。


同样，Consumer 端在订阅的时候也会去构架一个SubscriptionData，并告知 Broker
```java
public class SubscriptionData{
    private String topic;
    private String subString; //就是上面样例的 "a between 0 and 3"
    private String expressionType; //SQL92
}
```

Store的过滤过程如下，调用的方法和TAG方式一样，但里面的操作是不一样的
```java
...
long tagsCode = bufferConsumeQueue.getByteBuffer().getLong();
// isMatchedByConsumeQueue 会用 cqExtUnit 去 BloomFilter 里找这个之前是否命中过
if (messageFilter != null && !messageFilter.isMatchedByConsumeQueue(tagsCode, extRet ? cqExtUnit : null)) {
    if (getResult.getBufferTotalSize() == 0) {
        status = GetMessageStatus.NO_MATCHED_MESSAGE;
    }
    ...
}
...
// ConsumeQueue 的过滤通过了，用CommitLog中的数据进行真正的 SQL 过滤。
if (messageFilter != null && !messageFilter.isMatchedByCommitLog(selectResult.getByteBuffer().slice(), null){
    if (getResult.getBufferTotalSize() == 0) {
        status = GetMessageStatus.NO_MATCHED_MESSAGE;
    }
    ...
}
```
真正的 SQL expression 的构建和执行是由 rocketmq-filter 模块负责的，SQL 表达式的解析和执行不是我关注的重点，这里不详述（尼玛，明明是懒没去看，明明是水平有限看不懂还在这装）。每次过滤都去执行SQL表达式会影响效率，所以RocketMQ使用了BloomFilter避免了每次都去执行，BloomFilter的原理这里也不赘述了。

### **Consumer端的过滤**
Consumer端除了简单的TAG过滤（前面已经介绍过）外，还支持更灵活的过滤，使用者自己写过滤代码，上传到过滤服务器去执行，想怎么过滤就怎么过滤。rocketmq-filtersrv 模块是专门做这个工作的，部署的时候在 consumer 和 broker 之间加一层 filtersrv, broker 拿到数据后，先交给 filtersrv, 过滤完后再交给 consumer.
filtersrv 会动态加载并实例化filter class, 执行它的过滤方法。过滤器源码可以通过该接口上传到filtersrv：
```java
public void subscribe(String topic, String fullClassName, String filterClassSource)
```
也可以单独搞个web系统，来管理（增删改查）这些源码，filtersrv 可以按一定频率去这个系统 fetch 过滤器源码。



### **Reference**
* [RocketMQ 原理简介](http://alibaba.github.io/RocketMQ-docs/document/design/RocketMQ_design.pdf)
* [分布式开放消息系统(RocketMQ)的原理与实践](http://www.jianshu.com/p/453c6e7ff81c)

##### -
** 以上所有扯淡都是基于源码 https://github.com/apache/incubator-rocketmq (*tag:rocketmq-all-4.1.0-incubating*)  ** 所贴代码有所删减。

<style>
img[title="200"] {
  width:200px;
  display: block;
}

img[title="300"] {
  width:300px;
  width:300px;
  display: block;
}
img[title="700"] {
  width:700px;
  display: block;
}
</style>