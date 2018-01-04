title: Kafka Vs RocketMQ
date: 2018-01-03 13:27:48
tags:
- Java
- Scala
- RocketMQ
- Kafka
---
以下所有的分析都是基于 RocketMQ (tag rocketmq-all-4.1.0-incubating) 和 Kafka (tag 0.10.0.0)，对于 Kafka，涉及 Producer 和 Consumer 的，都是基于该版本的 Java API.
过去大半年时间断断续续看了很多 RocketMQ 和 Kafka 的源码，下面从源码的角度分析下两者的共同点/不同点。

### **部署和存储模型**
RocketMQ 的部署是主从架构，可以一主无备，一主一备，一主多备，多主多备，可靠性依次递增。
Kafka 的部署是互为主备的架构，一个 Broker 即可以是某个分区的主副本，又可以是其他分区的从副本。 <!--more-->

![](/images/Kafka-VS-RocketMQ_1.png)

底层存储上，如图，以 4 台 Broker 为例，
**Kafka**: 假设 Topic_A 有 3 个 Partition, 每个 Partition 有 3 个 Replica. 具体的分布如图（左上）所示，红色表示主副本，其他都是普通副本。每台 Broker 上都有图（左下）所示的目录结构，用来存储日志。 对于 Topic_A 的每个 Partition，都有 CommitLog 来存储消息，每个 CommitLog 有多个 Segment (默认大小是1G，如 /logs/kafka-logs/topic_A-0/00000000000000000000.log). 当 Segment 写满，或者 Segment 对应的 Index 写满，或者 Segment 存在超过一定的时间之后，会重新创建一个 Segment 来存储该 Partition 的消息。 每个 Segment 都有一个 Index 文件，这里的 Index 是一个稀疏索引，并不是 Segment 里的每一条消息都会在 Index 里有记录， 而是当 Segment 写入的消息超过一定的数量后，就往 Index 写入一条记录。

**RocketMQ**: 假设 Topic_A 有 3 个 ConsumeQueue. 具体的分布如图（右上）所示，两主两备。broker_a(master) 和 broker_b(master) 对外提供服务。每台 Broker 上都有图 (右下) 所示的目录结构，用来存储日志。所有的消息都会存储在 commitLog 目录下，该目录下每个文件的大小默认是 1G, 超过该大小就新创建一个。consumequeue 目录下，每个 Topic 都会创建 3 个 目录（0，1，2），每个目录下都会有 consumequeue 的 log文件。 consumequeue 可以理解成 commitLog 的一个索引，但和 commitLog 目录同级的还有一个 index 目录，index 目录下会有针对 commitLog 的 index 文件，这个 index 文件主要是用来根据 key 或 timestamp 来查询消息。


### **有序消息**
**Kafka**: 一个 Topic 有多个 Partition 的场景下，能保证每个 Partition 里的消息是有序的。所以要做到绝对有序消息，只需要给这个 Topic 配置一个 Partition 就可以了。

**RocketMQ**: 一个 Topic 可以有多个 consumequeue，每个 consumequeue 里的消息是有序的（前提用的是 sendSync，而不是sendAsync 和 sendOneway）。所以要做到绝对的有序消息，只需要给这个 Topic 配置一个 consumequeue 就可以了。

当然，它们还支持条件有序（举个例子，同一个订单相关的所有消息有序），针对这种场景：
**Kafka**: 一个 Topic 可以配置多个 Partition, Producer 在发送消息的时候，按照一定的策略(同一个订单的消息用同样的key）选择 Partition 来写消息，保证同一个订单的消息写道同一个 Partition 上。
**RocketMQ**: 和 Kafka 差不多，一个 Topic 配置多个 consumequeue，Producer 在发送消息的时候，用自定义的 MessageQueueSelector 来保证同一个订单的消息落到同一个 consumequeue 里。

### **发送端负载均衡**
**Kafka**: 一个 Topic 分配多个 Partition，每个 Partition 的 Replica 分配到各个 Broker上（Partition的分配，Leader Replica 的选举都是通过 Controller 和 ZK 来实现的，比较复杂）。Producer 在发送消息的时候，如果消息设置了 key，就用 key 的 hash % partitionNums 来选择 Partition, 如果没有设置，就随机选一个放到本地缓存，以后同一个 Topic 的消息就都用这个Partition了。

**RocketMQ**: 同样，也是按照一定的策略(默认的，也可以用自定义的)选择 Consumequeue.

### **消费端负载均衡**
**Kafka**: 当有新 Consumer 加入 ConsumerGroup，会发送 JoinGroup 消息给 Coordinator，Coordinator 收集到所有消息后，会选择一个主 Consumer， 把所有 Consumer 的信息以及 Partition 信息发送给它，主 Consumer 执行 Rebalance 算法，然后把结果告知 Coordinattor，然后 Coordinator 再同步给所有的 Consumer.

**RocketMQ**: 每个 Consumer 从 NameSrv 拿到 Consumequeue 相关信息后，自己执行 Rebalance.

### **数据可靠性**
**Kafka**: 支持同步/异步复制，异步刷盘(支持同步刷盘吗？)。
Producer 发送消息时，如果 acks 参数是 -1, broker 会保证所有 Replica 都同步后再返回。可以通过  log.flush.interval.messages 和 log.flush.interval.ms 来控制刷盘的频率。个人觉得，对于同步复制的模式，其实没必要同步刷盘，因为所有副本同时挂掉的可能性很小。

**RocketMQ**: 支持同步/异步复制，同步/异步刷盘。
可通过 flushDiskType（SYNC_FLUSH/ASYNC_FLUSH）来配置复制方式，brokerRole(SYNC_MASTER/ASYNC_MASTER) 来配置刷盘方式。个人觉得，对于同步复制的模式，没必要同步刷盘，Master 和它的 Slave 同时挂掉的可能性也很小。

有个问题，同步复制，如果一个 slave 写入失败或超时了，master 上已经写入的消息怎么办？


更详细的分析可以参考：
[Kafka vs RocketMQ——单机系统可靠性](http://jm.taobao.org/2016/04/28/kafka-vs-rocktemq-4/)
[Kafka vs RocketMQ——多Topic对性能稳定性的影响](http://jm.taobao.org/2016/04/20/kafka-vs-rocketmq-3/)
[Kafka vs RocketMQ—— Topic数量对单机性能的影响](http://jm.taobao.org/2016/04/07/kafka-vs-rocketmq-topic-amout/)


### **事务消息**
**Kafka**: 不支持事务消息
**RocketMQ**: [【RocketMQ源码学习】9-事务消息](https://fdx321.github.io/2017/08/23/%E3%80%90RocketMQ%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E3%80%919-%E4%BA%8B%E5%8A%A1%E6%B6%88%E6%81%AF/)

### **消息过滤**
**Kafka**: 不支持，业务代码自己做
**RocketMQ**: [【RocketMQ源码学习】7-消息过滤](https://fdx321.github.io/2017/08/23/%E3%80%90RocketMQ%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E3%80%917-%E6%B6%88%E6%81%AF%E8%BF%87%E6%BB%A4/)



