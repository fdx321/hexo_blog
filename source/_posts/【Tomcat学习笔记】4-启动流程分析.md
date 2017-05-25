title: 【Tomcat学习笔记】4-启动流程分析
date: 2017-05-18 13:35:11
tags:
- Java
- Tomcat
---
*讲真，假如你不幸看到这里了，别往下看了，写得一坨 Shit. 只有我自己能看懂的笔记。讲道理，这种流程，真是一图剩千言，然。。。画图好累的。*
### **1. 执行 Bootstrap 类的 static 代码块**
主要功能是确定 catalinaHome 和 catalinaBase 两个目录，如果有配置 "catalina.home"/“catalina.base” 这两个 property，就以配置的为准，没有配置就走 fall back，用默认的目录。那么 home 和 base 分别只哪两个目录呢，是用来干嘛的呢。先来看下官方的解释：
* **catalina.home** : the tomcat product installation path
* **catalina.base** : the tomcat instance installation path
看了是不是还是一脸闷逼？我也是。OK，看下我们的工程，里面有个目录
![](/images/【Tomcat学习笔记】启动流程分析_1.png)<!--more-->
一般情况下，catalina.home 和 catalina.base 指向同一个这样的目录. 但有时候我们需要在一台机器上运行多个 Tomcat 实例，没必要安装多个 Tomcat. 只需要配置多个工作目录，共享一个安装目录就可以了。Tomcat 运行需要 conf、lib、logs、。。等这些目录，对每个实例，可以单独弄个目录，在里面创建好 conf、lib、logs 这些目录，然后配置好各自的 catalina.base 就可以了。
*完了，扯着扯着就跑题了，感觉写不完了，（这里我是想加个表情的，于是发现 hexo 是可以换个支持表情的 markdown 渲染器的，然而我懒得玩了）*

### **2. 执行 Bootstrap 类的 main 方法**
这个方法里主要是调用 init 完成各种 classLoader（classLoader这个东西还是有点意思的，后面会有笔记具体分析） 的创建，实例化 Catalina ，然后根据入参来执行 Catalina 的 start、stop、... 等方法。

### **3. Catalina 的 start 方法**
为了简洁，代码有很大的删减或修改。懒得画图，只好以这种代码的方式来看流程了。
```java
public void start() {
  // load 方法会完成 StandardServer 的创建和初始化，然后调用 start 方法启动
  load();
  getServer().start();
}


public void load() {
  Digester digester = createStartDigester();

  File file = configFile();
  InputStream  inputStream = new FileInputStream(file);
  InputSource inputSource = new InputSource(file.toURI().toURL().toString());
  inputSource.setByteStream(inputStream);
  digester.push(this);
  digester.parse(inputSource);

  getServer().setCatalina(this);
  getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
  getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());
  getServer().init();
}
```
load方法的主要操作就是创建一个Digester，用该 digester 解析 server.xml，在解析的过程中，digester 会根据配置的 rule 在解析到某些节点的时候创建对应的组件，比如 StandardServer、StandardService、Listener、... 解析完成后，StandardServer 已经完成了实例化，设置一些属性后，调用init完成初始化。


在（3）中，整个Tomcat已经启动了，但中间有两个关键的过程还没有分析
* Server、Service、Engine、....这些组件是如何被创建的，它们之间的关系又是如何关联起来的
* 它们是如何一层一层完成初始化和启动的。


### **4. Server、Service、Engine、....这些组件是如何被创建的，它们之间的关系又是如何关联起来的**
这个过程主要是通过 Digester ( https://commons.apache.org/proper/commons-digester/ )来实现的. 下面是代码里给 Digester 配置的一些规则。
```java
  digester.addObjectCreate("Server","org.apache.catalina.core.StandardServer","className");
  digester.addSetProperties("Server");
  digester.addSetNext("Server","setServer","org.apache.catalina.Server");

  digester.addObjectCreate("Server/Listener",null,"className");
  digester.addSetProperties("Server/Listener");
  digester.addSetNext("Server/Listener","addLifecycleListener","org.apache.catalina.LifecycleListener");

  digester.addObjectCreate("Server/Service","org.apache.catalina.core.StandardService","className");
  digester.addSetProperties("Server/Service");
  digester.addSetNext("Server/Service","addService","org.apache.catalina.Service");

  digester.addObjectCreate("Server/Service/Listener",null,"className");
  digester.addSetProperties("Server/Service/Listener");
  digester.addSetNext("Server/Service/Listener","addLifecycleListener","org.apache.catalina.LifecycleListener");
```
有了这些规则之后，在执行parse的过程中，就会根据这些规则来做一些操作。比如，解析到server.xml中的 <Service> 节点的时候
```xml
<Server port="8005" shutdown="SHUTDOWN">
  ...
  <Service name="Catalina">
  ...
  ...  
```
就会根据下面的规则，先创建一个**org.apache.catalina.core.StandardService**对象，然后调用 StandardServer 的 addService 方法将他们关联起来。其他的组件也是按类似的方式完成创建和关联的。
```java
digester.addObjectCreate("Server/Service","org.apache.catalina.core.StandardService","className");
digester.addSetProperties("Server/Service");
digester.addSetNext("Server/Service","addService","org.apache.catalina.Service");
```

### **5.它们是如何一层一层完成初始化和启动的。**
已StandardServer为例，
![300](/images/【Tomcat学习笔记】启动流程分析_2.png)
调用**getServer().init()**方法后，会进入 LifecycleBase#init，这个方法主要是设置生命周期以及触发相应的事件，之后会调用 StandardServer#init()，它首先会调用 LifecycleMBeanBase#init 把自己注册到MBeanServer中(JMX后面会具体说)，然后完成 StandardServer 自己初始化需要做的事情，最后在遍历数组，依次调用各个service的init方法。 如下面的时序图所示：
<img src="/images/【Tomcat学习笔记】启动流程分析_2.svg"/>
到servie的init方法后，也是类似的流程，一层一层往下做。


1~5 是一个大概的启动过程，后面会再更具体的分析每个组件在 init 和 start 的过程中做了什么事情。

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
  height:150px;
  display: block;
}
</style>
