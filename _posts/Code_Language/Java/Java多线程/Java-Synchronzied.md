---
title: Java_Synchronzied
date: 2018-12-28 14:59:53
tags: [Java, 多线程]
---

> synchronized 是 Java 中的关键字，是利用**锁机制来实现同步**的。
>
> 本文将介绍synchronized锁的相关概念，辅以充足的代码示例。深入浅出介绍synchronized锁的使用。
>
> 最后研究其实现原理。加深对synchronized机制的理解

------

<!--more-->

## synchronized 概念篇

### 锁机制的特性

锁机制有如下两种特性：

- **互斥性**：即在**同一时间只允许一个线程持有锁**，因此被锁住的代码，实现了互斥操作。互斥性我们也往往称为操作的原子性。
- **可见性**：必须确保在锁被释放之前，对共享变量所做的修改，对于随后获得该锁的另一个线程是可见的（即在获得锁时应获得最新共享变量的值）。

### synchronized锁的类型

分为两种锁，一种为对象锁，另一种为类锁。

* **对象锁**：在Java中，每个对象都会有一个monitor对象，通常会被称为“内置锁”或“对象锁”。类的对象可以有多个，所以每个对象有其独立的对象锁，互不干扰。
* **类锁**：在Java中，针对每个类也有一个锁，被称作“类锁”，类锁实际上是通过对象锁实现的，即类的 Class 对象锁。每个类只有一个 Class 对象，所以每个类只有一个类锁。

### synchronized的使用方式

#### 根据修饰对象分类

* 修饰代码块
  * synchronized(this){}         类对象锁
  * synchronized(obejct){}    对象锁
  * synchronized(类.class){}  类锁

* 修饰方法
  * 修饰非静态方法		     类对象锁
  * 修饰静态方法                      类锁

#### 根据获得的锁分类

* 类锁
  * 修饰静态方法
  * synchronized(类.class){}

* 对象锁
  * synchronized(this){}         类对象锁
  * 修饰非静态方法		     类对象锁
  * synchronized(obejct){}     对象锁

------



## synchronized 使用篇

### 获取对象锁

>  多线程访问不同对象实例的synchronize方法块，并不会互相干扰。不同实例存在于堆内存不同区域。

#### 测试代码

```java
public class SyncThread implements Runnable {
	
    private static Integer count = 1;
    
    @Override
    public void run() {
        String threadName = Thread.currentThread().getName();
        if (threadName.startsWith("A")) {
            sync1();
        } else if (threadName.startsWith("B")) {
            sync2();
        } else if (threadName.startsWith("C")) {
            sync3();
        }
    }

    /**
     * 方法中有 synchronized(this|object) {} 同步代码块
     */
    private void sync1() {
        System.out.println(Thread.currentThread().getName() + "_Sync1: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
        synchronized (this) {
            try {
                System.out.println(Thread.currentThread().getName() + "_Sync1_Start: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                Thread.sleep(2000);
                System.out.println(Thread.currentThread().getName() + "_Sync1_End: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * synchronized 修饰非静态方法
     */
    private synchronized void sync2() {
        System.out.println(Thread.currentThread().getName() + "_Sync2: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
        try {
            System.out.println(Thread.currentThread().getName() + "_Sync2_Start: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
            Thread.sleep(2000);
            System.out.println(Thread.currentThread().getName() + "_Sync2_End: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    
    // 获取count类的对象锁
     private void sync3(){
        System.out.println(Thread.currentThread().getName() + "_Sync3: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
        synchronized (count) {
            try {
                System.out.println(Thread.currentThread().getName() + "_Sync3_Start: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                Thread.sleep(2000);
                System.out.println(Thread.currentThread().getName() + "_Sync3_End: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
		// 测试代码
		SyncThread syncThread = new SyncThread();
        Thread A_thread1 = new Thread(syncThread, "A_thread1");
        Thread A_thread2 = new Thread(syncThread, "A_thread2");
        Thread B_thread1 = new Thread(syncThread, "B_thread1");
        Thread B_thread2 = new Thread(syncThread, "B_thread2");
		Thread C_thread1 = new Thread(syncThread, "C_thread1");
        Thread C_thread2 = new Thread(syncThread, "C_thread2");
        A_thread1.start();
        A_thread2.start();
        B_thread1.start();
        B_thread2.start();
	    C_thread1.start();
        C_thread2.start();
```

#### 测试分析

