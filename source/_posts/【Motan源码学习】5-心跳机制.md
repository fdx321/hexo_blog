title: 【Motan源码学习】5-心跳机制
date: 2017-07-24 22:40:11
tags:
- Java
- Motan
---
Motan的心跳机制有两种，一种是Server(或者说服务发布者)发给注册中心的，一种是Client(或者说服务引用者)发给Server的。
### **Server HeartBeat to Registry**
```java
public AbstractRegistry(URL url) {
    this.registryUrl = url.createCopy();
    MotanSwitcherUtil.registerSwitcherListener(MotanConstants.REGISTRY_HEARTBEAT_SWITCHER, new SwitcherListener() {
        @Override
        public void onValueChanged(String key, Boolean value) {
            if (key != null && value != null) {
                if (value) {
                    available(null);
                } else {
                    unavailable(null);
                }
            }
        }
    });
}
```
在创建Registry的时候会为REGISTRY_HEARTBEAT_SWITCHER这个开关，注册一个监听器，当开关的值发生变更的时候，执行相应的操作。**这个机制是为了实现Server的预览以及优雅停机，Server跟zk等注册中心建立链接后，服务是unreachable状态，此时并不会真正对外提供服务。可以在unreachable状态下进行预览，确认服务正常后，在打开心跳开关，正式提供服务。服务下线时也是先停止心跳变为unreachable状态，等确认没有client调用后就可以停止服务了。**
<!--more-->
**1.ZK作为注册中心**
```java
//com.weibo.api.motan.registry.zookeeper.ZookeeperRegistry
protected void doRegister(URL url) {
    try {
        serverLock.lock();
        // 防止旧节点未正常注销
        removeNode(url, ZkNodeType.AVAILABLE_SERVER);
        removeNode(url, ZkNodeType.UNAVAILABLE_SERVER);
        createNode(url, ZkNodeType.UNAVAILABLE_SERVER);
    } catch (Throwable e) {
        throw new MotanFrameworkException(String.format("Failed to register %s to zookeeper(%s), cause: %s", url, getUrl(), e.getMessage()), e);
    } finally {
        serverLock.unlock();
    }
}
protected void doAvailable(URL url) {
    try{
        serverLock.lock();
        if (url == null) {
            availableServices.addAll(getRegisteredServiceUrls());
            for (URL u : getRegisteredServiceUrls()) {
                removeNode(u, ZkNodeType.AVAILABLE_SERVER);
                removeNode(u, ZkNodeType.UNAVAILABLE_SERVER);
                createNode(u, ZkNodeType.AVAILABLE_SERVER);
            }
        } else {
            availableServices.add(url);
            removeNode(url, ZkNodeType.AVAILABLE_SERVER);
            removeNode(url, ZkNodeType.UNAVAILABLE_SERVER);
            createNode(url, ZkNodeType.AVAILABLE_SERVER);
        }
    } finally {
        serverLock.unlock();
    }
}
```
注册服务的时候（doRegister）, 可以看到创建的Node是Unavailable的（createNode(url, ZkNodeType.UNAVAILABLE_SERVER)）. 当REGISTRY_HEARTBEAT_SWITCHER 这个开发打开的时候，会让该服务生效(doAvailable). 这个过程叫 HEARTBEAT 似乎不太合适，叫开关好像更合理，因为并没有一个线程按照一定的频率发送heartbeat message.

**2.Consul作为注册中心**
```java
//com.weibo.api.motan.registry.consul.ConsulRegistry
protected void doRegister(URL url) {
    ConsulService service = ConsulUtils.buildService(url);
    client.registerService(service);
    heartbeatManager.addHeartbeatServcieId(service.getId());
}
protected void doAvailable(URL url) {
    if (url == null) {
        heartbeatManager.setHeartbeatOpen(true);
    } else {
        throw new UnsupportedOperationException("Command consul registry not support available by urls yet");
    }
}
```
注册服务的时候（doRegister），会在heartbeatManager中添加该服务的Id, 当REGISTRY_HEARTBEAT_SWITCHER 这个开发打开的时候，会让HeartbeatManager开始工作。接下来看看HeartbeatManager是如何工作的：
```java
//com.weibo.api.motan.registry.consul.ConsulHeartbeatManager#start
public void start() {
  heartbeatExecutor.scheduleAtFixedRate(
  	new Runnable() {
  	  @Override
  	  public void run() {
  	  	try {
  	  	  boolean switcherStatus = isHeartbeatOpen();
  	  	  if (isSwitcherChange(switcherStatus)) { // 心跳开关状态变更
  	  	  	processHeartbeat(switcherStatus);
  	  	  } else {// 心跳开关状态未变更
  	  	  	if (switcherStatus) {// 开关为开启状态，则连续检测超过MAX_SWITCHER_CHECK_TIMES次发送一次心跳
  	  	  	  switcherCheckTimes++;
  	  	  	  if (switcherCheckTimes >= ConsulConstants.MAX_SWITCHER_CHECK_TIMES) {
  	  	  	  	processHeartbeat(true);
  	  	  	  	switcherCheckTimes = 0;
  	  	  	  }
  	  	  	}
  	  	  }
  	  	} catch (Exception e) {
  	  	  LoggerUtil.error("consul heartbeat executor err:", e);
  	  	}
  	  }
  	}, 
   ConsulConstants.SWITCHER_CHECK_CIRCLE, ConsulConstants.SWITCHER_CHECK_CIRCLE, TimeUnit.MILLISECONDS);
}
```
ConsulHeartbeatManager 里会有一个线程专门处理心跳，如果REGISTRY_HEARTBEAT_SWITCHER发生变更，就立刻checkPass/checkFail. 其他情况就按一定的频率去发心跳包。

