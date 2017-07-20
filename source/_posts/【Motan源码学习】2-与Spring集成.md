title: 【Motan源码学习】2-与Spring集成
date: 2017-07-21 22:56:11
tags:
- Java
- Motan
---
motan-core#com.weibo.api.motan.config 这个 package 定义了一系列 Config 类，用于处理服务发布和服务引用的一些配置。
关于Config, Motan 支持三种使用方式：
* 编程 API
* 与 Spring 集成的 XML 配置
* 与 Spring 集成的 Annotation 配置
我们先来看下编程 API 的使用方式，然后看看 Motan 是如何与 Spring 集成的，本文假设你已经对 Spring 的启动过程和一些 hook 接口有基本的了解。

### **编程 API**
**1. 服务发布**
```java
ServiceConfig<MotanDemoService> motanDemoService = new ServiceConfig<MotanDemoService>();
// 设置接口及实现类
motanDemoService.setInterface(MotanDemoService.class);
motanDemoService.setRef(new MotanDemoServiceImpl());
// 配置服务的group以及版本号
motanDemoService.setGroup("motan-demo-rpc");
motanDemoService.setVersion("1.0");
// 配置ZooKeeper注册中心
RegistryConfig zookeeperRegistry = new RegistryConfig();
zookeeperRegistry.setRegProtocol("zookeeper");
zookeeperRegistry.setAddress("127.0.0.1:2181");
motanDemoService.setRegistry(zookeeperRegistry);
// 配置RPC协议
ProtocolConfig protocol = new ProtocolConfig();
protocol.setId("motan");
protocol.setName("motan");
motanDemoService.setProtocol(protocol);
motanDemoService.setExport("motan:8002");
// 服务发布
motanDemoService.export();
```
<!--more-->
**2. 服务引用**
```java
RefererConfig<MotanDemoService> motanDemoServiceReferer = new RefererConfig<MotanDemoService>();
// 设置接口及实现类
motanDemoServiceReferer.setInterface(MotanDemoService.class);
// 配置服务的group以及版本号
motanDemoServiceReferer.setGroup("motan-demo-rpc");
motanDemoServiceReferer.setVersion("1.0");
motanDemoServiceReferer.setRequestTimeout(2000);
// 配置ZooKeeper注册中心
RegistryConfig zookeeperRegistry = new RegistryConfig();
zookeeperRegistry.setRegProtocol("zookeeper");
zookeeperRegistry.setAddress("127.0.0.1:2181");
motanDemoServiceReferer.setRegistry(zookeeperRegistry);
// 配置RPC协议
ProtocolConfig protocol = new ProtocolConfig();
protocol.setId("motan");
protocol.setName("motan");
motanDemoServiceReferer.setProtocol(protocol);
// 使用服务
MotanDemoService service = motanDemoServiceReferer.getRef();
service.hello("motan");
```
### **与 Spring 集成的 XML 配置**
接下来看下上面这两段代码是如何通过XML来玩的.
**1. 服务发布**
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:motan="http://api.weibo.com/schema/motan"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
       http://api.weibo.com/schema/motan http://api.weibo.com/schema/motan.xsd">

    <!-- 业务具体实现类 -->
    <bean id="motanDemoServiceImpl" class="com.weibo.motan.demo.server.MotanDemoServiceImpl"/>
    <!-- 注册中心配置 使用不同注册中心需要依赖对应的jar包。如果不使用注册中心，可以把check属性改为false，忽略注册失败。-->
    <motan:registry regProtocol="zookeeper" name="registry" address="127.0.0.1:2181"/> 
    <!-- 协议配置。为防止多个业务配置冲突，推荐使用id表示具体协议。-->
    <motan:protocol id="demoMotan" default="true" name="motan"
                    maxServerConnection="80000" maxContentLength="1048576"
                    maxWorkerThread="800" minWorkerThread="20"/>
    <!-- 通用配置，多个rpc服务使用相同的基础配置. group和module定义具体的服务池。export格式为“protocol id:提供服务的端口”-->
    <motan:basicService export="demoMotan:8002"
                        group="motan-demo-rpc" accessLog="false" shareChannel="true" module="motan-demo-rpc"
                        application="myMotanDemo" registry="registry" id="serviceBasicConfig"/>
    <!-- 具体rpc服务配置，声明实现的接口类。-->
    <motan:service interface="com.weibo.motan.demo.service.MotanDemoService"
                   ref="motanDemoServiceImpl" export="demoMotan:8002" basicService="serviceBasicConfig">
    </motan:service>
</beans>

