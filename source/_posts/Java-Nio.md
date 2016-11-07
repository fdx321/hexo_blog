title: Java Nio
date: 2016-11-07 10:26:30
tags:
- Java
- Nio
---
最近在看Netty的源码，顺便了解了一下Java的Nio, 结合JDK源码过了一遍[Java NIO Tutorial](http://tutorials.jenkov.com/java-nio/index.html)。下面对其中的一些知识点做下总结。

Java Nio 有三个关键的组件，Channel, Buffer, Selector.

### Channel and Buffer
主要包括以下几种Channel和Buffer
* FileChannel (文件)
* DatagramChannel (UDP)
* SocketChannel (TCP)
* ServerSocketChannel (TCP Server)
---
* ByteBuffer
* CharBuffer
* DoubleBuffer
* ...

Channel 和 Buffer 之间的数据传输<!--more-->
```
┌─────────┐                  ┌─────────┐
│ Channel │--- read to --->  │ Buffer  │
└─────────┘                  └─────────┘
┌─────────┐                  ┌─────────┐
│ Channel │<-- write from -- │ Buffer  │
└─────────┘                  └─────────┘
```
Channel 之间也可以直接进行数据交互，不通过Buffer. 通过channel的transformTo 或 transformFrom方法来实现。

Buffer 相关的几个指针
* capacity
* position
* limit

Buffer提供的这些方法，本质上就是调整position或者limit的位置：
* flit()
* reset()
* rewind()
* clear()
* ....

Scatter/Gather : 将channel中的数据按顺序读进多个buffer/将多个buffer的数据按顺序写入channel

### Selector
通常的用法
```java
//创建selector
Selector selector = Selector.open();

// 配置 channel 为非阻塞，只有在非阻塞的情况下才可以用selector.
// 所以, FileChannel就不能用selector
// A FileChannel cannot be set into non-blocking mode. It always runs in blocking mode
channel.configureBlocking(false);

// 将channel注册到selector中
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```
在register的过程中，相应的selector、channel会被attach到这个**selectionKey**上，因此可以通过selectionKey的channel()、selector()方法获得绑定在这个key上的channel和selector.
**SelectionKey**用于表示 selector 用于监听什么事件，可以监听4种事件：
* SelectionKey.OP_READ
* SelectionKey.OP_WRITE
* SelectionKey.OP_CONNECT
* SelectionKey.OP_ACCPET

如果想监听多种事件，可以通过按位或来实现

### 样例代码分析
作者jjenkov最后写了一个nio demo https://github.com/fdx321/java-nio-server. 过了一遍源码，工作过程大致如图：
![](/images/Java-Nio_1.png)

红色圈圈表示一个死循环，整个Server主要有两个线程，
* SocketAccepter 负责循环接受连接请求，连接建立后创建新的Socket放入队列
* SocketProcessor 负责循环处理各个client传过来的数据

这只是一个demo，性能并不是很好，从线程模型上就可以看出，只有一个线程来处理各个client的数据，而且中间还有不必要的数据拷贝。

后面我会记录一下Netty的源码分析，到时对这个线程模型应该会有一个明显的对比。

### Reference
* [Java NIO Tutorial](http://tutorials.jenkov.com/java-nio/index.html)
