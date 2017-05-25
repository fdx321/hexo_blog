title: 【Tomcat学习笔记】1-源码Debug
date: 2017-05-15 10:55:01
tags:
- Java
- Tomcat
---
最近一段时间看了一遍 Tomcat 8.0 的源代码，略有所得，打算整理一个系列的学习笔记。
Tomcat 源码的工程不是用 Maven 或 Gradle 管理的，使用起来不方便，Github 上有人整理一个 Maven 的工程。[我 Fork 了一份](https://github.com/fdx321/tomcat8.0-source-research)。

### 工程的整体结构:
```
tomcat8.0-source-research
    catalina-home
    jsp-demo
    PDF-resource
    springmvc-demo
    tomcat-8.0-doc
    tomcat-code
    .gitignore
    NOTES.md
    pom.xml
    README.md
    tomcat-study.iml
```
* 只有 tomcat-code（源码） 和 catalina-home （配置、lib等）是运行工程必须的 <!--more-->
* jsp-demo 和 springmvc-demo 是我为了方便学习 JSP 和 Servlet 机制写的两个简单的工程，可以用 ```mvn package``` 打成 war 包后放到 ./catalina-home/webapps 目录下使用。
* PDF-resource（tomcat 的书籍），tomcat-8.0-doc (文档)

### Debug
* 将项目导入 IDEA（```open pom.xml -a /Applications/IntelliJ\ IDEA.app/```）
* 配置 Debug Configuration ![](/images/【Tomcat学习笔记】源码Debug_1.png)

入口类是 org.apache.catalina.startup.Bootstrap, 启动参数为

-Dcatalina.home=catalina-home -Dcatalina.base=catalina-home -Djava.endorsed.dirs=catalina-home/endorsed -Djava.io.tmpdir=catalina-home/temp -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djava.util.logging.config.file=catalina-home/conf/logging.properties -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9876 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=127.0.0.1

这些参数可以自己调整，含有jmxremote的几个参数是为了开启 JMX 功能


然后就可以从 Bootstrap 类的 main 方法开始一步步Debug了。

##### .
** 以上皆是阅读源码 https://github.com/fdx321/tomcat8.0-source-research 所得 **
