title: 【Motan源码学习】4-服务发布与引用
date: 2017-07-23 22:18:37
tags:
- Java
- Motan
---

### **服务发布**
![](/images/【Motan源码学习】4-服务发布与引用_1.png)
<!--more-->
![500](/images/【Motan源码学习】4-服务发布与引用_2.png)
[图片来源](http://kriszhang.com/motan-rpc-impl/)


### **服务引用**
![](/images/【Motan源码学习】4-服务发布与引用_3.png)
![500](/images/【Motan源码学习】4-服务发布与引用_4.png)
[图片来源](http://kriszhang.com/motan-rpc-impl/)
关于服务引用，要重点看下时序图中红色部分，即服务订阅。主要在下面这个方法，当被注册中心通知或自己主动查询之后，会调用该方法，构建Refer.
```java
/**
  * 1 notify的执行需要串行
  * 2 notify通知都是全量通知，在设入新的referer后，cluster需要把不再使用的referer进行回收，避免资源泄漏;
  * 3 如果该registry对应的referer数量为0，而没有其他可用的referers，那就忽略该次通知；
  * 4 此处对protoco进行decorator处理，当前为增加filters
*/
public synchronized void notify(URL registryUrl, List<URL> urls) {
    if (CollectionUtil.isEmpty(urls)) {
        onRegistryEmpty(registryUrl);
        return;
    }
    // 通知都是全量通知，在设入新的referer后，cluster内部需要把不再使用的referer进行回收，避免资源泄漏
    // ////////////////////////////////////////////////////////////////////////////////
    // 判断urls中是否包含权重信息，并通知loadbalance。
    processWeights(urls);
    List<Referer<T>> newReferers = new ArrayList<Referer<T>>();
    for (URL u : urls) {
        if (!u.canServe(url)) {
            continue;
        }
        Referer<T> referer = getExistingReferer(u, registryReferers.get(registryUrl));
        if (referer == null) {
            // careful u: serverURL, refererURL的配置会被serverURL的配置覆盖
            URL refererURL = u.createCopy();
            mergeClientConfigs(refererURL);
            referer = protocol.refer(interfaceClass, refererURL, u);
        }
        if (referer != null) {
            newReferers.add(referer);
        }
    }
    if (CollectionUtil.isEmpty(newReferers)) {
        onRegistryEmpty(registryUrl);
        return;
    }
    // 此处不销毁referers，由cluster进行销毁
    registryReferers.put(registryUrl, newReferers);
    refreshCluster();
}
```
关于这段代码，有这么几个地方我认为是需要注意的
**1. 入参**
```
(URL registryUrl, List<URL> urls)
```
这两个URL并不是同一个意思，前者是注册URL，后者是服务URL. URL 是 Motan 中很重要的一个类，用于保存一些配置信息，用于标识注册信息或服务信息. 
```java
public class URL {
    // 对于注册URL，protocol 用于标识是哪种注册方式，是consul、zookeeper、local或其他
    // 对于服务URL, protocol 用于标识是哪种服务方式，是默认rpc、injvm、yar、restful或其他
    private String protocol;
    // 对于注册URL，即注册中心的host
    // 对于服务URL, 即服务提供者的host
    private String host;
    // 对于注册URL，即注册中心的port
    // 对于服务URL, 即服务提供者的port
    private int port;
    // 对于注册URL，是 com.weibo.api.motan.registry.RegistryService
    // 对于服务URL，是 服务对应的接口
    private String path;
    // 一些其他参数，具体看下面的两个例子
    private Map<String, String> parameters;
    private volatile transient Map<String, Number> numbers;
}
```


注册URL：即注册中心的信息，举个例子
![400](/images/【Motan源码学习】4-服务发布与引用-5.png)
在注册URL中，会嵌入服务URL，看**embeded**字段，内容是编码后的服务URL，Motan 实现了了 URL 对象和 String 之间按一定规则编解码。

服务URL: 即服务的信息，举个例子
![400](/images/【Motan源码学习】4-服务发布与引用-6.png)

**2.构建新的Refer**
一个注册URL，有多个服务URL，没有服务URL都有一个对应的URL. 构建好后，放在 registryReferers 这个Map里，然后开始刷新Cluster. 刷新
Cluster的过程中，要考虑老的Refer怎么处理。com.weibo.api.motan.cluster.support.ClusterSpi#onRefresh
```java
public synchronized void onRefresh(List<Referer<T>> referers) {
    if (CollectionUtil.isEmpty(referers)) {
        return;
    }
    loadBalance.onRefresh(referers);  //引用替换
    List<Referer<T>> oldReferers = this.referers;
    this.referers = referers; //引用替换
    haStrategy.setUrl(getUrl());
    if (oldReferers == null || oldReferers.isEmpty()) {
        return;
    }
    // 过滤出需要销毁的Refer
    List<Referer<T>> delayDestroyReferers = new ArrayList<Referer<T>>();
    for (Referer<T> referer : oldReferers) {
        if (referers.contains(referer)) {
            continue;
        }
        delayDestroyReferers.add(referer);
    }
    // 延迟销毁，提交给线程池，延迟一秒销毁
    if (!delayDestroyReferers.isEmpty()) {
        RefererSupports.delayDestroy(delayDestroyReferers);
    }
}
```


### **我的思考**
**1. 全量同步**
注册中心服务的同步是全量同步的，即这里的urls包含该注册中心所有的服务，那么量会不会太大？
```java
public synchronized void notify(URL registryUrl, List<URL> urls) {
```
并不会，Motan使用group和module来定义区分不同的服务池，举个例子：
```xml
<!--服务发布的配置，定义了group和module-->
<motan:basicService export="demoMotan:8002" group="motan-demo-rpc"  module="motan-demo-rpc-module1" 
                    application="myMotanDemo" registry="registry" id="serviceBasicConfig"/>
<!--服务引用的配置，也定义了group和module，在notify的时候，只会同步给group/module的服务-->
<motan:basicReferer group="motan-demo-rpc" module="motan-demo-rpc-module1"
                    application="myMotanDemo" protocol="motan" registry="registry"
                    id="motantestClientBasicConfig" />
```                        


##### .
** 以上皆是阅读源码 https://github.com/weibocom/motan （tag 0.3.1）所得，文中贴的代码或配置有所删减 **

### **Reference**
[从motan看RPC框架设计](http://kriszhang.com/motan-rpc-impl/)


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
