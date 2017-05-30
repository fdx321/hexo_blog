title: 【Tomcat学习笔记】9-ClassLoader
date: 2017-05-30 18:03:39
tags:
- Java
- Tomcat
---
*一直在入门，但从未精通，所以不敢说深入分析 XXX，顶多就是一个学习笔记，但是是原创，转载请注明出处，好吧，说的好像有人会看一样。*
*说明，为了简洁，这里贴的代码都有所删减。*

先来张图：
<img src="/images/【Tomcat学习笔记】ClassLoader_1.svg"/>

### **ClassLoader基础**
Java 自带了三个类加载器，BootstrapClassLoader、ExtClassLoader 和 AppClassLoader，它们之间的层级关系如图，上面的 ClassLoader 是下面 ClassLoader 的 parent.不同的 ClassLoader 有不同的职责：
* BootstrapClassLoader, 引导类加载器，加载JDK最核心的类，比如 rt.jar, java.lang.String, java.lang.Integer 这些类都在这个 JAR 中
* ExtClassLoader, 扩展类加载器，加载 jre/lib/ext 目录下的类
* AppClassLoader, 系统类加载器，加载应用程序 classpath 目录下的所有jar和class文件

如果想看 ClassLoader 具体加载了哪些类，可以 getURLs, 然后将它们打印出来
```java
URL[] urls = sun.misc.Launcher.getBootstrapClassPath().getURLs();
URL[] urls = ((URLClassLoader)Thread.currentThread().getContextClassLoader()).getURLs();
```
<!--more-->

应用程序也可以自己 创建 ClassLoader, 指定它的 parent, 指定它要加载的 类。那么，类的加载过程是怎样的呢，这里有个**双亲委派模型(parent delegation model)** 的概念。即，某 ClassLoader XXX 加载一个类的时候，一直向上委托，直到最顶端，然后开始加载，如果顶端找不到该类，再依次往下，最后回到 XXX , 如果 XXX 自己还是找不到该类，就报**ClassNotFoundException**. 双亲委派模型这种机制解决了什么问题呢，如果不用这种机制，用户可以自己定义一个JDK中的类(package也要一样)替换掉JDK中类。（这里有个问题，如果一个类已经被系统加载过了，再次加载的时候是会报错呢，还是以第一次加载的为准呢，还是以最后一次加载的为准呢？你这么聪明，肯定不用我说了）。上面大概就是 Java 自带 ClassLoader 的一些关键知识了，主要就是一个层级关系和双亲委托模型。

### **Tomcat的三大ClassLoader**
So, 为什么 Tomcat 里要自定义 ClassLoader 呢，先来考虑一个问题：一个Tomcat 部署两个应用，App1 和 App2, App1 里定义了一个 com.fdx.AAA 类，App2 也定义了一个 com.fdx.AAA 类，但是里面的实现是不一样的，如果不自定义 ClassLoader,
而都用 AppClassLoader 来加载的话，你让它加载哪一个呢，一个 ClassLoader 是不能加载两个一样的类的。所以，ClassLoader 最重要的一个功能就是 类隔离。

接下来看看 Tomcat 是如何弄的. Bootstrap#initClassLoaders 方法中会去创建 commonLoader(对应配置common.loader)、catalinaLoader(对应配置server.loader) 和 sharedLoader(对应配置shared.loader)，每个 loader 要加载哪些路径下载的类会在 conf/catalina.properties 中配置，默认配置是:
```properties
common.loader="${catalina.base}/lib","${catalina.base}/lib/*.jar","${catalina.home}/lib","${catalina.home}/lib/*.jar"
server.loader=
shared.loader=
```
在server.loader和shared.loader为空的情况下，这三个其实是同一个 ClassLoader, 不为空的情况下，commonLoader是它们的parent. 所以我图中将它们画在了一个层级。

### **Tomcat的WebappClassLoader创建过程**

