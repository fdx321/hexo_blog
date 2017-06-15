title: 【Tomcat学习笔记】11-Tomcat中的JMX
date: 2017-06-15 19:03:39
tags:
- Java
- Tomcat
---
### **JMX 科普**
JMX（Java Management Extensions）其实就是一套标准，定义了一套架构（设计模式或者说API）用于管理和监控Java应用程序。[JMX官方介绍](http://docs.oracle.com/javase/6/docs/technotes/guides/jmx/overview/intro.html#wp5529)

![](/images/【Tomcat学习笔记】Tomcat中JMX的使用_1.png)
这是 JMX 的架构，可以通过各种协议接入进来，然后通过 MBeanServer 操作各个 MBean.

这里的 MBean 分为静态Bean和动态Bean，静态Bean能力有限，只能用于获取一些静态信息。动态Bean, 可以在运行时改变它的属性，执行它的方法，动态Bean，必须继承 DynamicMBean 接口，动态 Bean 会定义 MBeanAttributeInfo（属性信息）/ MBeanOperationInfo（方法信息）等，然后一起注册到 MBeanServer 中去，使用的时候通过MBServer 获取该 MBean, 拿到属性信息或方法信息，然后通过反射的方式去动态的改变属性，执行方法。

关于DynamicBean最基础的用法可以参考[动态MBean：DynamicMBean](http://damies.iteye.com/blog/51799)

后面的介绍假设你已经了解JMX了基本概念和使用方法，<!--more-->

### **Tomcat里的JMX**
![](/images/【Tomcat学习笔记】Tomcat中JMX的使用_2.png)

这是Tomcat中和JMX相关的一些关键接口和类，MBeanRegistration定义了注册过程中一些方法，但Tomcat大部分Mbean只是实现了该接口，方法是空的，什么都没做，可以忽略。

LifecycleMBeanBase 是 Tomcat 大部分组件的父类，用于管理生命周期和JMX注册。在 **initInternal** 的时候，会通过 **Registry(注册中心)** 将该自己注册到 MbeanServer 中去。重点看下这个注册中心，它维护了一个MeanServer(通过MBeanServerFactory创建的)和所有的ManagedBean, 这里的 ManagedBean 和我们通常说的 JMX 的MBean 不是一回事，ManagedBean 是 Tomcat 用来管理 MBean 的一个类，主要维护了 MBean 的一些meta data(元数据)，比如最关键的attributes和operations。真正的 MBean 是这里的 BaseModelMBean，它实现了 DynamicMBean 接口，通过关联的 ManagedBean 获取 MBean 的相关元数据。

Registry 通过两个Map 来保存所有 ManagedBean
* HashMap<String,ManagedBean> descriptors, key 是指定的描述符
* HashMap<String,ManagedBean> descriptorsByClass, key 是class name，比如 org.apache.catalina.core.StandardServer

这两个 Map 中的 ManagedBean 是 Registry 在初始化的时候通过 MbeansDescriptorsDigesterSource 或 MbeansDescriptorsIntrospectionSource 创建并放进去的。
* MbeansDescriptorsDigesterSource， 按照指定的 MBean 配置文件（比如 org/apache/catalina/ha/session/mbeans-descriptors.xml ），通过 digester 解析配置文件并创建 ManagedBean 并 Map
* MbeansDescriptorsIntrospectionSource，通过反射的形式获得类的相关元信息，创建 ManagedBean 并放入 Map

和[动态MBean：DynamicMBean](http://damies.iteye.com/blog/51799) 对比，Tomcat 用 ManagedBean、Registry、BaseModelMBean 这三个关键类 将所有 MBean 的创建和注册放一起完美的解决了。

### **如何使用Tomcat里的JMX**
只看代码可能还是云里雾里，实际用一下JMX，一切都茅塞顿开了。要使用 JMX, 需要 enable 该功能，在 Tomcat 启动参数里，加上  
```shell
-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9876 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=127.0.0.1
```
不解释了，看名字应该知道什么意思。然后启动 jconsole,
远程进程输入 service:jmx:rmi:///jndi/rmi://127.0.0.1:9876/jmxrmi
![500](/images/【Tomcat学习笔记】Tomcat中JMX的使用_3.png)
![500](/images/【Tomcat学习笔记】Tomcat中JMX的使用_4.png)
左侧导航栏可以看到有很多 MBean，每个 MBean 都有属性，操作 和 通知，通过这些就可以控制 Tomcat 了。

本质就是通过RMI（也可以是其他方式，比如HTTP）将相关信息 发给 Tomcat，Tomcat 收到后交给 JMXProxyServlet，JMXProxyServlet 通过 MBServer 找到该 MBean, 通过反射的方式去调用它的方法完成操作。

### **更友好的JMX**
我把 Tomcat JMX 的这些东西看完后，发现有两个问题
1. JMX 使用很繁琐，每个 MBean 都要创建 MBeanAttributeInfo， MBeanConstructorInfo， MBeanOperationInfo等信息，然后还有balababala一堆破事
2. jconsole 的界面真丑，用的不爽。

关于第一个问题，用 [Spring JMX](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/jmx.html) 可以省很多事。关于第二个问题，可以自己整个 Web 的页面用于监控和管理 JMX 的应用，[JmxInWeb](https://github.com/ynitq/JmxInWeb) 就弄了一个这样的功能，代码不多，里面对 JMX 的使用可以学习学习。由于没有用 MVC 框架，里面很多代码都是在做 MVC 的功能，如果用 Spring MVC，代码会精简很多。




##### .
** 以上皆是阅读源码 https://github.com/fdx321/tomcat8.0-source-research 所得 **


<style>
img[title="300"] {
  width:300px;
  width:300px;
  display: block;
}
img[title="500"] {
  width:500px;
  height:500px;
  display: block;
}
</style>