```java
// 线程A1，A2同时访问了sync1方法 
A_thread2_Sync1: 21:38:03
A_thread1_Sync1: 21:38:03
// 线程B1同C1同时执行 
B_thread1_Sync2: 21:38:03
B_thread1_Sync2_Start: 21:38:03
B_thread1_Sync2_End: 21:38:05
// 线程A1也获取了类的对象锁 执行
A_thread1_Sync1_Start: 21:38:05
A_thread1_Sync1_End: 21:38:07
// 线程A2
A_thread2_Sync1_Start: 21:38:07
A_thread2_Sync1_End: 21:38:09
// 线程B2    
B_thread2_Sync2: 21:38:09
B_thread2_Sync2_Start: 21:38:09
B_thread2_Sync2_End: 21:38:11
 
// 特地与A和B分开显示 为了更直观    
    
// 线程C1，线程C2也访问了sync3方法
C_thread1_Sync3: 21:38:03
C_thread2_Sync3: 21:38:03
// 线程C1先执行 
C_thread1_Sync3_Start: 21:38:03
C_thread1_Sync3_End: 21:38:05
// 线程C2接着执行
C_thread1_Sync3_Start: 21:38:05
C_thread1_Sync3_End: 21:38:07
```

由结果可知A类和B类线程顺序执行，

* A类：使用synchronized()修饰**方法内代码块**
* B类：使用synchronized修饰**非静态类成员方法**
* C类：使用synchronized，同步对象锁为，**类静态成员**

AB虽然修饰的东西不同，但是由结果可知它们获取的是相同的锁，既**该类的对象锁**。

C是独立与AB执行的，同样获得了对象锁。但是**类静态成员的对象锁。**

### 获取类锁

#### 测试代码

```java
public class SyncThread1 implements Runnable{

    @Override
    public void run() {
        String threadName = Thread.currentThread().getName();
        if (threadName.startsWith("A")) {
            sync1();
        } else if (threadName.startsWith("B")) {
            sync2();
        }
    }

    /**
     * 方法中有 synchronized(类.class) {} 同步代码块
     */
    private void sync1() {
        System.out.println(Thread.currentThread().getName() + "_Sync1: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
        synchronized (SyncThread1.class) {
            try {
                System.out.println(Thread.currentThread().getName() + "_Sync1_Start: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                Thread.sleep(2000);
                System.out.println(Thread.currentThread().getName() + "_Sync1_End: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }


    /**
     * synchronized 修饰静态方法
     */
    private synchronized static void sync2() {
        try {
            System.out.println(Thread.currentThread().getName() + "_Sync2_Start: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
            Thread.sleep(2000);
            System.out.println(Thread.currentThread().getName() + "_Sync2_End: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

```

```java
		// 测试代码
		// 每个线程分别执行各自的Runnable对象
		// 同一对象不再测试，因为同一对象连对象锁都是顺序获取，更何况类锁高于对象锁
		Thread A_thread1 = new Thread(new SyncThread1(), "A_thread1");
        Thread A_thread2 = new Thread(new SyncThread1(), "A_thread2");
        Thread B_thread1 = new Thread(new SyncThread1(), "B_thread1");
        Thread B_thread2 = new Thread(new SyncThread1(), "B_thread2");
        A_thread1.start();
        A_thread2.start();
        B_thread1.start();
        B_thread2.start();
```

#### 测试分析

```java
// B1线程先获取类锁并执行
B_thread1_Sync2_Start: 22:16:41
// A1,A2也进入方法 当前未获得类锁
A_thread2_Sync1: 22:16:41
A_thread1_Sync1: 22:16:41
// B1线程执行完毕
B_thread1_Sync2_End: 22:16:43
// A1线程获取类锁
A_thread1_Sync1_Start: 22:16:43
A_thread1_Sync1_End: 22:16:46
// A2线程获取类锁
A_thread2_Sync1_Start: 22:16:46
A_thread2_Sync1_End: 22:16:48
// B2线程获取类锁
B_thread2_Sync2_Start: 22:16:48
B_thread2_Sync2_End: 22:16:50
```

结果分析

* A类：使用synchornized(类.class)修饰代码块
* B类：使用synchornized修饰静态方法

有结果可知，A类和B类获取的都是同一个锁，因此顺序执行。这个锁呢，就是**类锁。同一个类的不同实例化对象，有着不同的类对象锁，但是拥有相同的类锁。**



### 同时获取类锁和类对象锁

#### 测试代码