WebappClassLoader 和 StandardContext 是一一对应的，StandardContext 和 应用又是一一对应的。每个 WebappClassLoader 只加载对应应用的类，这样应用之间的类就隔离了。来看 StandardContext 启动过程中的一段代码。
```java
protected synchronized void startInternal() throws LifecycleException {
  if (getLoader() == null) {
      WebappLoader webappLoader = new WebappLoader(getParentClassLoader());
      webappLoader.setDelegate(getDelegate());
      setLoader(webappLoader);
  }

  ClassLoader oldCCL = bindThread();
  Loader loader = getLoader();
  if ((loader != null) && (loader instanceof Lifecycle))
      ((Lifecycle) loader).start();
  unbindThread(oldCCL);
  oldCCL = bindThread();
}
```
3、4、5 行 创建了一个 WebappLoader（这个并不是ClassLoader，它的作用是来管理WebappClassLoader），设置 delegate(delegate为true是，就是正常的加载顺序，为false，就不是标准的双亲委派了，而是WebappClassLoader自己先加载，加载不到再往上委派，默认是false)。11行启动了 WebappLoader, WebappLoader#startInternal 方法里会去创建 WebappClassLoader，设置它的 classPath, 安全权限，启动WebappClassLoader（Tomcat里基本上是个东西都实现了Lifecycle接口）。最后通过ubind, bind操作将当前线程的ClassLoader 设置为 WebappClassLoader。 WebappClassLoader的启动过程值得关注下，这里将应用的 /WEB-INF/classes 和 /WEB-INF/lib 加到了 localRepositories 中，这两个目录一个是打包后应用自身代码所在目录，一个是应用依赖的jar所以的目录。
```java
public void start() throws LifecycleException {
    WebResource classes = resources.getResource("/WEB-INF/classes");
    if (classes.isDirectory() && classes.canRead()) {
        localRepositories.add(classes.getURL());
    }
    WebResource[] jars = resources.listResources("/WEB-INF/lib");
    for (WebResource jar : jars) {
        if (jar.getName().endsWith(".jar") && jar.isFile() && jar.canRead()) {
            localRepositories.add(jar.getURL());
            jarModificationTimes.put(
                    jar.getName(), Long.valueOf(jar.getLastModified()));
        }
    }
}
```
默认情况都是使用lWebappClassLoader， 在 Tomcat 8+ 中，还加入了 ParallelWebappClassLoader，可以并行的加载类。用并行加载，需要在conf/context.xml中配置loaderClass，但是一般情况下，这种需求不大，得要多大的项目才有那么多类需要并行来加载啊。关于并行类加载器，有兴趣可以看 [Tomcat8+引入的并发ClassLoader](http://www.10tiao.com/html/308/201701/2650076391/1.html)

### **Tomcat的WebappClassLoader加载类的过程**
WebappClassLoaderBase#loadClass 方法实现了整个加载过程，主要有以下操作
* 检查该类是否有被加载过，如果加载过了，直接返回
* 使用 system classLoader 加载该类，如果加载到了，直接返回。（这一步主要是为了防止应用实现JDK里的类）
* 用SecurityManager进行安全检查，如果不通过，就抛出ClassNotFoundException, 这个安全过滤机制，是可以配置的，后面有空在细说
* 接下来有两个分支，如果delegate是true, 则先用parent加载，加载不到再自己加载。如果是false,则自己先加载，加载不到再用parent加载

那么 WebappClassLoader 自己在加载类的时候是如何绕过双亲委派机制的呢，代码好多，说不清楚，自己看去吧。


最后，留个问题，**为什么Tomcat要打破这种双亲委派机制呢？**

##### .
** 以上皆是阅读源码 https://github.com/fdx321/tomcat8.0-source-research 所得 **

### **参考文献**
* [图解Tomcat类加载机制](http://www.cnblogs.com/xing901022/p/4574961.html?utm_source=tuicool)
* [深入分析Java ClassLoader原理](http://blog.csdn.net/xyang81/article/details/7292380)
* [配置Tomcat的Loader组件（Nested Component之三）](http://www.10tiao.com/html/308/201701/2650076390/1.html)
* [Tomcat深入研究文章之十一(Tomcat classloader源码分析)](http://www.10tiao.com/html/308/201603/402159117/1.html)
* [Tomcat深入研究文章之十二(WebappClassLoader加载过程)](http://www.10tiao.com/html/308/201603/402165779/1.html)
* [Tomcat8+引入的并发ClassLoader](http://www.10tiao.com/html/308/201701/2650076391/1.html)
