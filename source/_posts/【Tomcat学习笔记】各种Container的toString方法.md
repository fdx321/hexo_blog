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
看了toString方法后，原来如此，这里为了性能，还用了StringBuilder，而不是 String + String

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
