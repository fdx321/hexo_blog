title: 【Tomcat学习笔记】12-Tomcat的Mapper机制
date: 2017-06-29 21:18:31
tags:
- Java
- Tomcat
---
*说明，为了简洁，这里贴的代码可能有所删减。*

[【Tomcat学习笔记】2-整体架构](https://fdx321.github.io/2017/05/16/%E3%80%90Tomcat%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E3%80%912-%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84/#more) 中有介绍过 Tomcat 中 Engine、Host、Context、Wrapper 这四个组件的关系。还是以 http://www.mydomain.com/app/index.html 为例。

![300](/images/【Tomcat学习笔记】整体架构_3.jpg)
 当我们访问这个网址的时候，Tomcat 如何一步步路由找到对应的 Host、Context、Wrapper 的呢？Tomcat 中维护了一个 **Mapper**，在收到 Http 请求后，会去 Mapper 中查找对应的 Host、Context 和 Wrapper，放到 request 的 **MappingData** 中一路带下去。


### **Mapper的构建过程** <!--more-->
![](/images/【Tomcat学习笔记】Tomcat的Mapper机制_1.png)
Mapper 中维护了一个 MappedHost 列表，MappedHost、MappedContext、ContextVersion、MappedWrapper 这四者也是依次包含的关系。Mapper 的构建过程是在 **MapperListener** 中完成的， MapperListener 实现了 Tomcat 的生命周期接口，在 StandardService 启动的时候会调用 MapperListener 的 start 方法，这里会去构建 Mapper. MapperListener 也实现了 ContainerListener 和 LifecycleListener 接口，在发生容器事件和生命周期事件的时候，会往 Mapper 中添加或删除一些映射。拿下面这段代码为例，在监听到 ADD_CHILD_EVENT（新增了一个子容器） 事件后，会递归将自己添加到该容器以及该容器的所有子容器的监听器列表中去，然后根据该容器的类型，执行registerHost、registerContext 或者 registerWrapper 操作。

```java
public void containerEvent(ContainerEvent event) {
    if (Container.ADD_CHILD_EVENT.equals(event.getType())) {
        Container child = (Container) event.getData();
        addListeners(child);
        if (child.getState().isAvailable()) {
            if (child instanceof Host) {
                registerHost((Host) child);
            } else if (child instanceof Context) {
                registerContext((Context) child);
            } else if (child instanceof Wrapper) {
                if (child.getParent().getState().isAvailable()) {
                    registerWrapper((Wrapper) child);
                }
            }
        }
    }
}
```

So, MapperListener 主要做了这些事情.
1. 在自己的生命周期方法中构建Mapper(start), 从所有容器的监听器列表中删除自己(stop)
2. 监听 Container Event，往 Mapper 中添加或移除映射
3. 监听 Lifecycle Event，往 Mapper 中添加或移除映射

### **registerHost**
org.apache.catalina.mapper.MapperListener#registerHost
```java
private void registerHost(Host host) {
    String[] aliases = host.findAliases();
    //1. 往Mapper中添加Host
    mapper.addHost(host.getName(), aliases, host); 
    //2. 遍历注册所有Context, 由于容器之间的层级关系，
    //在 Tomcat 中经常看到这样的代码，或者是一些递归的代码依次去处理所有容器
    for (Container container : host.findChildren()) {
        if (container.getState().isAvailable()) {
            registerContext((Context) container);
        }
    }
}
```

org.apache.catalina.mapper.Mapper#addHost
```java
public synchronized void addHost(String name, String[] aliases,
                                 Host host) {
    //Mapper中维护的 MappedHost 数组是一个有序的数组，按照  name 进行排序的。      
    //insertMap 会在 hosts 中找到正确的位置，插入新的 host
    MappedHost[] newHosts = new MappedHost[hosts.length + 1];
    MappedHost newHost = new MappedHost(name, host);
    if (insertMap(hosts, newHosts, newHost)) {
        hosts = newHosts;
    } else {
        // 没有插入成功，因为 hosts中已经有了同样name的host
        MappedHost duplicate = hosts[find(hosts, name)];
        if (duplicate.object == host) {
            newHost = duplicate;
        } else {
            return;
        }
    }
    // 构建别名host列表
    List<MappedHost> newAliases = new ArrayList<>(aliases.length);
    for (String alias : aliases) {
        MappedHost newAlias = new MappedHost(alias, newHost);
        if (addHostAliasImpl(newAlias)) {
            newAliases.add(newAlias);
        }
    }
    newHost.addAliases(newAliases);
}
```
### **registerContext**
org.apache.catalina.mapper.MapperListener#registerContext
```java
private void registerContext(Context context) {
    String contextPath = context.getPath();
    if ("/".equals(contextPath)) {
        contextPath = "";
    }
    Host host = (Host)context.getParent();
    WebResourceRoot resources = context.getResources();
    String[] welcomeFiles = context.findWelcomeFiles();
    
    // 准备context 下的所有 wrapper 信息
    List<WrapperMappingInfo> wrappers = new ArrayList<>();
    for (Container container : context.findChildren()) {
        prepareWrapperMappingInfo(context, (Wrapper) container, wrappers);
        if(log.isDebugEnabled()) {
            log.debug(sm.getString("mapperListener.registerWrapper",
                    container.getName(), contextPath, service));
        }
    }
    // 添加 contextVersion， 在这里面会将该context下的所有wrapper映射信息也构建好
    mapper.addContextVersion(host.getName(), host, contextPath,
            context.getWebappVersion(), context, welcomeFiles, resources,
            wrappers);
}
```
org.apache.catalina.mapper.Mapper#addContextVersion
```java
public void addContextVersion(String hostName, Host host, String path,
        String version, Context context, String[] welcomeResources,
        WebResourceRoot resources, Collection<WrapperMappingInfo> wrappers) {
    //1. 找到对应的MappedHost，没有就添加到hosts中去        
    MappedHost mappedHost  = exactFind(hosts, hostName);
    if (mappedHost == null) {
        addHost(hostName, new String[0], host);
        mappedHost = exactFind(hosts, hostName);
        if (mappedHost == null) {
            log.error("No host found: " + hostName);
            return;
        }
    }
    if (mappedHost.isAlias()) {
        log.error("No host found: " + hostName);
        return;
    }
    // 比如springmvc-demo这个app，path为/springmvc-demo，slash个数为1
    int slashCount = slashCount(path);
    synchronized (mappedHost) {
        //构建ContextVersion
        ContextVersion newContextVersion = new ContextVersion(version,
                path, slashCount, context, resources, welcomeResources);
        //往ContextVersion中添加各个wrapper
        if (wrappers != null) {
            addWrappers(newContextVersion, wrappers);
        }
        //将构建好的ContextVersion添加到MapperHost的contextList中去
        ContextList contextList = mappedHost.contextList;
        MappedContext mappedContext = exactFind(contextList.contexts, path);
        if (mappedContext == null) {
            //通常都是走这个分支
            mappedContext = new MappedContext(path, newContextVersion);
            ContextList newContextList = contextList.addContext(
                    mappedContext, slashCount);
            if (newContextList != null) {
                updateContextList(mappedHost, newContextList);
                contextObjectToContextVersionMap.put(context, newContextVersion);
            }
        } else {
            //对于一个app有不同version的，首次之后的添加会走这个分支
            ContextVersion[] contextVersions = mappedContext.versions;
            ContextVersion[] newContextVersions = new ContextVersion[contextVersions.length + 1];
            if (insertMap(contextVersions, newContextVersions,
                    newContextVersion)) {
                mappedContext.versions = newContextVersions;
                contextObjectToContextVersionMap.put(context, newContextVersion);
            } else {
                int pos = find(contextVersions, version);
                if (pos >= 0 && contextVersions[pos].name.equals(version)) {
                    contextVersions[pos] = newContextVersion;
                    contextObjectToContextVersionMap.put(context, newContextVersion);
                }
            }
        }
    }
}
```

### **registerWrapper**
Tomcat 的 wrapper 映射是完全符合 [Servlet 规范](http://download.oracle.com/otn-pub/jcp/servlet-3.0-fr-eval-oth-JSpec/servlet-3_0-final-spec.pdf)的，第12章规定了如何mapping request to servlet.
![](/images/【Tomcat学习笔记】Tomcat的Mapper机制_2.png)

简单来说，就是路由的时候优先选择最精确匹配的(后面再看路由过程的时候会具体分析)，下面这段添加wrapper mapping的代码完美的实现了 12.2 的 spec. 根据 path 的结构，选择将 wrapper 添加到 wildcardWrappers/extensionWrappers/defaultWrapper/exactWrappers 中去。
```java
protected void addWrapper(ContextVersion context, String path,
        Wrapper wrapper, boolean jspWildCard, boolean resourceOnly) {
    synchronized (context) {
        if (path.endsWith("/*")) {
            // Wildcard wrapper，路劲匹配
            String name = path.substring(0, path.length() - 2);
            MappedWrapper newWrapper = new MappedWrapper(name, wrapper,
                    jspWildCard, resourceOnly);
            MappedWrapper[] oldWrappers = context.wildcardWrappers;
            MappedWrapper[] newWrappers = new MappedWrapper[oldWrappers.length + 1];
            if (insertMap(oldWrappers, newWrappers, newWrapper)) {
                context.wildcardWrappers = newWrappers;
                int slashCount = slashCount(newWrapper.name);
                if (slashCount > context.nesting) {
                    context.nesting = slashCount;
                }
            }
        } else if (path.startsWith("*.")) {
            // Extension wrapper，扩展名匹配
            String name = path.substring(2);
            MappedWrapper newWrapper = new MappedWrapper(name, wrapper,
                    jspWildCard, resourceOnly);
            MappedWrapper[] oldWrappers = context.extensionWrappers;
            MappedWrapper[] newWrappers =
                new MappedWrapper[oldWrappers.length + 1];
            if (insertMap(oldWrappers, newWrappers, newWrapper)) {
                context.extensionWrappers = newWrappers;
            }
        } else if (path.equals("/")) {
            // Default wrapper，默认匹配
            MappedWrapper newWrapper = new MappedWrapper("", wrapper,
                    jspWildCard, resourceOnly);
            context.defaultWrapper = newWrapper;
        } else {
            // Exact wrapper，精确匹配
            final String name;
            if (path.length() == 0) {
                // Special case for the Context Root mapping which is
                // treated as an exact match
                name = "/";
            } else {
                name = path;
            }
            MappedWrapper newWrapper = new MappedWrapper(name, wrapper,
                    jspWildCard, resourceOnly);
            MappedWrapper[] oldWrappers = context.exactWrappers;
            MappedWrapper[] newWrappers = new MappedWrapper[oldWrappers.length + 1];
            if (insertMap(oldWrappers, newWrappers, newWrapper)) {
                context.exactWrappers = newWrappers;
            }
        }
    }
}
```

可以看到有些地方有统计slashCount，那么这个 count 会有什么用呢？我带着问题看明白了，然而我懒得说了。。。。额。。。
```java
int slashCount = slashCount(newWrapper.name);
if (slashCount > context.nesting) {
    context.nesting = slashCount;
}
```

### **请求路由过程**
贴了那么多代码（我也很无奈啊，不贴代码我说不明白啊），说明了Mapper的构建过程。下面来看看这个 Mapper 是如何被使用的。
Tomcat 接收到请求后经过一系列处理(这个过程后面会有笔记细说)到达 org.apache.catalina.connector.CoyoteAdapter#service. 
```java
//1. 获取路由
boolean postParseSuccess = postParseRequest(req, request, res, response);
if (postParseSuccess) {
    request.setAsyncSupported(connector.getService().getContainer().getPipeline().isAsyncSupported());
    //2. 路由信息随request一路在各个容器中走下去
    connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
}
```
上面是CoyoteAdapter#service中的一段代码，在 postParseRequest 中会去获取路由信息, 并将所有路由信息放到request.getMappingData() 中。
```java
boolean mapRequired = true;
while (mapRequired) {
    connector.getService().getMapper().map(serverName, decodedURI,version, request.getMappingData());
    ...
}
```
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response); 这里会真正开始进入容器的操作，先是进入 StandardEngineValve 的 invoke.
```java
public final void invoke(Request request, Response response) throws IOException, ServletException {
    //会去 mappingData 中拿到host， return mappingData.host;
    Host host = request.getHost();
    // 后续的操作也是类似，去request 的 mappingData 拿 context, wrapper 
    host.getPipeline().getFirst().invoke(request, response);
}
```

上面我们已经看到路由信息是怎么使用的，接下来具体看看这个路由信息是如何从 Mapper 中查询出来的。
```java
private final void internalMap(CharChunk host, CharChunk uri, String version, MappingData mappingData) throws IOException {
    if (mappingData.host != null) {
        throw new AssertionError();
    }
    uri.setLimit(-1);
    // 1. 找到 MappedHost, 没有就用默认的
    MappedHost[] hosts = this.hosts;
    MappedHost mappedHost = exactFindIgnoreCase(hosts, host);
    if (mappedHost == null) {
        if (defaultHostName == null) {
            return;
        }
        mappedHost = exactFind(hosts, defaultHostName);
        if (mappedHost == null) {
            return;
        }
    }
    // ### mappdingData host
    mappingData.host = mappedHost.object;
    
    // 2. 根据 uri 找到 context 列表
    ContextList contextList = mappedHost.contextList;
    MappedContext[] contexts = contextList.contexts;
    int pos = find(contexts, uri);
    if (pos == -1) {
        return;
    }

    // 3. 下面这一段用于从context列表中寻找最匹配的Context 
    int lastSlash = -1;
    int uriEnd = uri.getEnd();
    int length = -1;
    boolean found = false;
    MappedContext context = null;
    while (pos >= 0) {
        context = contexts[pos];
        if (uri.startsWith(context.name)) {
            length = context.name.length();
            if (uri.getLength() == length) {
                found = true;
                break;
            } else if (uri.startsWithIgnoreCase("/", length)) {
                found = true;
                break;
            }
        }
        if (lastSlash == -1) {
            lastSlash = nthSlash(uri, contextList.nesting + 1);
        } else {
            lastSlash = lastSlash(uri);
        }
        uri.setEnd(lastSlash);
        pos = find(contexts, uri);
    }
    uri.setEnd(uriEnd);
    if (!found) {
        if (contexts[0].name.equals("")) {
            context = contexts[0];
        } else {
            context = null;
        }
    }
    if (context == null) {
        return;
    }
    //### mappdingData contextPath
    mappingData.contextPath.setString(context.name);


    //4. 没有 version 就那最后一个，有version就根据version从数组中拿
    ContextVersion contextVersion = null;
    ContextVersion[] contextVersions = context.versions;
    final int versionCount = contextVersions.length;
    if (versionCount > 1) {
        Context[] contextObjects = new Context[contextVersions.length];
        for (int i = 0; i < contextObjects.length; i++) {
            contextObjects[i] = contextVersions[i].object;
        }
        mappingData.contexts = contextObjects;
        if (version != null) {
            contextVersion = exactFind(contextVersions, version);
        }
    }
    if (contextVersion == null) {
        contextVersion = contextVersions[versionCount - 1];
    }
    //### mappdingData context contextSlashCount
    mappingData.context = contextVersion.object;
    mappingData.contextSlashCount = contextVersion.slashCount;
    

    if (!contextVersion.isPaused()) {
        // 寻找 wrapper，代码比较多，不贴了
        internalMapWrapper(contextVersion, uri, mappingData);
    }
}
```


### **最后**
纯粹流水账，抽空仔细想想，要是能用几张图把整个过程以及数据结构描述清楚就最好了。


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