```java
public class SyncThread2 implements Runnable{

    @Override
    public void run() {
        String threadName = Thread.currentThread().getName();
        if (threadName.startsWith("A")) {
            sync1();
        } else if (threadName.startsWith("B")) {
            sync2();
        }
    }

    /**
     * 方法中有 synchronized(this) {} 同步代码块
     */
    private void sync1() {
        synchronized (this) {
            try {
                System.out.println(Thread.currentThread().getName() + "_Sync_Start: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
                Thread.sleep(2000);
                System.out.println(Thread.currentThread().getName() + "_Sync_End: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }


    /**
     * synchronized 修饰静态方法
     */
    private synchronized static void sync2() {
        try {
            System.out.println(Thread.currentThread().getName() + "_Sync_Start: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
            Thread.sleep(2000);
            System.out.println(Thread.currentThread().getName() + "_Sync_End: " + new SimpleDateFormat("HH:mm:ss").format(new Date()));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

```java
		// 测试代码
		SyncThread2 syncThread2 = new SyncThread2();
        Thread A_thread = new Thread( syncThread2, "A_thread");
        Thread B_thread = new Thread( syncThread2, "B_thread");
        A_thread.start();
        B_thread.start();
```

#### 测试分析

```java
// 分别执行
B_thread_Sync_Start: 22:31:48
A_thread_Sync_Start: 22:31:48
B_thread_Sync_End: 22:31:50
A_thread_Sync_End: 22:31:50
```

结果显示，A线程与B线程同时执行了

* A：类对象锁
* B：类锁

**因此可以得出，类对象锁与类锁并不冲突。**

------



## synchronized 原理篇

### 1. synchronized代码块

```java
public class SynchronizedDemo {
	public void method() {
		synchronized (this) {
			System.out.println("synchronized 代码块");
		}
	}
}
```

```shell
javac SynchronizedDemo
# 对于生成的字节码 我们使用IDEA打开 会直接帮助我们反编译字节码为JAVA代码
# 而javap 命令能够帮助我们直接看到字节码
javap -c -s -v -l SynchronizedDemo.class
# 节选部分Method方法内容字节码
Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3                  // String synchronized 代码块
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

我们可以看到字节码中存在**monitorenter**和**monitorexit**指令。**其中 monitorenter 指令指向同步代码块的开始位置，monitorexit 指令则指明同步代码块的结束位置。**

当执行 monitorenter 指令时，线程试图获取锁也就是获取 monitor(**monitor对象存在于每个Java对象的对象头中**，synchronized 锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因) 的持有权.当计数器为0则可以成功获取，获取后将锁计数器设为1也就是加1。相应的在执行 monitorexit 指令后，将锁计数器设为0，表明锁被释放。如果获取对象锁失败，那当前线程就要阻塞等待，直到锁被另外一个线程释放为止。

### 2. synchronize方法

```java
public class SynchronizedDemo {
    public synchronized void method() {
            System.out.println("synchronized 代码块");
    }
}
```

```shell
public synchronized void method();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String synchronized 代码块
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 5: 0
        line 6: 8
}
```

这次是同步方法的字节码，与同步代码不同之处在于。method方法的flag属性多了**ACC_SYNCHRONIZED**标识。JVM 通过该 ACC_SYNCHRONIZED 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

在 Java 早期版本中，synchronized 属于重量级锁，效率低下，因为监视器锁（**monitor**）是**依赖于底层的操作系统的 Mutex Lock** 来实现的，Java 的线程是映射到操作系统的原生线程之上的。如果要挂起或者唤醒一个线程，都需要操作系统帮忙完成，而操作系统实现线程之间的切换时需要从用户态转换到内核态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的 synchronized 效率低的原因。

------



## JDK1.6 之后的底层优化

JDK1.6 对锁的实现引入了大量的优化，如偏向锁、轻量级锁、自旋锁、适应性自旋锁、锁消除、锁粗化等技术来减少锁操作的开销。

锁主要存在四种状态，依次是：**无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态**，他们会随着竞争的激烈而逐渐升级。注意**锁可以升级不可降级**，这种策略是为了提高获得锁和释放锁的效率。

### **1. 偏向锁**——无竞争环境下，消除同步。否则升级为轻量级锁

引入偏向锁的目的和引入轻量级锁的目的很像，都是为了**在没有多线程竞争的前提下**，**减少传统的重量级锁使用操作系统互斥量(Mutex Lock)产生的性能消耗。**

但是不同是：

* **轻量级锁在无竞争的情况下使用 CAS 操作去代替使用互斥量。**
* **而偏向锁在无竞争的情况下会把整个同步都消除掉**。

偏向锁的“偏”就是偏心的偏，它的意思是会偏向于第一个获得它的线程，如果在接下来的执行中，该锁没有被其他线程获取，那么持有偏向锁的线程就不需要进行同步！关于偏向锁的原理可以查看《深入理解Java虚拟机：JVM高级特性与最佳实践》第二版的13章第三节锁优化P402。

