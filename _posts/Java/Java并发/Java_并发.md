---
layout: java编程思想
title: Java_并发
date: 2018-03-18 23:06:51
tags: [Java]
---

​	并发通常是提高在单处理器上的程序的性能。

　　这听起来有些违背直觉。如果仔细思考后发现，在单处理器上运行的并发程序开销确实比该程序的所有部分按顺序执行的开销大，因为其中增加了切换上下文的代价。表面上看，将程序的所有部分当作单个任务运行看上去好像是开销小点，并且可以节省切换上下文的代价。

<!-- more -->

　　但在代码实际运行中可能会出现**阻塞**。如果程序中的某个任务因为该程序控制范围之外的某些条件（通常I/O）而导致不能继续执行，那么我们就会说这个任务或线程阻塞了。如果没有并发，则整个程序都将停止下来，直至外部条件发生变化。但是，如果使用并发来编写程序，那么当一个任务阻塞时，程序中其他任务还可以继续执行，因为这个程序可以保持继续向前执行。事实上，从性能角度看，如果没有任务阻塞，那么在单处理器机器上使用并发就没有任何意义。

　	Java的线程机制是**抢占式**的，这表示调度机制会周期性地中断线程，将上下文切换到另一个线程。从而为每个线程都提供时间片，使得每个线程都会分配合理数量的时间去驱动它们的任务。

------



### 1.基本线程实现

```java
public interface Runnable{//线程驱动任务，用Runnable接口描述任务
   void run();
}
```

​	实现一个**Runnable**实例,Runnable接口的实际意义只是Task,进程不与任务绑定，从而增加了代码的可重用性。

```java
public class runnableDemo implements Runnable{
  void run(){
  	 System.out.println();
  }
}
Thread t = new Thread(new runnable());//实际上此时有一个Main进程跟实例化的新进程
t.start();//实例化一个线程，并通过线程start方法调用
```

------



### 2.线程池**(ThreadPool)**

**1.综述**

（1） 使用线程池的好处

- 降低资源消耗。通过重复利用已创的线程降低线程创建和销毁在成的消耗
- 提高响应速度。当任务到达的时候，不需要再去创建线程就能立即执行
- 提高线程的可管理性。线程是稀缺资源，如果无限制的创建不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配、调优和监控。

（2） 线程池的创建:

- ExecutorService是具有生命周期的Executor，知道如何构建恰当的上下文来执行Runnable。
- CachedThreadPool：可缓存线程池，将为每个任务都创建一个线程，并且当线程池大小超过了处理任务所需的线程，那么就会回收空闲线程（60S无执行）
- FixedThreadPool:使用有限的线程集来执行所提交的任务，一次性预先执行代价高昂的线程分配，可以限制线程的数量，可以节省时间因为不用为每一个任务都固定的付出创建线程的开销。
- SingleThreadExecutor：就像是线程数量为1的FixedThreadPool，如果向该线程池提交了多个任务，这些任务将排队，每个任务都会在下一个任务开始之前运行结束（因为采用的是同步队列，一个任务进入线程，就需要一个任务退出线程）所有的任务使用的是同一个线程。会序列化所有的任务，并会维护它自己的悬挂任务队列。

**2.线程核心数量**

* 任务的性质：CPU密集型任务，IO密集型任务和混合型任务。
* 任务的优先级：高，中和低。
* 任务的执行时间：长，中和短。
* 任务的依赖性：是否依赖其他系统资源，如数据库连接。

任务性质不同的任务可以用不同规模的线程池分开处理。

**CPU密集型**任务配置尽可能少的线程数量，如配置**Ncpu+1**个线程的线程池。

**IO密集型**任务则由于需要等待IO操作，线程并不是一直在执行任务，则配置尽可能多的线程，如**2*Ncpu**。

混合型的任务，如果可以拆分，则将其**拆分**成一个CPU密集型任务和一个IO密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐率要高于串行执行的吞吐率，如果这两个任务执行时间相差太大，则没必要进行分解。我们可以通过Runtime.getRuntime().availableProcessors()方法获得当前设备的CPU个数。

**优先级**不同的任务可以使用**优先级队列PriorityBlockingQueue**来处理。它可以让优先级高的任务先得到执行，需要注意的是如果一直有优先级高的任务提交到队列里，那么优先级低的任务可能永远不能执行。

执行时间不同的任务可以交给不同规模的线程池来处理，或者也可以使用优先级队列，让执行时间短的任务先执行。

**依赖数据库连接池**的任务，因为线程提交SQL后需要等待数据库返回结果，如果等待的时间越长CPU空闲时间就越长，那么**线程数应该设置越大**，这样才能更好的利用CPU。

