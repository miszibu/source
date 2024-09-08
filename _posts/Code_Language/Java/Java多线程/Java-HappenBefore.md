---
title: Java-HappenBefore
date: 2020-07-16 22:26:18
tags: [Java]
---

JSR 133中对Happen-before的定义如下：Two actions can be ordered by a happens-before relationship.If one action happens before another, then the first is visible to and ordered before the second.

**如果两个操作存在 Happen-before 的一个关系，那么前置的操作一定先于后置操作且对后置操作可见**

JMM 中定义了许多 Action, 有些 Action 之间就存在 Happen-Before 关系。

<!--more-->

## 工作内存 & 主内存

### 主内存 Main Memory

主内存可以简单理解为计算机当中的内存，但又不完全等同。主内存被所有的线程所共享，对于一个共享变量（比如静态变量，或是堆内存中的实例）来说，主内存当中存储了它的“本尊”。

### 工作内存 Working Memory

**线程的工作内存是cache和寄存器的一个统称。**每个线程拥有自己的工作内存，对于一个共享变量来说，工作内存当中存储了它的“副本”。

**由于内存和 Cache 访问速度的差距。**线程对共享变量的所有操作都必须在工作内存进行，不能直接读写主内存中的变量。不同线程之间也无法访问彼此的工作内存，变量值的传递只能通过主内存来进行。

Note: 其实 Happen-Before 原则，跟本小标题关系不大。



## Happen-before 规则

1. **程序执行规则**：一个线程中的每个操作，happen-before与该线程的中的任意后续操作。
2. **锁规则**：对于一个锁的解锁，happen-before与随后对这个锁的加锁。
3. **volatile变量规则**：对于一个volatile修饰的变量的写操作，happen-before于任意后续对这个变量的读操作。
4. **传递性**：如果A happen-before B，且B happen-before C，那么 A happen-before C。
5. **start规则**：如果线程A执行操作ThreadB.start()（启动B线程），那么A线程的ThreadB.start()操作happen-before于线程B中的任意操作。
6. **join操作**：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happen-before与线程A从ThreadB.join()操作成功返回。

## Volatile 中的 Happen-before

> **volatile变量规则**：对于一个volatile修饰的变量的写操作，happen-before于任意后续对这个变量的读操作。

**可以翻译成 一个 Volatile 的写操作执行后，一定是对后续的读操作可见。**也就是可见性



举两个例子来理解：

```java
// 在这里，如果写操作会发生在读操作之前，那么输出必然为10，然而输出为 null.
volatile Integer a = null;
new Thread(new Runnable() {
	@Override
	public void run() {
		System.out.println(a);
	}
}).start();

new Thread(new Runnable() {
	@Override
	public void run() {
		a = 10;
	}
}).start();

// 输出为 Null
```

```java
// 输出为10，因为写操作完成后，happen-before 原则决定了写操作的结果对读操作可见.
volatile Integer a = null;
new Thread(new Runnable() {
	@Override
	public void run() {
			a = 10;
	}
}).start();

new Thread(new Runnable() {
	@Override
	public void run() {
   		System.out.println(a);
	}
}).start();
```

## Volatile 的使用场景

1. 运行结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。
2. 变量不需要与其他的状态变量共同参与不变约束。

```java
volatile static int start = 3;
volatile static int end = 6;
// 线程A执行如下代码：
while (start < end){  
  //do something
}
// 线程B执行如下代码：
start+=3;end+=3;

// 这种情况下，一旦在线程A的循环中执行了线程B，start有可能先更新成6，造成了一瞬间 start == end，从而跳出while循环的可能性。
```



##  内存屏障---Happen-before的底层实现

JVM底层通过"内存屏障", 是一组处理器指令来实现对内存的顺序操作。

内存屏障（Memory Barrier）是一种CPU指令，是一种屏障指令，它使CPU或编译器对屏障指令之前和之后发出的内存操作执行一个排序约束。 这通常意味着在屏障之前发布的操作被保证在屏障之后发布的操作之前执行。
A memory barrier, also known as a membar, memory fence or fence instruction, is a type of barrier instruction that causes a CPU or compiler to enforce an ordering constraint on memory operations issued before and after the barrier instruction. This typically means that operations issued prior to the barrier are guaranteed to be performed before operations issued after the barrier.

具体内存屏障的细节，这里就点到为止，不再展开。



## Reference

[漫画：什么是volatile关键字？](https://zhuanlan.zhihu.com/p/56191979)

[happens-before 俗解](https://zhuanlan.zhihu.com/p/59039455)