在出现锁竞争情况下，偏向锁就失效了。会升级为轻量级锁。

### **2. 轻量级锁**

轻量级锁不是为了代替重量级锁，它的本意是**在没有多线程竞争的前提下**，**减少传统的重量级锁使用操作系统互斥量（Mutex Lock）产生的性能消耗**，

因为使用轻量级锁时，不需要申请互斥量。而是使用了**CAS操作来实现轻量级锁的加锁和解锁。** 关于轻量级锁的加锁和解锁的原理可以查看《深入理解Java虚拟机：JVM高级特性与最佳实践》第二版的13章第三节锁优化。

**轻量级锁能够提升程序同步性能的依据是“对于绝大部分锁，在整个同步周期内都是不存在竞争的”，这是一个经验数据。如果没有竞争，轻量级锁使用 CAS 操作避免了使用互斥操作的开销。但如果存在锁竞争，除了互斥量开销外，还会额外发生CAS操作，因此在有锁竞争的情况下，轻量级锁比传统的重量级锁更慢！如果锁竞争激烈，那么轻量级将很快膨胀为重量级锁！**

### **3. 自旋锁和自适应自旋**

轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。

互斥同步对性能最大的影响就是阻塞的实现，因为挂起线程/恢复线程的操作都需要转入内核态中完成（用户态转换到内核态会耗费时间）。

**一般线程持有锁的时间都不是太长，所以仅仅为了这一点时间去挂起线程/恢复线程是得不偿失的。**

**为了让一个线程等待，我们只需要让线程执行一个忙循环（自旋），这项技术就叫做自旋**。

百度百科对自旋锁的解释：

> 何谓自旋锁？它是为实现保护共享资源而提出一种锁机制。其实，自旋锁与互斥锁比较类似，它们都是为了解决对某项资源的互斥使用。无论是互斥锁，还是自旋锁，在任何时刻，最多只能有一个保持者，也就说，在任何时刻最多只能有一个执行单元获得锁。但是两者在调度机制上略有不同。对于互斥锁，如果资源已经被占用，资源申请者只能进入睡眠状态。但是自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁，"自旋"一词就是因此而得名。

JDK1.6之后，自旋锁为默认开启。需要注意的是：自旋等待不能完全替代阻塞，因为它还是要占用处理器时间。如果锁被占用的时间短，自旋锁能起到很好的效果！自旋等待的时间必须要有限度。如果自旋超过了限定次数任然没有获得锁，就应该挂起线程。**自旋次数的默认值是10次，用户可以修改--XX:PreBlockSpin来更改**。

另外,**在 JDK1.6 中引入了自适应的自旋锁。自适应的自旋锁带来的改进就是：自旋的时间不在固定了，而是和前一次同一个锁上的自旋时间以及锁的拥有者的状态来决定**。

### **4. 锁消除**

锁消除理解起来很简单，它指的就是虚拟机即时编译器在运行时，如果**检测到那些共享数据不可能存在竞争**，那么就执行锁消除。锁消除可以节省毫无意义的请求锁的时间。

锁消除的判定依据主要来自于`逃逸分析`，如果判断在一段代码种，堆中的数据不会逃逸出去被其他线程访问，那就可以把它们当做栈上数据对待，认为他们为线程私有，那就不需要同步加锁了。

问题在于：这些无需同步的代码，程序员为什么要加锁呢？

因为很多锁是源码上的锁，开发人员并不会注意到这些。然而同步代码在Java中又是非常普遍存在的，因此锁消除机制可以在编译过程中极大的优化消除无用的锁，提高代码性能。

### **5. 锁粗化**

原则上，我们在编写代码的时候，总是**推荐将同步块的作用范围限制得尽量小**——只在共享数据的实际作用域才进行同步，这样是为了尽量减少所需要的同步操作，如果存在锁竞争，那等待线程也能尽快拿到锁。

大部分情况下，上面的原则都是没有问题的，但是如果一系列的连续操作都对同一个对象反复加锁和解锁，那么会带来很多不必要的性能消耗。

例如循环体内，对每一次循环内的操作进行锁同步操作，那么将会消耗相当多的资源来做这件事情。

因此**当JVM探测到一系列琐碎的操作对同一对象加锁时，会将加锁同步的范围扩展（粗化）到整个操作序列的外部。**

## Synchronized 和 ReenTrantLock 的对比

### **1. 两者都是可重入锁**

