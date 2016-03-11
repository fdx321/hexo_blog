title: GroovyShell永久区内存泄露
date: 2015-12-24 12:55:43
tags:
- Java
- Groovy

categories:
- 博客
---
今天想在项目中用一下Groovy，主要是为了动态的执行用户配置的命令。代码大致如下:
```java
Binding binding = new Binding();
binding.setVariable("R",new RuleFunc());
GroovyShell shell = new GroovyShell(,binding);
shell.evaluate("Groovy Command");
```
<!--more-->把上面的代码放在一个循环里多跑几次，过一会就Out of Memory了。
![](/images/GroovyGC.png)
对代码稍微作下修改
```java
Binding binding = new Binding();
binding.setVariable("R",new RuleFunc());
GroovyShell shell = new GroovyShell(new GroovyClassLoader(),binding);
shell.evaluate("Groovy Command");
```
![](/images/GroovyNoGC.png)
修改后的代码因为每次都是使用不同的ClassLoader，之前不用的loader会被GC，它加载的类也会被GC，虽然修改后OOM的问题解决了，但是因为GroovyShell每次都会去编译加载类，效率很低。


##### Reference
* [在Java里整合Groovy脚本的一个陷阱](http://rednaxelafx.iteye.com/blog/620155)

