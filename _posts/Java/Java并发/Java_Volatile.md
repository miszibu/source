---
title: Java_Volatile
date: 2018-04-11 09:53:09
tags: [Java]
---

​	Volatile是轻量级的synchronized，如果一个变量是用volatile，则它的成本低于synchronized，由于它不会引起线程上下文切换和调度。

当一个变量被定义为volatile之后，它将具备两种性质

1. **保证该变量对所有线程的可见值**。倘若某个线程对volatile修饰的共享变量进行更新，那么其他线程立马可以看到这个更新。因此所有线程取该变量值时，都是最新的值。
2. **禁止指令重排序**。

<!--more-->

## 内存模型概念

​	计算机在运行程序时，在CPU中执行指令，在主存中进行读写，主存读写的速度慢于指令执行的速度，倘若任何交互都需要与主存打交道则会影响效率，因此有了cache，然而CPU缓存为某个CPU独有，只与在该CPU运行的线程有关。

​	虽然Cache解决了效率问题，但带来了数据一致性的问题，在多线程环境下会出现异常，这时我们需要引入方案解决缓存一致性问题。共有两种

* 通过在总线加LOCK锁的方式

* 通过缓存一致性协议

  但是总线是CPU共享的，若被独占，其他CPU会被阻塞，违反了我们想要提高效率的初衷。

​        缓存一致性协议（MESI协议）它确保每个缓存中使用的共享变量的副本是一致的。其核心思想如下：当某个CPU在写数据时，如果发现操作的变量是共享变量，则会通知其他CPU告知该变量的缓存行是无效的，因此其他CPU在读取该变量时，发现其无效会重新从主存中加载数据。

## Java内存模型

​	上面是操作系统如何保证数据一致性，下面我们来看一下Java内存模型，稍微研究一下Java内存模型为我们提供了哪些保证以及在Java中提供了哪些方法和机制来让我们在进行多线程编程时能够保证程序执行的正确性。

在并发编程中我们一般都会遇到这三个基本概念：原子性、可见性、有序性。我们稍微看下volatile

### 原子性

> 原子性：即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

这点类似于数据库的事务

~~~java
i = 0;             ---1
j = i ;            ---2
i++;               ---3
i = j + 1;         ---4
~~~

上面四个操作，其实只有1才是原子操作，其余均不是。

1—在Java中，对基本数据类型的变量和赋值操作都是原子性操作；
2—包含了两个操作：读取i，将i值赋值给j
3—包含了三个操作：读取i值、i + 1 、将+1结果赋值给i；
4—同三

在单线程环境下我们可以认为整个步骤都是原子性操作，但是在多线程环境下则不同，Java只保证了基本数据类型的变量和赋值操作才是原子性的（注：在32位的JDK环境下，对64位数据的读取不是原子性操作，如long、double）。要想在多线程环境下保证原子性，则可以通过锁、synchronized来确保。

> volatile是无法保证复合操作的原子性

### 可见性

> 可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

在上面已经分析了，在多线程环境下，一个线程对共享变量的操作对其他线程是不可见的。

Java提供了volatile来保证可见性。

当一个变量被volatile修饰后，表示着线程本地内存无效，当一个线程修改共享变量后他会立即被更新到主内存中，当其他线程读取共享变量时，它会直接从主内存中读取。
当然，synchronize和锁都可以保证可见性。

### 有序性

> 有序性：即程序执行的顺序按照代码的先后顺序执行。

在Java内存模型中，为了效率是允许编译器和处理器对指令进行重排序，当然重排序它不会影响单线程的运行结果，但是对多线程会有影响。

Java提供volatile来保证一定的有序性。最著名的例子就是单例模式里面的DCL（双重检查锁）。

## 剖析volatile原理

> volatile可以保证线程可见性且提供了一定的有序性，但是无法保证原子性。在JVM底层volatile是采用“内存屏障”来实现的。

由于volatile变量只能保证可见性，在不符合以下两条规则的运算场景中，我们仍然要通过加锁（synchronized或者JUC包下的院子类）来保证原子性。

1. **运算结果并不依赖变量的当前值（反例：a++），或者能够确保只有单一的线程修改变量的值。**
2. **变量不需要与其他的状态变量共同参与不变约束**

