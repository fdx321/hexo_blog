title: 【Tomcat学习笔记】7-分析各个组件的init和start
date: 2017-05-21 13:52:41
tags:
- Java
- Tomcat
---
*你TM写这种东西不烦吗？我也很无奈啊，太懒，只能靠写博客、吹牛逼驱动自己看代码啊。*

下面分析每个组件时，只分析它初始化自己的那部分逻辑，其它各种如何调用子组件的逻辑就不一一赘述了。

### StandardServer#initInternal
```java
if (getCatalina() != null) {
    ClassLoader cl = getCatalina().getParentClassLoader();
    // Walk the class loader hierarchy. Stop at the system class loader.
    // This will add the shared (if present) and common class loaders
    while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
        if (cl instanceof URLClassLoader) {
            URL[] urls = ((URLClassLoader) cl).getURLs();
            for (URL url : urls) {
                if (url.getProtocol().equals("file")) {
                    try {
                        File f = new File (url.toURI());
                        if (f.isFile() &&
                                f.getName().endsWith(".jar")) {
                            ExtensionValidator.addSystemResource(f);
                        }
                    } catch (URISyntaxException e) {
                        // Ignore
                    } catch (IOException e) {
                        // Ignore
                    }
                }
            }
        }
        cl = cl.getParent();
    }
}
```
<!--more-->
从 Catalina 的 parentClassLoader 开始，向上一直遍历到 ExtClassLoader，把它们会加载的 jar 包都用ExtensionValidator记录下来，后面再 StandardContext 启动的时候，会用 ExtensionValidator 来校验 StandardContext 对应的 Web App 依赖的一些 jar 包是否已经被加进来了。

### ContainerBase#initInternal
StandardEngine,StandardHost,StandardContext,StandardWrapper 四种容器本身的 initInternal 没有什么操作，主要都是调用ContainerBase#initInternal.
```java
protected void initInternal() throws LifecycleException {
    BlockingQueue<Runnable> startStopQueue = new LinkedBlockingQueue<>();
    startStopExecutor = new ThreadPoolExecutor(
            getStartStopThreadsInternal(),
            getStartStopThreadsInternal(), 10, TimeUnit.SECONDS,
            startStopQueue,
            new StartStopThreadFactory(getName() + "-startStop-"));
    startStopExecutor.allowCoreThreadTimeOut(true);
    super.initInternal();
}
```
这里主要就是创建一个线程池，用来处理子容器的 start 和 stop. 线程的数量 startStopThreads 默认是 1，如果配置的值小于等于0，则线程数为 Runtime.getRuntime().availableProcessors() + startStopThreads。如果部署了多个应用，配置多个线程可以并行部署，加快启动速度。

### ContainerBase#startInternal
上一节 init 时候创建的线程池，在这里 start 的时候就派上用场了。将 child 的 start 任务提交到线程池里。
```java
@Override
protected synchronized void startInternal() throws LifecycleException {
    ...
    Container children[] = findChildren();
    List<Future<Void>> results = new ArrayList<>();
    for (int i = 0; i < children.length; i++) {
        results.add(startStopExecutor.submit(new StartChild(children[i])));
    }
    ...
}
```

**但是这里有个问题**，当 StandardHost 执行到这段代码时，它并没有 child, 这个时候 Context 并没有创建。 So， Context 以及 后面的 Wrapper 是如何创建和初始化的呢？谢晞鸣带着疑问继续翻着代码。。。。


##### .
** 以上皆是阅读源码 https://github.com/fdx321/tomcat8.0-source-research 所得 **
