title: 【Tomcat学习笔记】15-Connector
date: 2017-07-14 17:50:01
tags:
- Java
- Tomcat
---
### **Connector配置**
Connector 属于 StandardService 里的一个组件，可以在 server.xml 中配置，指定协议、端口、超时时间等。
```xml
<Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" /> 
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />                         
</Service>     
```

每个 Service 里可以配置多个 Connector. Tomcat 的 Connector 支持两种协议，HTTP 和 AJP. 关于这两种协议，官方文档如此说：
* *The HTTP connector is setup by default with Tomcat, and is ready to use. This connector features the lowest latency and best overall performance.*
* *When using a single server, the performance when using a native webserver in front of the Tomcat instance is most of the time significantly worse than a standalone Tomcat with its default HTTP connector, even if a large part of the web application is made of static files. If integration with the native webserver is needed for any reason, an AJP connector will provide faster performance than proxied HTTP. AJP clustering is the most efficient from the Tomcat perspective. It is otherwise functionally equivalent to HTTP clustering.*
<!--more-->

通过配置不同的 protocol 属性，可以使用不同的 ProtocalHandler. 配置HTTP/1.1时，默认会使用org.apache.coyote.http11.Http11NioProtocol. 也可以通过指定类名的方式来配置，比如
protocol="org.apache.coyote.http11.Http11Nio2Protocol".
针对 HTTP 和 AJP 两种协议，都有 BIO/NIO/NIO2/APR 四种处理方式。
* HTTP
    * org.apache.coyote.http11.Http11Protocol
    * org.apache.coyote.http11.Http11NioProtocol
    * org.apache.coyote.http11.Http11Nio2Protocol
    * org.apache.coyote.http11.Http11AprProtocol
* AJP
    * org.apache.coyote.http11.Ajp11Protocol
    * org.apache.coyote.http11.Ajp11NioProtocol
    * org.apache.coyote.http11.Ajp11Nio2Protocol
    * org.apache.coyote.http11.Ajp11AprProtocol

AJP 的处理我并没有去细看，但大的代码结构基本和 HTTP 的处理类似。后面重点看下 HTTP 连接的几种处理方式。

### **HTTP BIO，同步阻塞**
![400](/images/【Tomcat学习笔记】Connector-1.png)
使用的是Java BIO 技术，一个 Acceptor 线程负责创建连接，把建立好连接的 socket 给到 SocketProcessor, 然后把SocketProcessor 丢到线程池里去执行。经过一系列处理后最终到达 CoyoteAdapter. 线程模型即：
* Acceptor: 单线程
* SocketProcessor：线程池

### **HTTP NIO， 同步非阻塞**
![400](/images/【Tomcat学习笔记】Connector-2.png)
使用的是Java NIO 技术，一个 Acceptor 线程负责创建连接，把建立好连接的 socket 交给 一个Poller. 这里有多个Poller，每个Poller一个线程。Poller 负责把 socket 注册到 selector, 负责轮询 selector（*关于selector, 我当年给NetX协议栈写 BSD SOCKET 兼容接口的时候还亲自实现过，哈哈*）, 有数据可读的时候，就交给SocketProcessor丢到线程池里去执行。经过一系列处理后最终到达 CoyoteAdapter. 线程模型即：
* Acceptor: 单线程
* Poller：多个线程
* SocketProcessor: 线程池

### **HTTP NIO2，异步非阻塞**
![400](/images/【Tomcat学习笔记】Connector-3.png)
使用的是 Java NIO2(AIO) 技术，一个 Acceptor 线程(代码里 Acceptor 只支持创建多个的，通过变量 acceptorThreadCount 来控制)负责创建连接，把建立好连接的 socket 给到 SocketProcessor, 然后把SocketProcessor 丢到线程池里去执行。 线程模型即：
* Acceptor: 单线程
* SocketProcessor: 线程池

是不是看起来和 HTTP BIO 很像，但是这里的 read/write 用的是 NIO2 的异步非阻塞方式，即read的时候带个callback，等有OS有数据了，再来回调你的处理方法，效率比BIO高很多。

### **HTTP APR**
[APR](http://apr.apache.org/) ，讲真，没仔细研究过，就不瞎BB了。

关于BIO/NIO/NIO2(AIO)，已及网络编程的线程模型，更详细的可以看[Netty学习笔记](https://fdx321.github.io/2016/11/07/Netty%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/). 关于这几种模式的性能比较，网上也有很多测试数据。

### **代码结构**
如前面几张图所示，整个连接器的代码结构还是很清晰的。在 Tomcat 启动的时候，通过 Digest 解析 server.xml 配置的时候，会在 StandardService 里创建一些 Connector. Connector 也是实现生命周期接口的，在 StandardService 初始化和启动的时候，Connector 也会完成自己的初始化和启动操作，启动过程中，EndPoint 会去完成 bind 操作，然后启动 accept 线程。ProtocalHandler、EndPoint、ConnectionHandler、Processor、Adaptor 这几个角色互相配合，完成各自的工作。

最后，谢晞鸣在想，CoyotoAdapter 这个适配器之前的所有这些的网络处理，是不是都可以用Netty来实现呢，抽空想想看？


##### .
** 以上皆是阅读源码 https://github.com/fdx321/tomcat8.0-source-research 所得 **


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
img[title="500"] {
  width:500px;
  height:500px;
  display: block;
}
</style>
