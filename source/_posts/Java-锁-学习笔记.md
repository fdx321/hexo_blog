title: Java 锁 学习笔记
date: 2017-04-19 13:40:48
tags:
- Java
---
*纯粹个人笔记，为什么要写博客？强化自己学习的知识点。实践过的东西才最深刻，但很多知识平时项目中不一定有机会实践，所以唯有自己能够写下来、描述清楚才是真的理解了。*

最近要做个Java锁的分享，看了些资料，做了个slide. 这里再对这些东西做个总结。

锁是一种多线程间的同步工具，最简单的例子，两个线程同时执行1000次count++操作(count是int形的)，最后的结果并不一定是2000. 因为count++并不是一个原子操作。

JDK定义了一个标准的Lock接口，如下：
```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    //如果lock成功返回true, 否则返回false，不会阻塞线程
    boolean tryLock();
    //如果在一段时间内lock成功返回true, 返回false
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```

Java里有两种类型的锁，一种是synchronized，一种是实现了Lock接口的锁。原理上，前者是在JVM层面做的，后者更多的是在Java层面做的，所以后者也更容易在应用层做扩展，更容易通过JDK源码去理解它的机制。<!--more-->

## synchronized
synchronized的用法很多，可以用于同步方法(锁的是当前实例对象，即this)、同步静态方法(锁的是当前的class)、同步代码块(锁的是括号里指定的对象).

```java
public synchronized void func() {}

public static synchronized void func() {}

public void func(){
  synchronized(this) {}
}

public void func(){
  ClassA a = new ClassA();
  synchronized(a) {}
}

class AAA {
  public static void func(){
    synchronized(AAA.class) {}
  }
}
```

[深入JVM锁机制1-synchronized](http://blog.csdn.net/chen77716/article/details/6618779) 和 [聊聊并发（二）Java SE1.6中的Synchronized](http://ifeve.com/java-synchronized/) 对 synchronized 的原理进行了详细的分析。

对于同步代码块的，可以通过javap反汇编来看看实际做了哪些操作
```java
class AAA {
  public void func(){
    synchronized(this) {
      System.out.println("aaa");
    }
  }
}
```
执行**javac AAA.java** 和 **javap -c AAA** 命令后得到：
```java
  0: aload_0
  1: dup
  2: astore_1
  3: monitorenter
  4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
  7: ldc           #3                  // String aaa
  9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
  12: aload_1
  13: monitorexit
  14: goto          22
  17: astore_2
  18: aload_1
  19: monitorexit
  20: aload_2
  21: athrow
  22: return
```
可以看到在进入同步块之前执行了monitorenter，结束后执行了monitorexit. 在JVM实现中，称它为对象监视器。

对象监视器会设置几种状态来区分等待锁的线程，如图所示：
![](/images/Java-锁-学习笔记-1.gif)
[图片来源](http://blog.csdn.net/chen77716/article/details/6618779)
请求锁的线程首先被放置到Contention List，有资格成为候选的再被移动到Entry List，Owner就是最后获得锁的线程。在Contention List、Entry List中的线程都处于等待状态。让线程等待通常有两种方法，一种是通过系统调用让线程阻塞，这个操作涉及用户态和内核态的来回切换，比较消耗资源。一种就是采用**自旋锁 (Spin Lock)**。

## 自旋锁 (Spin Lock)
下面是一种很简单的自旋锁实现，这种实现方式相对于其他阻塞锁的实现来说有它的优点，它只是简单的循环，避免了用户态和内核态的切换。但是当并发量大的时候，所有等待的线程都要空跑(循环)，也很浪费资源。
```
public class SpinLock {
  private AtomicReference<Thread> owner =new AtomicReference<>();
  public void lock(){
    Thread current = Thread.currentThread();
    //compareAndSet (CAS) 是一个原子操作
    while(!owner.compareAndSet(null, current)){
    }
  }
  public void unlock (){
    Thread current = Thread.currentThread();
    owner.compareAndSet(current, null);
  }
}
```
关于自旋锁，还有很多深入的东西要考虑，对于自旋锁**旋转周期**的选择，HotSpot认为最佳时间应是一个线程上下文切换的时间，但是不同硬件的这个时间是不一样的，所以很多只是一个近似的实现。
为了解决可重入性、公平性等问题，又有了Ticket Lock, MCS Lock, CLH Lock等算法，具体可以参考[5][6][7]。


## ReentrantLock
ReentrantLock 是基于 AbstractQueuedSynchronizer （简称AQS）实现的，AQS是JDK中一个很重要的同步器， concurrent package 中很多类都使用了它。ReentrantLock是可重入的，支持公平性的。对于读多写少的场景，由于读读没必要互斥，为了提高性能，又有了ReentrantReadWriteLock.
关于ReentrantLock、AbstractQueuedSynchronizer的原理, 文末的参考文献[2][8]中有几篇分析得很清楚。

哎，曾经看过ReentrantLock和AQS的源码，然。。。。现在又忘了，没啥印象了。。。



### 参考文献
[1] [深入JVM锁机制1-synchronized](http://blog.csdn.net/chen77716/article/details/6618779)
[2] [深入JVM锁机制2-Lock](http://blog.csdn.net/chen77716/article/details/6641477)
[3] [聊聊并发（二）Java SE1.6中的Synchronized](http://ifeve.com/java-synchronized/)
[4] JDK 1.8 源码和注释
[5] [自旋锁、排队自旋锁、MCS锁、CLH锁](http://coderbee.net/index.php/concurrent/20131115/577)
[6] [java锁的种类以及辨析（一）：自旋锁](http://ifeve.com/java_lock_see1/)
[7] [Java锁的种类以及辨析（二）：自旋锁的其他种类](http://ifeve.com/java_lock_see2/)
[8] [AbstractQueuedSynchronizer的介绍和原理分析](http://ifeve.com/introduce-abstractqueuedsynchronizer/)
