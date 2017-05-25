title: 【Tomcat学习笔记】8-Context创建过程（应用的部署过程）
date: 2017-05-22 18:50:28
tags:
- Java
- Tomcat
---
之前说过，Context 和 Wrapper 的创建和其他组件的创建过程有点不一样，这两者没有在 server.xml 中配置，并不是在解析 server.xml 的时候创建的. 谢晞鸣 Debug 代码，发现。。。

HostConfig 是 StandardHost 的一个 LifeCycleListener，在 StandardHost start 的时候，会触发 START_EVENT 事件，
HostConfig 监听到该事件后，。。。。

```java
//HostConfig#start
public void start() {
    ....
    if (host.getDeployOnStartup())
        deployApps();
    ...
}
```
<!--more-->
判断是否 deployOnStartup，如果是，就开始部署app。*如果 deployOnStartup 是 false，那么是在什么时候部署的呢？好问题，谢晞鸣还不知道，记下来，回头去研究一下。*
```java
protected void deployApps() {
    ...
    // Deploy XML descriptors from configBase
    deployDescriptors(configBase, configBase.list());
    // Deploy WARs
    deployWARs(appBase, filteredAppPaths);
    // Deploy expanded folders
    deployDirectories(appBase, filteredAppPaths);

}
```
deployApps 里有三种部署方式
* Descriptor
* War
* Directory

下面以我写的 jsp-demo 的部署过程来一步步分析 deployDirectories 是怎么做的，*你为啥不分析deployDescriptors呢，靠，descriptors是什么鬼我TM不知道啊，有空再细看吧*


deployDirectories() webapps 目录下的其他文件过滤掉，只部署 directory 形式的。部署之前，会首先检查一下该应用是否已经部署过了，没有部署过则创建一个部署任务，丢给startStopExecutor. 后面会依次

* 实例化一个 StandardContext  SC
* 实例化一个 ContextConfig CC
* 将ContextConfig对象CC添加到StandardContext对象SC的生命周期监听器中去
* 将 jsp-demo 这个 Web App 的相关信息 set 到 Context 中去
* 将SC添加到StandardHost的children中去，在addChild的过程中，会调用该child的start方法，也就是SC的start方法
* 部署完后，会将该 app 添加到一个 map 中，方便以后查看或检查  

```java
Map<String, DeployedApplication> deployed = new ConcurrentHashMap<>();
```

So, 接下来就看 StandardContext 会如何去处理这个 jsp-demo 了。

#### **StandardContext#startInternal**
讲道理，这个函数很长，我很难有条理的说明白，列几个它主要干的事情吧。
1. 创建 WebappLoader, 设置 parentClassLoader 为 sharedClassLoader, delegate 默认为 false. (WebappLoader 主要是后面用来管理WebappClassLoader的，这两个不是一个东西，前者不是ClassLoader), TODO, delegate true 和 false 有什么区别？
2. 用ExtensionValidator做依赖检查，ExtensionValidator里面保存了已经加载的依赖，是在StandardServer#init的时候加进去的，之前有说过
3. 创建WebappClassLoader, 设置它的一些属性，并将当前线程的ClassLoader 切换成 WebappClassLoader (关于这个ClassLoader在我的另一篇笔记中会详细介绍)
4. 触发 Lifecycle.CONFIGURE_START_EVENT 事件，ContextConfig 会监听该事件，然后执行 ContextConfig#configureStart(ContextConfig 里会做很多事情，解析 web.xml, 创建 servlet, 创建 wrapper 等，后面具体分析)
5. 启动 wrapper
6. 启动 pipeline
7. 创建并启动 contextManager
8. 通过 StandardWrapper 启动并加载 各自的 servlet

未完待续。。。。













##### .
** 以上皆是阅读源码 https://github.com/fdx321/tomcat8.0-source-research 所得 **
