title: 【Tomcat学习笔记】整体架构
date: 2017-05-15 12:58:05
tags:
- Java
- Tomcat
---
Tomcat的整体架构其实和 server.xml 这个配置文件是可以对应起来的。这是一个最简单的但是能用的 server.xml
```xml
<?xml version='1.0' encoding='utf-8'?>
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <Service name="Catalina">

    <Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />

    <Engine name="Catalina" defaultHost="localhost">
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>

      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">

        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />

      </Host>
    </Engine>
  </Service>
</Server>

```

下面是Tomcat的整体架构<!--more-->
<img src="/images/【Tomcat学习笔记】整体架构_1.svg"/>


代码和 server.xml 这个配置文件一样，也是一样的层级结构，Server > Service > Engine > Host > Context > Wrapper 是依次包含的关系。Tomcat 里也定义了同样名字的 Interface, 这些接口的默认实现是 StandardServer > StandardService > StandardEngine > StandardHost > StandardContext > StandardWrapper。这个类图是Tomcat最主要的一个结构：
<img src="/images/【Tomcat学习笔记】整体架构_2.svg"/>

关于这个架构图，有几点需要说明，但细节可能需要后面写具体的笔记分析（立个flag在这，打脸）
* Server 和 Service 并不是 Container（容器）, 它们不像其它容器那样有Valve
* 每个组件都可以配置很多 Listener, 这些 Listener 都实现了 LifecycleListener 接口，每个组件在经历生命周期的每个阶段的时候都会去循环通知这些 Listener
* 每个组件都实现了生命周期 Lifecycle 接口 (关于组件的生命周期，后面应该可以单独整理一个笔记的)
* Context 和 Wrapper 在 server.xml 中并没有配置（要配置应该也是可以的吧，待确认），
![400](/images/【Tomcat学习笔记】整体架构_3.jpg)
【图片来源】( http://gearever.iteye.com/blog/1532822 )， Host对应www.mydomain.com那一层，在 Tomcat 启动之前就知道的。Context 对应 app 那一层，这个 app 是 Tomcat 在启动过程中，扫描 catalina-home/webapps（默认是这个目录） 这个目录的时候才知道有哪几个应用需要部署，才创建对应的 Context, 所以这个是可以不在 server.xml 中配置的。可以理解成一个 Java 应用 对应一个 Context. 那么 Wrapper 也是在扫描了待部署应用里面的内容后才创建的。Engine 和 Host 则是启动过程中通过解析 server.xml 的时候创建的。
* Engine、Host、Context、Wrapper 四种 Container 都可以配置 **Valve**，即使不配置，每个 Container 代码里都有默认的Valve（StandardEngineValve, StandardHostValve ...）是处理请求的时候必须经过的。关于 Pipeline 和 Valve，就是一个水管中间有多个阀门，每个数据流过来都在阀门的地方被处理一下。 四个容器的Pipeline串起来，可以用张图来描述一下：
<img src="/images/【Tomcat学习笔记】整体架构_4.svg"/>
实际代码中并没有这样一个 Pipeline 的数据结构或者类， 这只是一个抽象概念，代码里就是类似于链表的形式，getNext().getNext()这样.请求request进到 Engine 后，会经过几个Valve的处理，然后会选择一个 Host，进入它的 Valve 链里进行处理，后面也是按这种方式进行，响应数据最后也是按这个路径原路返回的。和现实中的 Pipeline 最大的不同是，现实中的水管，水是分流到几个细的水管，这里不是，这里是选择一根 Pipeline往下走。


#### Reference

<style>
img[title="400"] {
  width:400px;
  width:400px;
  display: block;
}
img[title="500"] {
  width:500px;
  height:150px;
  display: block;
}
</style>
