title: 【Motan源码学习】7-Shutdown机制
date: 2017-07-26 22:51:37
tags:
- Java
- Motan
---
Motan中很多地方用了线程池，有一些还是按一定频率一直在run的任务，比如心跳、Failback机制中的一些重试线程。这样会有一个问题，当把应用部署在Tomcat中的时候，用shutdown.sh关闭Tomcat的时候，这些线程池就关不掉，导致通过只能kill进程的方式来关闭。

在 0.3.1 中，Motan 实现了 ServletContextListener 接口和 ShutdownHook，在创建任务的时候，会去 ShutdownHook 中注册， 当 Context 关闭的时候，会通知该 Listener，然后该 Listener 就会去 shutdown 已经注册的这些任务。

```java
public class ShutDownHookListener implements ServletContextListener {
    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        ShutDownHook.runHook(true);
    }
}
```
<!--more-->
当然，这个Listener是需要用户在web.xml中配置的，
```xml
<listener> 
    <listener-class>com.weibo.api.motan.closable.ShutDownHookListener</listener-class> 
</listener>
```

我可是看过 Tomcat 源码的人，看下 Tomcat 关闭过程中是如何处理的。首先，ServletContextListener 的通知当然应该在 StandardContext 中做，在 stopInternal() 中会调用 listenerStop()，这里会遍历所有监听器，如果是 ServletContextListener，则通知他们 context 销毁事件。
```java
//org.apache.catalina.core.StandardContext#listenerStop
public boolean listenerStop() {
    Object listeners[] = getApplicationLifecycleListeners();
    for (int i = 0; i < listeners.length; i++) {
        int j = (listeners.length - 1) - i;
        if (listeners[j] == null)
            continue;
        if (listeners[j] instanceof ServletContextListener) {
            ServletContextListener listener = (ServletContextListener) listeners[j];
            try {
                fireContainerEvent("beforeContextDestroyed", listener);
                if (noPluggabilityListeners.contains(listener)) {
                    listener.contextDestroyed(tldEvent);
                } else {
                    listener.contextDestroyed(event);
                }
                fireContainerEvent("afterContextDestroyed", listener);
            } catch (Throwable t) {
                ...
            }
        }
        ....
    }
}
```


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
