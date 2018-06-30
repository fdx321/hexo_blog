title: 【RocketMQ源码学习】9-事务消息
date: 2017-08-23 20:45:30
tags:
- Java
- RocketMQ
---
还是拿最经典的有君转账给芋芫100元为例来说明事务问题吧，所谓事务，即要保证 “有君账户的钱减100“和”芋芫账户的钱加100”这两个操作要么同时成功，要么同时失败，这在单机情况下很好实现。如果有君账户减100元的操作是在AAA应用里完成的，芋芫账户加100的操作是在BBB应用里完成的，这个事务要怎么保证呢，分布式系统设计领域有一些办法来实现（什么两阶段提交、paxos，raft等，对后两者我一点都不懂，不瞎BB了，说的好像你对两阶段提交很懂一样。。。）。 
我们这里只关注 **小事务 + 异步** 这种方式。所谓**小事务 + 异步**, 就是指 【有君+100】、【芋芫-100】这两个本地事务加一个 AAA 发给 BBB 的异步消息，如图：
![500](/images/【RocketMQ源码学习】9-事务消息_1.png)
<!--more-->
事务消息要解决的问题就是，保证 【有君+100】和 发送MQ 这两个操作要么同时成功，要么同时失败。现在问题来了，这两个操作谁先谁后呢:
* 先【有君+100】后发MQ，前者成功，后者失败了怎么办？
* 先发MQ后【有君+100】，前者成功，后者失败了怎么办？

所以，RocketMQ 是如何来解决这个问题的呢？ 
![600](/images/【RocketMQ源码学习】9-事务消息_2.png)

```java
    public TransactionSendResult sendMessageInTransaction(final Message msg, final LocalTransactionExecuter tranExecuter, final Object arg){
        SendResult sendResult = null;
        MessageAccessor.putProperty(msg, MessageConst.PROPERTY_TRANSACTION_PREPARED, "true");
        MessageAccessor.putProperty(msg, MessageConst.PROPERTY_PRODUCER_GROUP, this.defaultMQProducer.getProducerGroup());

        // 1. 第一阶段，发送 PREPARED 消息
        try {
            sendResult = this.send(msg);
        } catch (Exception e) {
            throw new MQClientException("send message Exception", e);
        }

        LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;

        switch (sendResult.getSendStatus()) {
            case SEND_OK: {
                try {
                    // 第二阶段，执行本地事务操作，即 【有君+100】
                    localTransactionState = tranExecuter.executeLocalTransactionBranch(msg, arg);
                    if (null == localTransactionState) {
                        localTransactionState = LocalTransactionState.UNKNOW;
                    }
                } catch (Throwable e) {
                    ....
                }
            }
            break;
            case FLUSH_DISK_TIMEOUT:
            case FLUSH_SLAVE_TIMEOUT:
            case SLAVE_NOT_AVAILABLE:
                localTransactionState = LocalTransactionState.ROLLBACK_MESSAGE;
                break;
            default:
                break;
        }

        // 到这一步，如果前面的都成功了，LocalTransactionState 应该是COMMIT_MESSAGE, 否则应该是 ROLLBACK_MESSAGE 或 UNKNOWN

        // 第三阶段，会根据 LocalTransactionState 的值，发送不同类型的请求给 broker 去确认第一阶段发的消息。
        try {
            this.endTransaction(sendResult, localTransactionState, localException);
        } catch (Exception e) {
            .....
        }
        .....
    }
```
第一阶段发送的 PREPARED 消息会被 Broker 保存到 commitLog 中，但是不会构建对应的 ConsumeQueue，自然也是不能被消费的。
```java
public void dispatch(DispatchRequest request) {
    final int tranType = MessageSysFlag.getTransactionValue(request.getSysFlag());
    switch (tranType) {
        case MessageSysFlag.TRANSACTION_NOT_TYPE:
        case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
            DefaultMessageStore.this.putMessagePositionInfo(request);
            break;
        // 这两种类型的消息不会构建 ConsumeQueue    
        case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
        case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
            break;
    }
}
```

第三阶段，会发送 RequestCode为END_TRANSACTION 的请求，不同本地事务状态会发送不同类型的消息：
```java
switch (localTransactionState) {
    case COMMIT_MESSAGE:
        requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_COMMIT_TYPE);
        break;
    case ROLLBACK_MESSAGE:
        requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_ROLLBACK_TYPE);
        break;
    case UNKNOW:
        requestHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_NOT_TYPE);
        break;
}
```
Broker的 EndTransactionProcessor 会去做处理,

```java
...
switch (requestHeader.getCommitOrRollback()) {
    case MessageSysFlag.TRANSACTION_NOT_TYPE: {
        // 本地事务状态未知的，啥都不做，直接返回，这意味着 Broker 里之前那条消息一直是 PREPARED 状态
        return null;
    }
    case MessageSysFlag.TRANSACTION_COMMIT_TYPE: {
        break;
    }
    case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE: {
        ...
        break;
    }
}
...
// 设置消息是 ROLLBACK 还是 COMMIT
msgInner.setSysFlag(MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), requestHeader.getCommitOrRollback()));
//ROLLBACK的，把消息内容清空，ROLLBACK的消息是不会去构建ConsumeQueue的，自然也不会被消费
if (MessageSysFlag.TRANSACTION_ROLLBACK_TYPE == requestHeader.getCommitOrRollback()) {
    msgInner.setBody(null);
}
final PutMessageResult putMessageResult = messageStore.putMessage(msgInner);

```

这里有个问题，如果第三阶段发送失败，或者发送的是TRANSACTION_NOT_TYPE的消息，那么 broker 里的消息一直是 Prepared ，一直不能被消费。这种情况该怎么办呢，broker 端会定期扫描这些消息（**我在我看的这个tag里并没有找到这部分代码**），发送 RequestCode 为CHECK_TRANSACTION_STATE给 Producer来询问事务状态。Producer会调用应用代码注册的Listener去决定状态，并告知broker.
```java
public void checkTransactionState(final String addr, final MessageExt msg, final CheckTransactionStateRequestHeader header) {
   Runnable request = new Runnable() {
       @Override
       public void run() {
           // 1. 获取应用代码注册的 TransactionCheckListener
           TransactionCheckListener transactionCheckListener = DefaultMQProducerImpl.this.checkListener();
           if (transactionCheckListener != null) {
               LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
               Throwable exception = null;
               try {
                   // 2. 应用代码决定事务状态
                   localTransactionState = transactionCheckListener.checkLocalTransactionState(message);
               } catch (Throwable e) {
                   ....
               }
               // 3. 这里面会根据状态发送该消息是COMMIT还是ROLLBACK还是继续UNKONWN
               this.processTransactionState(localTransactionState, group, exception);
           } else {
               ...
           }
       }
   }
   ...
}
```

消息被COMMIT后，BBB 就可以消费了，然后就可以执行 【芋芫-100】的操作了。 BBB 消费有异常情况会不停的重试，如果最终还是消费失败，就只能人工介入了。

### **谢晞鸣的思考**
以上是 RocketMQ 处理事务性消息的过程，这种方案和下面这种方案相比，优势在哪里呢？下面这种方案有什么缺点吗？

 **先【有君+100】后发MQ，然后把这两个操作放在一个本地事务里（这里假设用的是Spring事务模板）），如果 【有君+100】失败，事务直接回滚，消息自然也不会发送，如果 【有君+100】 成功，消息发送失败，Spring捕捉到异常后会回滚事务，也没问题。**



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
img[title="500"] {
  width:500px;
  display: block;
}
img[title="600"] {
  width:600px;
  display: block;
}
</style>
