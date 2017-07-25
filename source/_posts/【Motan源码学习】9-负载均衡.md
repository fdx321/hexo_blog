title: 【Motan源码学习】9-负载均衡
date: 2017-07-28 22:31:23
tags:
- Java
- Motan
---
Motan中的LoadBalance就是实现按一定的策略选择Refer去调用服务。这是一个扩展点，用户可以方便的扩展，按自己的需求实现策略。Motan支持以下策略。
**1. RandomLoadBalance**
没什么可说的，随机从数组里取一个Refer.
**2. RoundRobinLoadBalance**
也没啥可说的，就是一个数组循环取。
**3. LocalFirstLoadBalance**
本地服务优先，对referers根据ip顺序查找本地服务，多存在多个本地服务，获取Active最小的本地服务进行服务。当不存在本地服务，但是存在远程RPC服务，则根据ActivWeight获取远程RPC服务。当两者都存在，所有本地服务都应优先于远程服务，本地RPC服务与远程RPC服务内部则根据ActiveWeight进行
**4. ActiveWeightLoadBalance**
按活跃权重（即Refer被在使用的count），被用的少的优先。由于Referer List可能很多，比如上百台，如果每次都要从这上百个Referer或者最低并发的几个，性能有些损耗，因此 random.nextInt(list.size()) 获取一个起始的index，然后获取最多不超过MAX_REFERER_COUNT的，状态是isAvailable的referer进行判断activeCount. <!--more-->
**5. ConfigurableWeightLoadBalance**
*可配置权重，这个我看了代码之后还有点疑问，像作者提问了，得到答复后再总结。*
**6. ConsistentHashLoadBalance**
一致性Hash，关于一致性Hash算法，这里就不介绍了, 网上有很多介绍。Motan的实现上，循环N次，每次把所有的refers打乱，添加到一个List里。这样最后那个List就有 N * refers.size() 个 refer. 按一致性Hash算法，可以理解成有N个Node. 每次取的时候，根据request计算hash值，然后看hash % List的size, 决定取这个环里的哪一个refer.

##### .
** 以上皆是阅读源码 https://github.com/weibocom/motan （tag 0.3.1）所得，文中贴的代码或配置有所删减 **