**建议使用有界队列**，有界队列能**增加系统的稳定性和预警能力**，可以根据需要设大一点，比如几千。例如:后台任务线程池的队列和线程池全满了，不断的抛出抛弃任务的异常，通过排查发现是数据库出现了问题，导致执行SQL变得非常缓慢，因为后台任务线程池里的任务全是需要向数据库查询和插入数据的，所以导致线程池里的工作线程全部阻塞住，任务积压在线程池里。如果当时我们设置成无界队列，线程池的队列就会越来越多，有可能会撑满内存，导致整个系统不可用，而不只是后台任务出现问题。当然我们的系统所有的任务是用的单独的服务器部署的，而我们使用不同规模的线程池跑不同类型的任务，但是出现这样问题时也会影响到其他任务。

**3.实现**

```java
  ExecutorService exec = Executors.newCachedThreadPool();
  //ExecutorService exec = Executors.newFixedThreadPool();
  //ExecutorService exec = Executors.newSingleThreadPool();
  for (int i=0;i<5;i++){
       exec.execute(new runnableDemo());
  }
  exec.shutdown();//在所有进程执行完毕后退出
```

**4.关闭线程池**

1. 我们可以通过调用线程池的shutdown或shutdownNow方法来关闭线程池，它们的原理是遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以**无法响应中断的任务可能永远无法终止。**
2. shutdownNow首先将线程池的状态设置成STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表。
3. 而shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。调用该方法，当前线程将继续运行在shutdown()被调用之前所提交的所有任务，再退出。
4. 只要调用了这两个关闭方法的其中一个，isShutdown方法就会返回true。当所有的任务都已关闭后,才表示线程池关闭成功，这时调用isTerminaed方法会返回true。至于我们应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定，通常调用shutdown来关闭线程池，如果任务不一定要执行完，则可以调用shutdownNow。

**5.线程池分析**

当提交一个新任务到线程池时，线程池的处理流程如下：

* 首先线程池判断基本线程池是否已满？没满，创建一个工作线程来执行任务。满了，则进入下个流程。 
* 其次线程池判断工作队列是否已满？没满，则将新提交的任务存储在工作队列里。满了，则进入下个流程。 
* 最后线程池判断整个线程池是否已满？没满，则创建一个新的工作线程来执行任务，满了，则交给饱和策略来处理这个任务。

------



### 3.ThreadPoolExecutor详解

**1.ThreadPoolExecutor的完整构造方法的签名是：**

~~~java
ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler)
~~~

**corePoolSize** - 池中所保存的线程数，包括空闲线程。

**maximumPoolSize**-池中允许的最大线程数。

**keepAliveTime** - 当线程数大于核心时，此为终止前多余的空闲线程等待新任务的最长时间。

**unit** - keepAliveTime 参数的时间单位。

**workQueue** - 执行前用于保持任务的队列。此队列仅保持由 execute方法提交的 Runnable任务。

**threadFactory** - 执行程序创建新线程时使用的工厂。

**handler** - 由于超出线程范围和队列容量而使执行被阻塞时所使用的处理程序。

**2.ThreadPoolExecutor是Executors类的底层实现。**

在JDK帮助文档中，有如此一段话：

“强烈建议程序员使用较为方便的Executors工厂方法Executors.newCachedThreadPool()（无界线程池，可以进行自动线程回收）、Executors.newFixedThreadPool(int),Executors.newSingleThreadExecutor()

它们均为大多数使用场景预定义了设置。”

**源码分析**

~~~java
public static ExecutorService newFixedThreadPool(int nThreads) {   
           return new ThreadPoolExecutor(nThreads, nThreads,   
                                        0L, TimeUnit.MILLISECONDS,   
                                        new LinkedBlockingQueue<Runnable>());   
      }
~~~

corePoolSize和maximumPoolSize的大小是一样的（实际上，后面会介绍，如果使用无界queue的话maximumPoolSize参数是没有意义的）

keepAliveTime和unit的设置标志不想keep alive！

最后的BlockingQueue选择了LinkedBlockingQueue，该queue有一个特点，他是无界的。

~~~java
public static ExecutorService newSingleThreadExecutor() {   
            return new FinalizableDelegatedExecutorService(new ThreadPoolExecutor(1, 1,   
                                       0L, TimeUnit.MILLISECONDS,   
                                      new LinkedBlockingQueue<Runnable>()));   
       }
~~~

同FixedThreadPool,只是将corePoolSize和maximumPoolSize的大小设置为1 

~~~java
public static ExecutorService newCachedThreadPool() {   
             return new ThreadPoolExecutor(0, Integer.MAX_VALUE,   
                                         60L, TimeUnit.SECONDS,   
                                        new SynchronousQueue<Runnable>());   
    }
~~~

无界的线程池，maximumPoolSize为极大值。

