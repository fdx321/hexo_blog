title: 【Motan源码学习】3-扩展机制
date: 2017-07-22 22:18:37
tags:
- Java
- Motan
---
RPC 框架中有很多地方需要可扩展，比如，序列化（Thrift, Protocol Buffer, Hessian等），LoadBalance策略（随机、权重、Round Robin等），这些东西都要方便扩展且可以在配置文件中指定使用哪一种。
Motan 使用了一种 SPI （Service Provider Interface）的机制，实现上和阿里的 [cooma](https://github.com/alibaba/cooma) 类似。本质上都是定义一个接口，该接口可以有很多实现，代码中面向接口编程，但具体使用哪个实现可以在配置文件中指定，然后用反射的机制去加载和使用它。只不过看实现的是否优雅。


![400](/images/【Motan源码学习】3-扩展机制_1.png)
motan-core 模块中的 META-INF/services 定义了一系列扩展点。文件名为接口名，文件内容为接口的实现类名, 可以配置多个实现类，每个类名占一行。
<!--more-->
所有支持扩展的接口上需加上 @Spi 的注解, 实现类上需加上 @SpiMeta 的注解，指定name. 用户配置的时候就可以用该name来配置使用哪种实现。
这里和cooma的实现有点不一样，comma是直接在配置文件里指定name的。
```
name1 = com.xxx.xxx.impl_1
name2 = com.xxx.xxx.impl_2
```

扩展点的使用方法举例，
```java
ConfigHandler configHandler = ExtensionLoader.getExtensionLoader(ConfigHandler.class).getExtension(MotanConstants.DEFAULT_VALUE);
```
getExtensionLoader(ConfigHandler.class)的时候会返回一个单例的ExtensionLoader，如果是首次getExtension，会去加载配置文件里的所有实现类（加载后就放到Map里，下次就不加载了），然后实例化, 如果是单例就将其缓存在Map里，方便下次返回。

具体的扩展机制实现在 com.weibo.api.motan.core.extension package里，代码比较简单易懂。




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