在基于Spring的项目中，引用这个配置文件即可完成服务的发布。那么 Motan 是怎么与Spring集成的呢？
Motan 自定义了一个 shcema motan.xsd，然后在 motan-core # META-INF/spring.handlers 中定义了 这个 schema 的 handler.
```sh
http\://api.weibo.com/schema/motan=com.weibo.api.motan.config.springsupport.MotanNamespaceHandler
```
MotanNamespaceHandler 中定义了 <motan:registry>  <motan:protocol> <motan:service> 等各个标签的Parser. 如下，Spring启动过程中会去调用该init方法，该方法会注册 registry、protocol 等标签的Parser， 等启动过程中解析到这些配置的时候，就会用 MotanBeanDefinitionParser 去解析这些配置，完成 ServiceConfig、ProtocolConfig、RegistryConfig 这些 Bean 的属性设置。
```java
public void init() {
    registerBeanDefinitionParser("service", new MotanBeanDefinitionParser(ServiceConfigBean.class, true));
    registerBeanDefinitionParser("protocol", new MotanBeanDefinitionParser(ProtocolConfig.class, true));
    registerBeanDefinitionParser("registry", new MotanBeanDefinitionParser(RegistryConfig.class, true));
}
```

在 *编程 API---服务发布* 中，最后还调用 motanDemoService.export() 完成了服务的发布。那么这里是怎么自动完成服务发布的呢？
ServiceConfigBean 实现了 Spring 的一些 hook 接口，当 Bean 初始化完成之后，Spring 会来调用该方法，完成服务发布:
```java
public void onApplicationEvent(ContextRefreshedEvent event) {
    if (!getExported().get()) {
        export();
    }
}
```

**2. 服务引用**
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:motan="http://api.weibo.com/schema/motan"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
       http://api.weibo.com/schema/motan http://api.weibo.com/schema/motan.xsd">

    <!-- 注册中心配置 使用不同注册中心需要依赖对应的jar包。-->
    <motan:registry regProtocol="zookeeper" name="registry" address="127.0.0.1:2181"/>
    <!-- motan协议配置 -->
    <motan:protocol default="true" name="motan" haStrategy="failover"
                    loadbalance="roundrobin" maxClientConnection="10" minClientConnection="2"/>
    <!-- 通用referer基础配置 -->
    <motan:basicReferer requestTimeout="200" accessLog="false"
                        retries="2" group="motan-demo-rpc" module="motan-demo-rpc"
                        application="myMotanDemo" protocol="motan" registry="registry"
                        id="motantestClientBasicConfig" throwException="false" check="true"/>
    <!-- 具体referer配置。使用方通过beanid使用服务接口类 -->
    <motan:referer id="motanDemoReferer"
                   interface="com.weibo.motan.demo.service.MotanDemoService"
                   connectTimeout="300" requestTimeout="300" basicReferer="motantestClientBasicConfig"/>
</beans>
```
在基于Spring的项目中， 自动注入 motanDemoReferer 后， 即可用 motanDemoReferer.func(xxx) 来发起远程调用了。配置的解析和服务发布中讲的类似。

在 *编程 API---服务发布* 中，最后还调用了 motanDemoServiceReferer.getRef() 来获得引用，这个过程会去做一个动态代理，在这里也是通过实现 Spring 的 hook 接口去自动实现的。

### **与 Spring 集成的 Annotation 配置**
motan-demo 模块中有使用样例，这里就不贴代码了。与Spring集成的切入点还是在 MotanNamespaceHandler：
```java
    public void init() {
        registerBeanDefinitionParser("annotation", new MotanBeanDefinitionParser(AnnotationBean.class, true));
    }
```
在使用的时候，除了在一些类/方法/属性上打上annotation之外，还需要在Spring配置文件中加上:
```xml
<motan:annotation package="xxx.xxx.xx"/>
```
在 Spring 启动过程中，解析到该 Element 的时候，就会实例化 AnnotationBean, 该 Bean 同样实现了很多 Spring 的 Hook 接口，会在启动过程中去 scan 这个 package 中存在的 annotation, 完成相应的操作。

### **总结**
从 Config 的角度看，Motan 支持 API、XML、Annotation 三种使用方式。与 Spring 集成的切入点在于，通过 motan-core 中 spring.schemas、spring.handlers 这两个配置文件指定自定义的 schema 和对应的 NamespaceHandler。NamespaceHandler 中会指定每个自定义标签的Parser 和对应的 Class. Parser 会完成配置的解析. 有些标签对应的Class实现了Spring的一些 hook 接口，会在 Spring 启动过程中自动完成一些初始化操作。 具体可以看 motan-springsupport 这个模块。



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