其次BlockingQueue的选择上使用SynchronousQueue。可能对于该BlockingQueue有些陌生，简单说：该QUEUE中，每个插入操作必须等待另一个线程的对应移除操作。

**3.queue上的三种类型。**

排队有三种通用策略：

**直接提交。**工作队列的默认选项是 SynchronousQueue，它将任务直接提交给线程而不保持它们。在此，如果不存在可用于立即运行任务的线程，则试图把任务加入队列将失败，因此会构造一个新的线程。

**无界队列。**使用无界队列（例如，不具有预定义容量的 LinkedBlockingQueue）将导致在所有corePoolSize 线程都忙时新任务在队列中等待。这样，创建的线程就不会超过 corePoolSize。（因此，maximumPoolSize的值也就无效了。）当每个任务完全独立于其他任务，即任务执行互不影响时，适合于使用无界队列；例如，在 Web页服务器中。这种排队可用于处理瞬态突发请求，当命令以超过队列所能处理的平均数连续到达时，此策略允许无界线程具有增长的可能性。

**有界队列。**当使用有限的 maximumPoolSizes时，有界队列（如 ArrayBlockingQueue）有助于防止资源耗尽，但是可能较难调整和控制。队列大小和最大池大小可能需要相互折衷：使用大型队列和小型池可以最大限度地降低 CPU 使用率、操作系统资源和上下文切换开销，但是可能导致人工降低吞吐量。如果任务频繁阻塞（例如，如果它们是 I/O边界），则系统可能为超过您许可的更多线程安排时间。使用小型队列通常要求较大的池大小，CPU使用率较高，但是可能遇到不可接受的调度开销，这样也会降低吞吐量。 

**4.三种队列的实际应用**

**基本原则**：如果无法将请求加入队列，则创建新的线程，除非创建此线程超出maximumPoolSize，在这种情况下，任务将被拒绝，并返回异常。

**例一：使用直接提交策略，也即SynchronousQueue**：任务进入队列，判断是否有空余进程，若有则运行。若无空闲进程，则将请求加入队列SynchronousQueue，队列大小只有一，若加入队列失败，则判断基本原则。

**例二：使用无界队列策略，即LinkedBlockingQueue**：同例一，然而最大的区别就是，若使用链表来实现，则队列长度为无穷（除非线程开销耗尽资源）。所以几乎不会触发产生新的线程，就需要设置一个良好CorePoolSize。

**例三：有界队列，使用ArrayBlockingQueue**：最为复杂，看完前面两个，原则相同，自行理解即可。

------



### 4.从任务中产生返回值

**4.1Callable具体实现**

Runnable接口作为执行工作的独立任务，不产生返回值。为了任务完成返回时，实现Callable接口。Callable是一种具有类型参数的泛型。类型参数表示的是方法call()中返回的值，并且使用submit方法。

```java
public class TaskWithResult implements Callable<String>{
    static int count=0;
    private int id = 0;
    public String call() throws Exception {
        id = count++;
        return "TaskWithResult"+id;//call接口返回一个值
    }
}
public static void main(String[] args) {
        ExecutorService exec = Executors.newCachedThreadPool();
        ArrayList<Future<String>> results = new ArrayList<Future<String>>();
        for (int i=0;i<5;i++){
            results.add(exec.submit(new TaskWithResult()));//submit对象会返回Future对象
        }
        for (Future<String> fs:results){
            try {
                System.out.println(fs.get());//调用get来获取结果，若不使用isDone()来检查，get()将阻塞，直到结果准备就绪。
            }catch (InterruptedException e){
                System.err.println(e);
            }catch (ExecutionException e){
                System.err.println(e);
            }finally {
                exec.shutdown();
            }
        }
    }
```

**4.2Future对象**

~~~java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
~~~

* **cancel**:用于取消任务，参数mayInterruptIfRunning,true则可以取消正在执行的任务

* **isCancelled**:用于判定任务是否取消成功

* **isDone**:用于表示任务是否完成

* **get()**：用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回

* **get(long timeout, TimeUnit unit)**用来获取执行结果，如果在指定时间内，还没获取到结果，就直接返回null

  然而Future只是一个接口，不能直接用来创建对象使用。

**4.3FutureTask**

~~~java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}//RunnableFuture接口继承了Runnable,Future接口
public class FutureTask<V> implements RunnableFuture<V>
//既可以作为Runnable被线程执行，又可以作为Future得到Callable的返回值
~~~

------



### 5.休眠

TimeUnit下的方法进行休眠，语义化更强。而实际上TimeUnit包调用的也是Thread.sleep

调用Sleep方法会抛出InterruptedException，因为**异常不能跨线程传播会Main方法**，因此必须在本地处理内部所产生的异常。

