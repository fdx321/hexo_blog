title: 【Tomcat学习笔记】3-组件声明周期
date: 2017-05-17 08:58:05
tags:
- Java
- Tomcat
---
Tomcat 里很多组件都实现了生命周期的接口，大到 Server、Service、Engine、Host、Context、Wrapper 这样的关键组件，小到 Connector、Valve 这样的小组件。
<img src="/images/【Tomcat学习笔记】组件声明周期_1.svg"/>
<!--more-->

大部分组件都继承了 LifecycleMBeanBase 类，从名字就可以看出，这个类主要实现了两种功能 **生命周期** 和 **JMX**，**JMX**这里先不关注，后面会篇专门的笔记分析这块。**Lifecycle** 这个接口定义了下面这几个方法
* init()
* start()
* stop()
* destroy()
这些方法的执行都会让组件从生命周期的一个阶段进入另一个阶段，**LifecycleState** 这个枚举类定义了生命周期的各个阶段，
* NEW
* INITIALIZING
* INITIALIZED
* STARTING_PREP
* STARTING
* ...
这个状态机是这样子滴：
```pre
           start()
 -----------------------------
 |                           |
 | init()                    |
NEW -»-- INITIALIZING        |
| |           |              |     ------------------«-----------------------
| |           |auto          |     |                                        |
| |          \|/    start() \|/   \|/     auto          auto         stop() |
| |      INITIALIZED --»-- STARTING_PREP --»- STARTING --»- STARTED --»---  |
| |         |                                                  |         |  |
| |         |                                                  |         |  |
| |         |                                                  |         |  |
| |destroy()|                                                  |         |  |
| --»-----«--       auto                    auto               |         |  |
|     |       ---------«----- MUST_STOP ---------------------«--         |  |
|     |       |                                                          |  |
|    \|/      ---------------------------«--------------------------------  ^
|     |       |                                                             |
|     |      \|/            auto                 auto              start()  |
|     |  STOPPING_PREP ------»----- STOPPING ------»----- STOPPED ----»------
|     |                                ^                  |  |  ^
|     |               stop()           |                  |  |  |
|     |       --------------------------                  |  |  |
|     |       |                                  auto     |  |  |
|     |       |                  MUST_DESTROY------«-------  |  |
|     |       |                    |                         |  |
|     |       |                    |auto                     |  |
|     |       |    destroy()      \|/              destroy() |  |
|     |    FAILED ----»------ DESTROYING ---«-----------------  |
|     |                        ^     |                          |
|     |     destroy()          |     |auto                      |
|     --------»-----------------    \|/                         |
|                                 DESTROYED                     |
|                                                               |
|                            stop()                             |
----»-----------------------------»------------------------------
```

**Lifecycle**还定义了这些方法用来管理每个组件的监听器，当组件所在的生命周期发生变更时，就会触发一个事件，然后通知到所有监听器。
* addLifecycleListener()
* findLifecycleListeners()
* removeLifecycleListener()
为此，**Lifecycle** 定义了
* BEFORE_INIT_EVENT
* AFTER_INIT_EVENT
* START_EVENT
* ...
等一系列 EVENT 用来表示是哪个阶段的事件。

下面是 **LifecycleBase** 中的 init 方法的实现
```java
public final synchronized void init() throws LifecycleException {
    if (!state.equals(LifecycleState.NEW)) {
        invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
    }
    setStateInternal(LifecycleState.INITIALIZING, null, false);

    try {
        initInternal();
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        setStateInternal(LifecycleState.FAILED, null, false);
        throw new LifecycleException(sm.getString("lifecycleBase.initFail",toString()), t);
    }

    setStateInternal(LifecycleState.INITIALIZED, null, false);
}

// NOTE:这里比源码有所删减
private synchronized void setStateInternal(LifecycleState state,
    Object data, boolean check) throws LifecycleException {
    this.state = state;
    String lifecycleEvent = state.getLifecycleEvent();
    if (lifecycleEvent != null) {
        fireLifecycleEvent(lifecycleEvent, data);
    }
}
/**
 *  这个方法是在 LifecycleSupport 类里实现的，LifecycleBase 只是 简单的调用了一下，
 *  其他的addLifecycleListener,removeLifecycleListener ... 等管理listener的也都是
 *  通过 LifecycleSupport 来操作的
 */
public void fireLifecycleEvent(String type, Object data) {
    LifecycleEvent event = new LifecycleEvent(lifecycle, type, data);
    LifecycleListener interested[] = listeners;
    for (int i = 0; i < interested.length; i++)
        interested[i].lifecycleEvent(event);
}

```
init、start、stop、destroy 这些方法都会调用各自的 initInternal、starttInternal、... 方法，internal 方法由具体的组件(LifecycleBase的子类)自己实现. 除了调用 internal 方法，这个方法的主要功能就是，设置状态，触发事件。触发事件的时候会遍历数组中的所有Listener，调用它们的lifecycleEvent方法。

##### .
** 以上皆是阅读源码 https://github.com/fdx321/tomcat8.0-source-research 所得 **