`可重入锁`概念是：自己可以再次获取自己的内部锁。比如一个线程获得了某个对象的锁，此时这个对象锁还没有释放，当其再次想要获取这个对象的锁的时候还是可以获取的，**如果不可锁重入的话，就会造成死锁**。同一个线程每次获取锁，锁的计数器都自增1，所以要等到锁的计数器下降为0时才能释放锁。

### **2. synchronized 依赖于 JVM 而 ReenTrantLock 依赖于 API**

synchronized 是依赖于 JVM 实现的，前面我们也讲到了 虚拟机团队在 JDK1.6 为 synchronized 关键字进行了很多优化，但是这些优化都是在虚拟机层面实现的，并没有直接暴露给我们。ReenTrantLock 是 JDK 层面实现的（也就是 API 层面，需要 lock() 和 unlock 方法配合 try/finally 语句块来完成），所以我们可以通过查看它的源代码，来看它是如何实现的。

### **3. ReenTrantLock 比 synchronized 增加了一些高级功能**

相比synchronized，ReenTrantLock增加了一些高级功能。主要来说主要有三点：**①等待可中断；②可实现公平锁；③可实现选择性通知（锁可以绑定多个条件）**

- **ReenTrantLock提供了一种能够中断等待锁的线程的机制**，通过lock.lockInterruptibly()来实现这个机制。也就是说正在等待的线程可以选择放弃等待，改为处理其他事情。
- **ReenTrantLock可以指定是公平锁还是非公平锁。而synchronized只能是非公平锁。所谓的公平锁就是先等待的线程先获得锁。** ReenTrantLock默认情况是非公平的，可以通过 ReenTrantLock类的`ReentrantLock(boolean fair)`构造方法来制定是否是公平的。
- synchronized关键字与wait()和notify/notifyAll()方法相结合可以实现等待/通知机制，ReentrantLock类当然也可以实现，但是需要借助于Condition接口与newCondition() 方法。Condition是JDK1.5之后才有的，它具有很好的灵活性，比如可以实现多路通知功能也就是在一个Lock对象中可以创建多个Condition实例（即对象监视器），**线程对象可以注册在指定的Condition中，从而可以有选择性的进行线程通知，在调度线程上更加灵活。 在使用notify/notifyAll()方法进行通知时，被通知的线程是由 JVM 选择的，用ReentrantLock类结合Condition实例可以实现“选择性通知”** ，这个功能非常重要，而且是Condition接口默认提供的。而synchronized关键字就相当于整个Lock对象中只有一个Condition实例，所有的线程都注册在它一个身上。如果执行notifyAll()方法的话就会通知所有处于等待状态的线程这样会造成很大的效率问题，而Condition实例的signalAll()方法 只会唤醒注册在该Condition实例中的所有等待线程。

如果你想使用上述功能，那么选择ReenTrantLock是一个不错的选择。

### **4. 性能已不是选择标准**

在JDK1.6之前，synchronized 的性能是比 ReenTrantLock 差很多。具体表示为：synchronized 关键字吞吐量岁线程数的增加，下降得非常严重。而ReenTrantLock 基本保持一个比较稳定的水平。我觉得这也侧面反映了， synchronized 关键字还有非常大的优化余地。后续的技术发展也证明了这一点，我们上面也讲了在 JDK1.6 之后 JVM 团队对 synchronized 关键字做了很多优化。**JDK1.6 之后，synchronized 和 ReenTrantLock 的性能基本是持平了。所以网上那些说因为性能才选择 ReenTrantLock 的文章都是错的！JDK1.6之后，性能已经不是选择synchronized和ReenTrantLock的影响因素了！而且虚拟机在未来的性能改进中会更偏向于原生的synchronized，所以还是提倡在synchronized能满足你的需求的情况下，优先考虑使用synchronized关键字来进行同步！优化后的synchronized和ReenTrantLock一样，在很多地方都是用到了CAS操作**。





------

## 补充说明

1. synchronized关键字不能继承。

   对于父类的synchronized方法，子类覆盖该方法，默认情况下不会同步，需要显示的使用synchronized关键词进行修饰。

2. synchronized不能用于定义接口方法
3. 构造方法不能用synchronized修饰，但是构造方法内部可以使用synchronized修饰代码块。

------



## 相关资料

[Java 之 synchronized 详解](https://juejin.im/post/594a24defe88c2006aa01f1c#heading-10)

[Java Guide synchronized.md](https://github.com/Snailclimb/JavaGuide/blob/master/Java%E7%9B%B8%E5%85%B3/synchronized.md)

《深入理解Java虚拟机》