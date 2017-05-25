title: 【Tomcat学习笔记】6-shutdown过程分析
date: 2017-05-20 18:04:23
tags:
- Java
- Tomcat
---
### **缘起**
在看 BootStrap 类代码的时候，突然想到一个问题，关闭 Tomcat 的入口也是从 BootStrap 的 main 方法开始的，但是从 main 方法开始执行就会另起了一个进程，它是如何关掉另一个进程的 Tomcat 的呢，不可能是简单的 kill 一个 pid, 因为很多组件都有 destroy 方法要执行。
于是我决定研究一下 shutdown 的过程是如何执行的。<!--more-->

### **Catalina#stopServer**
这里代码有所删减.
```java
public void stopServer(String[] arguments) {
  Digester digester = createStopDigester();
  File file = configFile();
  InputSource is = new InputSource(file.toURI().toURL().toString());
  is.setByteStream(new FileInputStream(file));
  digester.push(this);
  digester.parse(is);

  s = getServer();
  if (s.getPort()>0) {
    try {
      Socket socket = new Socket(s.getAddress(), s.getPort());
      OutputStream stream = socket.getOutputStream();
      String shutdown = s.getShutdown();
      for (int i = 0; i < shutdown.length(); i++) {
        stream.write(shutdown.charAt(i));
      }
      stream.flush();
    }
  }
}
```
首先创建一个 digester 用于解析 server.xml, 这个 digester 比较简单，只配置了 Server 那一层，用于创建 Server，获取 address 和 port. 得到 address 和 port 后，就创建 socket，发送一个 shutdown 命令。port 和 command 配置在这里的, 不配置的话，默认是 8005 和 SHUTDOWN。
```xml
<Server port="8005" shutdown="SHUTDOWN">
```

### **Catalina#start**
```java
public void start() {
    ...
    if (await) {
        await();
        stop();
    }
}
```
这是 Catalina#start 方法的最后几行代码，在 Tomcat 启动后，await 方法会创建一个 Socket 用于监听 8005 端口的请求，如果是 shutdown 命令，就跳出循环，然后执行 stop 方法，否则就一直等待。具体可以看 StandardServer#await， 这里就不 copy 代码了。


总结， Tomcat 在启动完成之后会创建一个 socket 用于监听 shutdown 请求，关闭 Tomcat 时会从 server.xml 中获取 address 和 port，然后发送 shutdown 命令，Server 接收到请求后，结束等待，执行 stop 命令。到此，我的疑问得到了解答。

### **使用 kill pid 命令关闭Tomcat**

```java
public void start() {
  ...
  if (useShutdownHook) {
    if (shutdownHook == null) {
      shutdownHook = new CatalinaShutdownHook();
    }
    Runtime.getRuntime().addShutdownHook(shutdownHook);

    ...
  }
  ...
}
```
这是 Catalina#start 方法中的一段代码，可以看到，在Tomcat启动的时候，注册了shutdownHook. 使用 kill 命令关闭 JVM 进程的时候，会先执行该 hook 方法，再结束JVM进程。该 hook 方法里会去调用 stop 方法完成Tomcat的关闭。


##### .
** 以上皆是阅读源码 https://github.com/fdx321/tomcat8.0-source-research 所得 **
