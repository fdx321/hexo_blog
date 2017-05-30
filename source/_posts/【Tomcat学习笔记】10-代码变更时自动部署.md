title: 【Tomcat学习笔记】10-代码变更时自动部署
date: 2017-05-30 21:31:19
tags:
- Java
- Tomcat
---
*一直在入门，但从未精通，所以不敢说深入分析 XXX，顶多就是一个学习笔记，但是是原创，转载请注明出处，好吧，说的好像有人会看一样。*
*说明，为了简洁，这里贴的代码都有所删减。*

在看 ClassLoader 代码的过程中，我发现了 WebappLoader 的 reloadable 属性，通过分析代码，把自动部署的过程摸清楚了。这个属性在开发过程中挺有用的，有时候在Debug的时候改了代码，重启Tomcat又要等半天，reloadable 这个属性就可以派上用场了。

每个 Container 在start的最后，都会执行这样一段代码，ContainerBase#threadStart.
```java
protected void threadStart() {
    if (thread != null)
        return;
    if (backgroundProcessorDelay <= 0)
        return;

    threadDone = false;
    String threadName = "ContainerBackgroundProcessor[" + toString() + "]";
    thread = new Thread(new ContainerBackgroundProcessor(), threadName);
    thread.setDaemon(true);
    thread.start();
}
```
<!--more-->
就是每个 Container 会启动一个 后台线程，后台线程按一定的时间间隔不停的运行，这个时间间隔就是 backgroundProcessorDelay ，单位是秒。这里有个判断，如果间隔小于等于0，就不会创建这样的后台线程，这个 backgroundProcessorDelay 是可配置的。

接下来，我们就只看 StandardContext 的这个后台线程干了啥，简单看下调用链路，
ContainerBase.ContainerBackgroundProcessor#run ---> ContainerBase.ContainerBackgroundProcessor#processChildren --> StandardContext#backgroundProcess  --> WebappLoader#backgroundProcess

```java
public void backgroundProcess() {
    if (reloadable && modified()) {
        try {
            Thread.currentThread().setContextClassLoader
                (WebappLoader.class.getClassLoader());
            if (context != null) {
                context.reload();
            }
        } finally {
            if (context != null && context.getLoader() != null) {
                Thread.currentThread().setContextClassLoader
                    (context.getLoader().getClassLoader());
            }
        }
    }
}
```
WebappLoader#backgroundProcess 回去判断 reloadable 属性，当然这个属性也是可配置的，如果为 true且modifed()，则重新加载context. modified 方法主要干这么几件事：
* 检查 WebappClassLoader加载的 class 的 last modify time 有没有变化
* 检查 /WEB-INF/lib 下jar的数量有没有变化（增加或减少），检查它们的 last modify time 有没有变化

**这里有个问题诶，如果是代码里新增了一个类，modifed 会检查到吗，看代码是没有检查，回头写个代码测一下**

context.reload()的过程：
* 设置 paused 为 true(这个变量为 true 了之后，会暂停接收请求，在CoyoteAdapter中会用该变量去判断)
* stop()
* start()
* 设置 paused 为 false
可以看到，Tomcat 中很多像paused这种变量都是 volatile 的，就是为了保证多线程的时候，该变量的可见性。



##### .
** 以上皆是阅读源码 https://github.com/fdx321/tomcat8.0-source-research 所得 **
