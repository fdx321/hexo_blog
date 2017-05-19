title: 【Tomcat学习笔记】启动流程分析
date: 2017-05-19 13:35:11
tags:
- Java
- Tomcat
---
*讲真，假如你不幸看到这里了，别往下看了，写得一坨 Shit. 只有我自己能看懂的笔记。讲道理，这种流程，真是一图剩千言，然。。。画图好累的。*
### 1. 执行 Bootstrap 类的 static 代码块
主要功能是确定 catalinaHome 和 catalinaBase 两个目录，如果有配置 "catalina.home"/“catalina.base” 这两个 property，就以配置的为准，没有配置就走 fall back，用默认的目录。那么 home 和 base 分别只哪两个目录呢，是用来干嘛的呢。先来看下官方的解释：
* **catalina.home** : the tomcat product installation path
* **catalina.base** : the tomcat instance installation path
看了是不是还是一脸闷逼？我也是。OK，看下我们的工程，里面有个目录
![](/images/【Tomcat学习笔记】启动流程分析_1.png)<!--more-->
一般情况下，catalina.home 和 catalina.base 指向同一个这样的目录. 但有时候我们需要在一台机器上运行多个 Tomcat 实例，没必要安装多个 Tomcat. 只需要配置多个工作目录，共享一个安装目录就可以了。Tomcat 运行需要 conf、lib、logs、。。等这些目录，对每个实例，可以单独弄个目录，在里面创建好 conf、lib、logs 这些目录，然后配置好各自的 catalina.base 就可以了。
*完了，扯着扯着就跑题了，感觉写不完了，（这里我是想加个表情的，于是发现 hexo 是可以换个支持表情的 markdown 渲染器的，然而我懒得玩了）*

### 2. 未完待续。。
