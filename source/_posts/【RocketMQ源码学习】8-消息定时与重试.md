title: 【RocketMQ源码学习】8-消息定时与重试
date: 2017-08-23 12:06:57
tags:
- Java
- RocketMQ
---
### **消息重试**
Consumer 端消息消费失败后，会send message back to broker.
```java
List<MessageExt> msgBackFailed = new ArrayList<MessageExt>(consumeRequest.getMsgs().size());
for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
    MessageExt msg = consumeRequest.getMsgs().get(i);
    // 将消费失败的消息发回给Broker
    boolean result = this.sendMessageBack(msg, context);
    if (!result) {
        // 消息消费次数递增
        msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
        msgBackFailed.add(msg);
    }
}
// 重新提交ConsumeRequest到队列里
if (!msgBackFailed.isEmpty()) {
    consumeRequest.getMsgs().removeAll(msgBackFailed);
    this.submitConsumeRequestLater(msgBackFailed, consumeRequest.getProcessQueue(), consumeRequest.getMessageQueue());
}
```
<!--more-->
发回去的requestHeader里有两个和重试相关的关键字段 delayLevel（决定了延迟的时间） 和 maxConsumeRetryTimes。

![](/images/【RocketMQ源码学习】8-消息定时与重试_1.png)

Broker 在接收到消息后，将消息的 topic 改成 "%RETRY% + consumerGroup", 将真正的 topic 放到消息的RETRY_TOPIC属性中。
然后交给Store去存储.
```java
String newTopic = MixAll.getRetryTopic(requestHeader.getGroup());
MessageAccessor.putProperty(msgExt, MessageConst.PROPERTY_RETRY_TOPIC, msgExt.getTopic());
msgInner.setTopic(newTopic);
```

store 先判断它的 delayLevel，如果大于0，证明要重试，又将消息的 topic 改成 "SCHEDULE_TOPIC_XXXX"，并把 "%RETRY% + consumerGroup" 保存在消息的 REAL_TOPIC 属性中。然后去写 commitLog，构建 ConsumeQueuqe. 这个时候，图中的
SCHEDULE_TOPIC_XXXX 目录下就会有 ConsumeQueuqe 了.

```java
topic = ScheduleMessageService.SCHEDULE_TOPIC;
queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

// Backup real topic, queueId
MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueO(msg.getQueueId()));

msg.setTopic(topic);
msg.setQueueId(queueId);
```

ScheduleMessageService 是 broker 中一个单独的线程，它会不停的去扫描 SCHEDULE_TOPIC_XXXX 里的 ConsumeQueuqe，根据里面tagsCode字段记录的时间戳判断是否到了delay的时间, 如果到了，就根据 ConsumeQueuqe 找到对应的 CommitLog里的消息，并将它的 topic 改回 "%RETRY% + consumerGroup", 重新交给 store 去做 putMessage，这个时候图中的 %RETRY%xxxxx 之类的目录下就会有 ConsumeQueue了。
```java
msgInner.setTopic(msgInner.getProperty(MessageConst.PROPERTY_REAL_TOPIC));
```
每个delayLevel对应要delay的时间是通过这个变量关联起来的，这个变量在 MessageStoreConfig 中，是可配置的。
```java
private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h";
```

Consumer端在启动的时候，就订阅了 %RETRY%+consumerGroup 这个 topic 的消息，所以它会一直去长轮询这类消息，拿到消息后，将Topic 从 "%RETRY% + consumerGroup" 改回 RETRY_TOPIC属性中的真正的Topic。
```java
public void resetRetryTopic(final List<MessageExt> msgs) {
    final String groupTopic = MixAll.getRetryTopic(consumerGroup);
    for (MessageExt msg : msgs) {
        String retryTopic = msg.getProperty(MessageConst.PROPERTY_RETRY_TOPIC);
        if (retryTopic != null && groupTopic.equals(msg.getTopic())) {
            msg.setTopic(retryTopic);
        }
    }
}
```

### **定时消息**
讲完了消息的重试之后，定时消息原理也就很简单了，Broker先将消息的 消息的 topic 改成 "SCHEDULE_TOPIC_XXXX"，并把真正的Topic保存在消息的 REAL_TOPIC 属性中. ScheduleMessageService 在时间到了之后，会将它捞出来，恢复Topic, 并开始写 CommitLog 和 ConsumeQueue.


### **Reference**
* [RocketMQ 原理简介](http://alibaba.github.io/RocketMQ-docs/document/design/RocketMQ_design.pdf)
* [分布式开放消息系统(RocketMQ)的原理与实践](http://www.jianshu.com/p/453c6e7ff81c)