```java
try {
 	// Old-style:
    // Thread.sleep(100)
    // Java SE5/6-style:
       TimeUnit.MILLISECONDS.sleep(100);
    } catch(InterruptedException e) {
      System.err.println("Interrupted");
    }
```

------



### 6.优先级 && 让步 

线程的优先级将该线程的重要性传递给调度器。尽管CPU处理现有线程集的顺序是不确定的，但是调度器将倾向于让优先级最高的线程来执行。然而，这并不意味着线程优先级低的线程将得不到执行。优先级较低的线程仅仅只是线程执行的没那么频繁。**因此不会造成线程阻塞**

在绝大多数时间里，所有线程都应该按照默认优先级执行，试图操作线程优先级通常是一种错误。

可以用getPriority()来读取现有线程的优先级，并且在任意时刻都可以通过setPriority()来修改它。

```java
public class SimplePriorities implements Runnable {
  private int countDown = 5;
  private volatile double d; // No optimization
  private int priority;
  public SimplePriorities(int priority) {
    this.priority = priority;
  }
  public String toString() {//重写toString方法
    return Thread.currentThread() + ": " + countDown;
  }
  public void run() {
    Thread.currentThread().setPriority(priority);//设置当前进程优先级
    while(true) {
      // An expensive, interruptable operation:
      for(int i = 1; i < 100000; i++) {
        d += (Math.PI + Math.E) / (double)i;
        if(i % 1000 == 0)
          Thread.yield();	//让步 实际上只是一个建议，任何有实际确定功能的进程切换都不能依赖于让步
      }
      System.out.println(this);
      if(--countDown == 0) 
        return;
    }
  }
  public static void main(String[] args) {
    ExecutorService exec = Executors.newCachedThreadPool();
    for(int i = 0; i < 5; i++)
      exec.execute(new SimplePriorities(Thread.MIN_PRIORITY));
   	  //exec.execute(new SimplePriorities(Thread.MAX_PRIORITY));
      //exec.execute(new SimplePriorities(Thread.NORM_PRIORITY));
      exec.shutdown();
  }
} 
```

jdk与OP的进程优先级映射有问题。因此建议使用三种优先级即可

------



### 7.后台进程

1. 所谓后台线程是指在程序运行的时候在后台提供一种通用服务的线程，并且这种线程并不属于程序中不可或缺的部分。**当所有的非后台线程结束时，程序也就终止，同时会杀死进程中所有的后台进程。**
2. 线程必须在启动之前调用setDaemon()方法，才能把它设置为后台线程。
3. 后台进程在**不执行finally子句的情况下就会终止其run()方法**

```java
 Thread thread = new Thread(new SimpleDaemons());
 thread.setDaemon(true);//后台进程设置需要在进程启动前调用
 thread.start();
```

------



### 8.Join

一个线程可以调用其他线程的join()方法，其效果是等待其他线程结束才继续执行。如果某个线程调用t.join()，此线程将被挂起，直到目标线程t结束才恢复（即t.isAlive()为假）。

对join()方法的调用可以被中断，做法就是在调用线程上调用interrupt()方法，即t.interrupt()。例子如下：

```java
public class Joiner extends Thread{
    private Sleeper sleeper;
    public Joiner(String name, Sleeper sleeper){
        super(name);
        this.sleeper = sleeper;
        start();
    }
    public void run() {
        try {
            sleeper.join();//将自身进程挂起,知道Sleeper执行完
        } catch (Exception e) {
            System.out.println("Interupted");
        }
        System.out.println(getName() + " join completed");
    }
}

package com.xqq.join;

public class Sleeper extends Thread{

    private int duration;
    public Sleeper(String name, int sleeperTime){
        super(name);
        duration = sleeperTime;
        start();
    }

    public void run() {
        try {
            sleep(duration);
        } catch (Exception e) {
            System.out.println(getName() + " was interrupted. " + "isInterupted:"+isInterrupted());
            return;
        }
        System.out.println(getName() + " has awakened");
    }
}

/**
 * 如果某一个线程在另一个线程t上调用t.join()，此线程将被挂起，直到线程t运行结束
 * 对join方法的调用可以被中断，在调用线程上调用interrupt()方法。
 * @author zibu
 */
public class Joining {
    public static void main(String[] args) {
        Sleeper sleepy = new Sleeper("Sleepy", 1500);
        Sleeper grumpy = new Sleeper("Grumpy", 1500);

        new Joiner("Dopey", sleepy);
        new Joiner("Doc", grumpy);
        grumpy.interrupt();//grumpy被终止，随后Doc开始执行
    }
}
运行结果：
Grumpy was interrupted. isInterupted: false
Doc join completed
Sleepy has awakened
Dopey join completed
```
