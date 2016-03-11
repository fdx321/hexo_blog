title: 终于搞清楚Java的日志了
date: 2015-12-20 18:11:30
tags:
- java
- log
categories:
- 博客
---
Java的日志框架很多，JUL, Log4J, Lobback, JCL, SLF4J等，之前也都大概知道它们是干什么的，网上也有各种讲怎么去配置的。但总觉得不得要领，这周结合官网文档和网上一些文章，把JUL, Log4J, JCL, SLF4J的源码过了一遍，爽。<!--more-->
#### JUL
![](/images/JUL.png)
JUL相对来说比较简单，就这么几个类。Level定义了日志级别，这里的日志级别和其他框架的命名不太一样。
* SEVERE (highest value)
* WARNING
* INFO
* CONFIG
* FINE
* FINER
* FINEST  (lowest value)

In addition there is a level **OFF** that can be used to turn off logging, and a level **ALL** that can be used to enable logging of all messages.
日志信息是保存在LogRecord中的，Logger首先判断日志级别，决定是否打印，然后调用Filter看是否过滤，最后调用Handler的publish方法来打印日志。Handler有多种，包括MemeoryHandler, ConsoleHandler，FileHandler等。在publish中，会获取Formatter来格式化日志，然后打印。中间稍微复杂的地方是有个日志层级的概念。这篇文章分析得很清楚，[Java Logging: Logger Hierarchy](http://tutorials.jenkov.com/java-logging/logger-hierarchy.html),整个系列教程也很不错。

#### Log4j
2.X比1.X的代码无论是排版上还是注释上都好很多。
![](http://logging.apache.org/log4j/2.x/images/Log4jClasses.jpg)
顺便了解了一下NDC和MDC
* [NDC vs MDC - Which one should I use?](https://wiki.apache.org/logging-log4j/NDCvsMDC)
* [在 Web 应用中增加用户跟踪功能](http://www.ibm.com/developerworks/cn/web/wa-lo-usertrack/)

#### JCL
![](/images/JCL.png)
commons-logging只是做了一层包装，提供了一个门面接口，下层可以通过配置自由的选择各种日志框架，加载顺序如下

* Use a factory configuration attribute named <code>org.apache.commons.logging.Log</code> to identify the requested implementation class.
* Use the <code>org.apache.commons.logging.Log</code> system property to identify the requested implementation class.
* If <em>Log4J</em> is available, return an instance of <code>org.apache.commons.logging.impl.Log4JLogger</code>.
* If <em>JDK 1.4 or later</em> is available, return an instance of <code>org.apache.commons.logging.impl.Jdk14Logger</code>.
* Otherwise, return an instance of <code>org.apache.commons.logging.impl.SimpleLog</code>.


#### SLF4J
![](/images/slf4j.png)


#### 其他
关于这么多框架如何搭配使用，这篇文章总结的不错，[slf4j、jcl、jul、log4j1、log4j2、logback大总结](http://my.oschina.net/pingpangkuangmo/blog/410224?fromerr=OUHSuJjF)

[为什么要使用SLF4J而不是Log4J](http://www.importnew.com/7450.html)中重点介绍了SLF4J中的占位符，以及它是如何提高效率的。
这是在Log4j中使用的方案
```java
if (logger.isDebugEnabled()) {
    logger.debug("Processing trade with id: " + id + " symbol: " + symbol);
}
```
SLF4J的方案：
```java
logger.debug("Processing trade with id: {} and symbol : {} ", id, symbol);
```
明显后一种更优雅，且效率更高。
#### Reference
* [Java Logging: Logger Hierarchy](http://tutorials.jenkov.com/java-logging/logger-hierarchy.html)
* [Java Util Logging Hierarchy reason?](http://stackoverflow.com/questions/13397899/java-util-logging-hierarchy-reason)
* [NDC vs MDC - Which one should I use?](https://wiki.apache.org/logging-log4j/NDCvsMDC)
* [在 Web 应用中增加用户跟踪功能](http://www.ibm.com/developerworks/cn/web/wa-lo-usertrack/)
* [slf4j、jcl、jul、log4j1、log4j2、logback大总结](http://my.oschina.net/pingpangkuangmo/blog/410224?fromerr=OUHSuJjF)
* [为什么要使用SLF4J而不是Log4J](http://www.importnew.com/7450.html)