title: 【Tomcat学习笔记】各种Container的toString方法
date: 2017-05-18 21:08:24
tags:
- Java
- Tomcat
---
在 Debug 源码的时候发现日志打得特别清晰，容器之间的关系特别清晰，都是这样的形式
```
StandardEngine[xxx].StandardHost[xxx].StandardContext[xxx].StandardWrapper[xxx]
```
看了toString方法后，原来如此。

*靠，这 TM 也要写篇博客？这么水？哦，我只是觉得我自己专门去看过这块代码，觉得很有印象。*

#### StandardEngine#toString
```java
public String toString() {
  StringBuilder sb = new StringBuilder("StandardEngine[");
  sb.append(getName());
  sb.append("]");
  return (sb.toString());
}
```
<!--more-->
#### StandardHost#toString
```java
public String toString() {
  StringBuilder sb = new StringBuilder();
  if (getParent() != null) {
      sb.append(getParent().toString());
      sb.append(".");
  }
  sb.append("StandardHost[");
  sb.append(getName());
  sb.append("]");
  return (sb.toString());
}
```
#### StandardContext#toString
```java
StringBuilder sb = new StringBuilder();
  if (getParent() != null) {
      sb.append(getParent().toString());
      sb.append(".");
    }
    sb.append("StandardContext[");
    sb.append(getName());
    sb.append("]");
    return (sb.toString());
}
```

##### .
** 以上皆是阅读源码 https://github.com/fdx321/tomcat8.0-source-research 所得 **
