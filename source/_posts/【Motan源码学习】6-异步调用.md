title: 【Motan源码学习】6-异步调用
date: 2017-07-25 22:16:37
tags:
- Java
- Motan
---
Motan支持异步调用，在需要支持异步调用的接口上加 @MotanAsync 注解，在编译时就会自动生成一个Async结尾的异步接口。例如：
```java
@MotanAsync
public interface MotanDemoService {
	String hello(String name);
}
```
编译后会生成
```java
public interface MotanDemoServiceAsync extends MotanDemoService {
  ResponseFuture helloAsync(String name);
}
```
服务调用者就可以用这个接口来配置引用，用带有Async后缀的方法返回的ResponseFuture去异步的获得结果了。

这里用到了编译时处理注解的技术，Motan 中实现了 com.weibo.api.motan.transport.async.MotanAsyncProcessor 用来处理 @MotanAsync 注解，自动生成代码。并在 motan-core META-INF/services 的 javax.annotation.processing.Processor 文件中添加该Processor，这样在编译的时候就会去处理需要异步的接口了。
<!--more-->

在动态代理层，会根据接口后缀、方法后缀来判断是同步调用还是异步调用, 并发async这个boolean放在RpcContext这个ThreadLocal中。
```java
//com.weibo.api.motan.proxy.RefererInvocationHandler#init
private void init() {
    ...
    interfaceName = MotanFrameworkUtil.removeAsyncSuffix(clz.getName());
}
//com.weibo.api.motan.proxy.RefererInvocationHandler#invoke
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    ...
    String methodName = method.getName();
    boolean async = false;
    if (methodName.endsWith(MotanConstants.ASYNC_SUFFIX) && method.getReturnType().equals(ResponseFuture.class)) {
        methodName = MotanFrameworkUtil.removeAsyncSuffix(methodName);
        async = true;
    }
    RpcContext.getContext().putAttribute(MotanConstants.ASYNC_SUFFIX, async);
    ...
}
```
在NettyClient发起调用的时候，就会根据ThreadLocal的async，来判断是返回一个Future Response，还是拿到结果后在返回Response.
```java
private Response request(Request request, boolean async) throws TransportException {
	Channel channel = null;
	Response response = null;
	try {
		channel = borrowObject();
		...
		// 这里chanel的request本来就是异步的
		response = channel.request(request);
		// return channel to pool
		returnObject(channel);
	} catch (Exception e) {
		...
	}
	//如果async为true，就直接返回之前的response，否则用这个response get到value后再返回
	response = asyncResponse(response, async);
	return response;
}
```

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