**常用volatile关键字的两个场景：**

1.状态标记量

在高并发的场景中，通过一个boolean类型的变量isopen，控制代码是否走促销逻辑，该如何实现?

~~~java
public class ServerHandler {
　　private volatile isopen;
　　public void run() {
		if (isopen) {
          //促销逻辑
　　	   } else {
　　		//正常逻辑
	 　　}
　　}
 　public void setIsopen(boolean isopen) {
 　   this.isopen = isopen
  }
}
~~~

​	用户的请求线程执行run方法，如果需要开启促销活动，可以通过后台设置，具体实现可以发送一个请求，调用setIsopen方法并设置isopen为true，由于isopen是volatile修饰的，所以一经修改，其他线程都可以拿到isopen的最新值，用户请求就可以执行促销逻辑了。

2.double check

​	单例模式的一种实现方式，但很多人会忽略volatile关键字，因为没有该关键字，程序也可以很好的运行，只不过代码的稳定性总不是100%，说不定在未来的某个时刻，隐藏的bug就出来了。

~~~java
class Singleton {
　　private volatile static Singleton instance;
　　public static Singleton getInstance() {
　	　if (instance == null) {
　	　  syschronized(Singleton.class) {
　　		 if (instance == null) {
　　         instance = new Singleton();
　　       }
　　     }
　　   }
　   　return instance;
　　}
}
~~~

　不过在众多单例模式的实现中，我比较推荐懒加载的优雅写法Initialization on Demand Holder(IODH)。

~~~java
public class Singleton {
　　static class SingletonHolder {
　　  static Singleton instance = new Singleton();
　　}
　　public static Singleton getInstance(){
　　  return SingletonHolder.instance;
　　}
}
~~~

**如何保证内存可见性?**

​	在JVM的内存模型中，有主内存和工作内存的概念，每个线程对应一个工作内存，并共享主内存的数据，下面看看操作普通变量和volatile变量有什么不同：

​	1、对于**普通变量**：读操作会优先读取工作内存的数据，如果工作内存中不存在，则从主内存中拷贝一份数据到工作内存中;写操作只会修改工作内存的副本数据，这种情况下，其它线程就无法读取变量的最新值。

　　2、对于**volatile变量**，读操作时JMM会把工作内存中对应的值设为无效，要求线程从主内存中读取数据;写操作时JMM会把工作内存中对应的数据刷新到主内存中，这种情况下，其它线程就可以读取变量的最新值。

　　volatile变量的内存可见性是基于**内存屏障(Memory Barrier)**实现的，什么是内存屏障?内存屏障，又称内存栅栏，是一个CPU指令。在程序运行时，为了提高执行性能，编译器和处理器会对指令进行重排序，JMM为了保证在不同的编译器和CPU上有相同的结果，通过插入特定类型的内存屏障来禁止特定类型的编译器重排序和处理器重排序，插入一条内存屏障会告诉编译器和CPU：不管什么指令都不能和这条Memory Barrier指令重排序。

​	**Java代码**：

```
　instance = new Singleton();
```

　　**汇编代码**：

```
　0x01a3de1d: movb $0x0,0x1104800(%esi);

　0x01a3de24: **lock** addl $0x0,(%esp);
```

　　这个lock前缀指令相当于上述的内存屏障，提供了以下保证：

　　1、将当前CPU缓存行的数据写回到主内存;

　　2、这个写回内存的操作会导致在其它CPU里缓存了该内存地址的数据无效。

　　CPU为了提高处理性能，并不直接和内存进行通信，而是将内存的数据读取到内部缓存(L1，L2)再进行操作，但操作完并不能确定何时写回到内存，如果对volatile变量进行写操作，当CPU执行到Lock前缀指令时，会将这个变量所在缓存行的数据写回到内存，不过还是存在一个问题，就算内存的数据是最新的，其它CPU缓存的还是旧值，所以为了保证各个CPU的缓存一致性，每个CPU通过嗅探在总线上传播的数据来检查自己缓存的数据有效性，当发现自己缓存行对应的内存地址的数据被修改，就会将该缓存行设置成无效状态，当CPU读取该变量时，发现所在的缓存行被设置为无效，就会重新从内存中读取数据到缓存中。