title: 【Motan源码学习】1-总体架构
date: 2017-07-19 15:26:56
tags:
- Java
- Motan
---
【Motan源码学习】系列，不搞大而虚的架构，还是会继续深入源码，剖析RPC框架的内幕，会尝试总结出一些好的系统设计，尝试带着问题去看源码。搞这个系列最主要的目的是驱动自己认真的读源码。虽然说不搞大而虚的架构，但脑子里还是需要一幅大图的. 看过 Tomcat 源码之后深有体会，刚开始掌握大概流程和整体架构，然后一步步深入，搞懂图中的每个组件。

### **简介**
Motan 是微博研发并开源的 Java 的 RPC 框架，项目地址：https://github.com/weibocom/motan. 我将在 0.3.1 版本上开始源码的学习。（为什么你不去看 dubbo 呢？dubbo 虽然很屌，除了阿里，在国内很多其他互联网公司也都用它。然。。。我大概过了一遍两个项目，觉得 Motan 更小而美一点，可能看源码的门槛低一点吧）
<!--more-->
### **Motan的使用**
[Motan 快速入门](https://github.com/weibocom/motan/wiki/zh_quickstart). 如果你之前有过 RPC 的使用经验，看这份文档应该很轻松的。

### **Motan的架构**
[Motan 用户指南](https://github.com/weibocom/motan/wiki/zh_userguide)里有简单的介绍.
![400](/images/【Motan源码学习】1-总体架构_2.jpg)
这是RPC框架最最抽象的模型了，服务提供者 将自己的服务注册到注册中心（包括服务提供者的IP，端口，接口等信息），服务使用者订阅注册中心（需要的时候可以主动去注册中心查，也可以接收注册中心的通知）. 通过注册中心，服务使用者就可以拿到服务提供者暴露的服务，然后直接call了。

![450](/images/【Motan源码学习】1-总体架构_1.png)
这是我理解的 Motan 的主要架构，随着后面深入，我可能会进一步完善这张图。下面我会一层层分析这张图，然后决定我接下来要去看哪些东西。

**1. Config** 
配置层，负责解析服务暴露、服务引用的一些配置，支持 XML 方式的配置，Annotation的配置，以及通过API自己手动编程方式的配置.  
**2. Proxy** 
这一层只有服务使用者才会用到。我们都很习惯 object.func(xxx,yyy) 这种方式的接口调用. 当接口调用方和提供方在一个JVM进程里，这很简单。但是当两者属于两个不同的应用、部署在不同的机器上时，我们就必须写一堆代码去完成这个调用，RPC 框架为了让这一堆代码对我们透明，整了一个Proxy层，使用动态代理的技术让服务调用和object.func(xxx,yyy)一样简单。
**3. Registry** 
这一层完成服务的注册和发现，Motan 支持 consul 和 zookeeper 两种注册中心，使用的时候引用不同的依赖就可以了。Motan 使用 SPI 机制很好的支持这部分的扩展，可以很简单的使用其它注册中心。
**4. Cluster**
这部分也是只有服务使用者才会用到，主要做 HA 和 LoadBalance. 默认支持 Fail-Over 和 Fail-Fast 两种 HA 策略，LoadBalance 策略则有 一致性哈希/随机/本地服务优先/可配置权重/Round robin. HA 和 LoadBalance 的策略也是可扩展的，用户可以通过 SPI 机制加入自己需要的策略。
**5. Protocal**
协议层，默认支持 motan （即默认的rpc）和 injvm 两种, injvm 是指服务使用者和服务提供者在一个JVM进程里，即本地调用。Yar（给PHP做的RPC扩展） 和 Restful 则是在扩展里提供的。
**6. Transport**
传输层，默认使用netty。从代码结构上，这里的 Encoder（编码，即序列化）和 Decoder（解码，即反序列化）是放在 protocal 层的，且也是可扩展的，可以使用各种序列化算法。但是使用上，他们是放在 netty 的 pipeline 里的，所以我这里将它们画在了传输层。

### **Motan的工程结构**
![300](/images/【Motan源码学习】1-总体架构_3.png)
Motan 的工程结构也很清晰：
* motan-benchmark，测性能的
* motan-core，RPC 的核心实现
* motan-demo，使用样例
* motan-extension，扩展
* motan-manager，管理后台，可用于运维和监控
* motan-registry-consul, consul 注册中心
* motan-registry-zookeeper， zookeeper注册中心
* motan-springsupport， Spring 集成
* motan-transport-netty， netty 网络通信

前期，我主要会关注 motan-core，motan-registry-consul， motan-springsupport， motan-transport-netty 这四个 package，其他 package 不影响我研究整个RPC内幕。


##### .
** 以上皆是阅读源码 https://github.com/weibocom/motan （tag 0.3.1）所得，文中贴的代码或配置有所删减 **


<style>
img[title="300"] {
  width:300px;
  width:300px;
  display: block;
}
img[title="400"] {
  width:400px;
  width:400px;
  display: block;
}
img[title="450"] {
  width:450px;
  width:450px;
  display: block;
}
img[title="500"] {
  width:500px;
  height:500px;
  display: block;
}
</style>