### **Client HeartBeat to Server**
```java
//com.weibo.api.motan.transport.support.AbstractEndpointFactory
public AbstractEndpointFactory() {
    heartbeatClientEndpointManager = new HeartbeatClientEndpointManager();
    heartbeatClientEndpointManager.init();
}
```
EndpointFactory 用于创建 NettyServer 和 NettyClient，是一个扩展点，单例的。在构造函数里会初始化一个心跳管理器。心跳管理器会按一定的频率发送heartbeat message给Server.

```java
//com.weibo.api.motan.transport.support.HeartbeatClientEndpointManager#init
executorService.scheduleWithFixedDelay(new Runnable() {
  @Override
  public void run() {
      for (Map.Entry<Client, HeartbeatFactory> entry : endpoints.entrySet()) {
        Client endpoint = entry.getKey();
        try {
          // 如果节点是存活状态，那么没必要走心跳
          if (endpoint.isAvailable()) {
              continue;
          }
          HeartbeatFactory factory = entry.getValue();
          endpoint.heartbeat(factory.createRequest());
        } catch (Exception e) {
          LoggerUtil.error("HeartbeatEndpointManager send heartbeat Error: url=" + endpoint.getUrl().getUri(), e);
        }
      }
  }
}, MotanConstants.HEARTBEAT_PERIOD, MotanConstants.HEARTBEAT_PERIOD, TimeUnit.MILLISECONDS);
```

Server端会有一个message handler用来处理 heartbeat message. CreateServer的时候，会用HeartMessageHandleWrapper将原始的messageHandler装饰一下，HeartMessageHandleWrapper在收到message后，判断如果是心跳包，就直接response，否则就交给原始的handler处理。 这就是一个很好的**装饰者模式**使用场景啊。
```java
// com.weibo.api.motan.transport.support.AbstractEndpointFactory#createServer
public Server createServer(URL url, MessageHandler messageHandler) {
    HeartbeatFactory heartbeatFactory = getHeartbeatFactory(url);
    messageHandler = heartbeatFactory.wrapMessageHandler(messageHandler);
}

//com.weibo.api.motan.transport.support.DefaultRpcHeartbeatFactory#wrapMessageHandler
public MessageHandler wrapMessageHandler(MessageHandler handler) {
    return new HeartMessageHandleWrapper(handler);
}

//com.weibo.api.motan.transport.support.DefaultRpcHeartbeatFactory.HeartMessageHandleWrapper
private class HeartMessageHandleWrapper implements MessageHandler {
    private MessageHandler messageHandler;
    public HeartMessageHandleWrapper(MessageHandler messageHandler) {
        this.messageHandler = messageHandler;
    }
    @Override
    public Object handle(Channel channel, Object message) {
        if (isHeartbeatRequest(message)) {
            DefaultResponse response = new DefaultResponse();
            response.setValue("heartbeat");
            return response;
        }
        return messageHandler.handle(channel, message);
    }
}
```

### **我的思考**
**1. Server HeartBeat to Registry，在使用ZK作为注册中心的时候，只是在开关状态变更的时候去通知一下ZK. 在使用Consul做为注册中心的时候，却有一个线程按照一定的频率去CheckPass, why?**
zk和consul的心跳机制是不一样的，consul不是类似zk的长链接，所以需要自己定时进行checkpass。

**2. EndpointFactory会创建心跳管理器，不管是Server还是Client都会有线程一直run，会不会有问题？**
Server的endpoints（for (Map.Entry<Client, HeartbeatFactory> entry : endpoints.entrySet())）是空的，因此线程会空跑。那么对Server是否可以不创建HeartbeatEndpointManager呢。我觉得影响不大，后端绝大多数应用，即使服务引用者，也是服务发布者，再加上EndpointFactory是单例的，只有一个HeartbeatEndpointManager。

**3. 当 Server 服务挂掉的时候，注册中心会感知到，然后就会通知各个订阅者。为什么Client 还需要发心跳包检测 Server是否OK呢？**
client心跳并不是针对server当机这种情况，而是针对server服务正常，但是client和server之间网络异常这种情况

感谢作者Ray的解答
![](/images/【Motan源码学习】5-心跳机制-1.jpeg)



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


