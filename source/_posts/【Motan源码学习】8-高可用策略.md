title: 【Motan源码学习】8-高可用策略
date: 2017-07-27 22:24:26
tags:
- Java
- Motan
---
Motan 实现了 FailFast、FailOver、FailBack 这几种HA策略。
**1.FailFast**
*Fail-fast is a property of a system or module with respect to its response to failures. A fail-fast system is designed to immediately report at its interface anyfailure or condition that is likely to lead to failure. Fail-fast systems are usually designed to stop normal operation rather than attempt to continue a possibly flawed process. Such designs often check the system's state at several points in an operation, so any failures can be detected early. A fail-fast module passes the responsibility for handling errors, but not detecting them, to the next-higher system design level.*

这是维基百科关于FailFast的描述，就是错误要尽早发现，失败立即报错结束。 Motan 的 FailFast 实现的比较简单，和简单的调用一样。这让我想起了Hystrix这个熔断框架，里面也有FailFast的机制，可以去了解一下。<!--more-->
**2.FailOver**
*In computing, failover is switching to a redundant or standby computer server, system, hardware component or network upon the failure or abnormal termination of the previously active application,[1] server, system, hardware component, or network. Failover and switchover are essentially the same operation, except that failover is automatic and usually operates without warning, while switchover requires human intervention.*

这是维基百科关于FailOver的描述，即失效转移，一个节点失败了，立刻尝试别的节点。具体到 Motan 的服务调用，是这样实现的, 即一个Refer失败了，就尝试别的Refer.
```java
public Response call(Request request, LoadBalance<T> loadBalance) {
    List<Referer<T>> referers = selectReferers(request, loadBalance);
    URL refUrl = referers.get(0).getUrl();
    // 先使用method的配置
    int tryCount = refUrl.getMethodParameter(request.getMethodName(), request.getParamtersDesc(), URLParamType.retries.getName(),URLParamType.retries.getIntValue());
    for (int i = 0; i <= tryCount; i++) {
        Referer<T> refer = referers.get(i % referers.size());
        try {
            request.setRetries(i);
            return refer.call(request);
        } catch (RuntimeException e) {
            ....
        }
    }
    throw new MotanFrameworkException("FailoverHaStrategy.call should not come here!");
}
```
**3.FailBack**
*Failback is the process of restoring a system, component, or service previously in a state of failure back to its original, working state, and having the standby system go from functioning back to standby.*
这是维基百科关于FailBack的描述, 即失败自动恢复，在实现上一般是后台记录失败请求，定时重发。Motan 在服务调用的时候并没有做FailBack. 而是在服务注册的时候使用了FailBack的机制。
```java
public FailbackRegistry(URL url) {
    super(url);
    long retryPeriod = url.getIntParameter(URLParamType.registryRetryPeriod.getName(), URLParamType.registryRetryPeriod.getIntValue());
    retryExecutor.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            try {
                retry();
            } catch (Exception e) {
                LoggerUtil.warn(String.format("[%s] False when retry in failback registry", registryClassName), e);
            }
        }
    }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
}
```
FailbackRegistry 中会专门有个现成定时的去重试，注册失败的服务会被记录到failedRegistered这个Set中，重试的时候就去重新注册这个Set里的服务，成功了就移除，失败了下次继续。


Motan 还实现了一种异常隔离机制，调用Server连续失败超过指定次数后置为不可用，然后定期进行心跳探测，OK后再将Server置为可用。这个功能是在 transport 层实现的，在讲心跳机制的时候有说过。


##### .
** 以上皆是阅读源码 https://github.com/weibocom/motan （tag 0.3.1）所得，文中贴的代码或配置有所删减 **
