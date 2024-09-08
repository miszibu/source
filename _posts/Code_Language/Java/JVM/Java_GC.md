---
title: Java_GC
date: 2018-03-29 14:44:32
tags: [Java]
---

相比较C++需要在程序中自行处理内存的分配和回收。Java在JVM虚拟机上增加了GC机制，用以在合适的时间通过垃圾回收，回收不需要的数据，从而减少内存占用，避免OOM(Out Of Memory)。

掌握GC知识一方面可以帮助我们快速排查因JVM导致的线上问题，另一方面也可以帮助我们在Java应用发布之前合理地对JVM进行调优，提高应用的执行效率、可靠性和健壮性。

<!--more-->

## Java堆内存结构

Java堆内存分三大部分:**新生代(Eden,Survivor0,Survivor1)，老年代和永久代**。

从 GC 的角度而言，我们**只会处理新生代和老年代的数据**

```java
+----------------------------+----------------------------------+--------------+
|                  |    |    |                                  |              |
|        Eden      | S0 | S1 |          Old generation          |     Perm     |
|        8/10      |1/10|1/10|                                  |  Class和 Meta|
+----------------------------+----------------------------------+--------------+
|<----Young Gen Space------->|                                   
|<---- 1/3 Total Heap------->||<------- 2/3 Total Heap--------->|<---- 4MB ---->
```

### 新生代：

用于存放新生的对象，一般占据堆的1/3空间，由于频繁的创建对象，新生代会频繁触发 MinorGC 进行垃圾回收。

* Eden 区域：新对象的存放的地方（如果新创建的对象占用内存很大，则直接分配到老年代。**当 Eden区域不足以分配新对象时，触发 MinorGC**
* Survivor0/Survivor1: 各自占用年轻代1/10的堆内存，用于辅助MinorGC 
* **MinorGC**: 基于**复制算法**，每次回收 Eden 区+一个 Survivor 区，并将存活的对象存放到另外一个 Survivor 区，并将这些对象的年龄加一。
  * 如果对象的年龄满足进入老年代的标准（默认年龄为15），则复制到老年代。
  * 如果存活的对象超过 Survivor 区的最大容量，则将超额对象直接放入老年代。

### 老年代：

用于存放程序中生命周期较长的对象。

老年代的对象相对稳定，MajorGC 不会频繁执行，由于数据不会直接在老年代创建，往往是 **MinorGC 将部分晋升的数据放入老年代时，发现内存空间不足，**才会触发 MajorGC。或者当**没有足够大的连续空间**分配给直接创建在老年代的对象时，也会**触发 MajroGC**。

> 并非所有的对象创建都会在Eden区中分配内存空间。对于Serial和ParNew垃圾收集器,通过指定-XX:PretenureSizeThreshold={size}来设置超过这个阈值大小的对象直接进入老年代。

MajorGC 采用**标记清除算法**，首先扫描一次所有老年代，标记出存活的对象，然后回收没 有标记的对象。MajorGC 的耗时比较长，因为要扫描再回收。MajorGC 会产生内存碎片，为了减 少内存损耗，我们一般需要进行合并或者标记出来方便下次直接分配。当老年代也满了装不下的 时候，就会抛出 OOM（Out of Memory）异常。

### 永久代:

指内存的永久保存区域，主要存放 Class 和 Meta（元数据）的信息,Class 在被加载的时候被 放入永久区域，它和和存放实例的区域不同,GC 不会在主程序运行期对永久区域进行清理。所以这 也导致了永久代的区域会随着加载的 Class 的增多而涨满，最终抛出 OOM 异常。

> 在 Java8中，永久代已经被移除，被一个称为“元数据区”（元空间）的区域所取代。元空间 的本质和永久代类似，元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用 **本地内存**。因此，默认情况下，元空间的大小仅受本地内存限制。类的元数据放入 native memory, 字符串池和类的静态变量放入 java 堆中，这样可以加载多少类的元数据就不再由 MaxPermSize 控制, 而由系统的实际可用空间来控制



---



## 如何确认垃圾

### 引用计数法Reference Counting Collector

在 Java 中，引用和对象是有关联的。如果要操作对象则必须用引用进行。通过引用计数来判断一个对象是否可以回收。如果一个对象如果没有任何与之关联的引用，即他们的引用计数为 0，则说明对象不太可能再被用到，那么这个对象就是可回收对象。

**缺点：存在循环引用问题**

四种引用状态

**1、强引用**

代码中普遍存在的类似"Object obj = new Object()"这类的引用，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象。

**2、软引用**

描述有些还有用但并非必需的对象。在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围进行二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。Java中的类SoftReference表示软引用。

**3、弱引用**

描述非必需对象。被弱引用关联的对象只能生存到下一次垃圾回收之前，垃圾收集器工作之后，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。Java中的类WeakReference表示弱引用。

**4、虚引用**

这个引用存在的唯一目的就是在这个对象被收集器回收时收到一个系统通知，被虚引用关联的对象，和其生存时间完全没关系。Java中的类PhantomReference表示虚引用。

![](\Java_GC/四种引用.png)



----

### 根搜索算法-----**可达性分析法**

这个算法的基本思想是通过一系列称为“GC Roots”的对象作为起始点，从这些节点向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链（即GC Roots到对象不可达）时，则证明此对象是不可用的。

在Java语言中，可以作为GCRoots的对象包括下面几种：

* **虚拟机栈**（栈帧中的局部变量区，也叫做局部变量表）中引用的对象。

* 方法区中的类静态属性/常量引用的对象。

* 本地方法栈中JNI(Native方法)引用的对象。

![可达性分析法](./Java_GC/可达性分析法.png)



## 垃圾清除算法

垃圾回收主要针对Java堆内存中的新生代和老年代，不同生代的垃圾回收采用不同的回收算法。

针对新生代，主要采用**复制算法**，而针对老年代，通常采用**标记-清除算法**或者**标记-整理算法**来进行回收。

### 标记-清除算法

算法分为两个阶段：**标记，清除**

在**标记阶段**将标记出需要回收的对象空间，然后在**清除阶段**里面，将这些标记出来的对象空间回收掉。

缺点：内存碎片化严重，这样会导致在分配大对象时候无法找到足够的连续内存而触发另一次GC。

![标记-清除算法图解](./Java_GC/show-mark-sweep-algorithm.png)



### 复制算法

复制算法的思想是将内存分成大小相等的两块区域，每次使用其中的一块。当这一块的内存用完了，就将还存活的对象复制到另一块区域上，然后对该块进行内存回收。

优点：实现简单，且不易产生碎片

缺点: 只使用了一半的内存空间，且存活对象多，拷贝对象的效率较低。

![复制算法图解](./Java_GC/show-copy-algorithm.png)





> 有研究表明，新生代中的对象98%存活期很短，所以并不需要以1:1的比例来划分整个新生代，通常的做法是将新生代内存空间划分成一块较大的Eden区和两块较小的Survivor区，两块Survivor区域的大小保持一致。每次使用Eden和其中一块Survivor区，当回收的时候，将还存活的对象复制到另外一块Survivor空间上，最后清除Eden区和一开始使用的Survivor区。
>
> 假如用于复制的Survivor区放不下存活的对象，那么会将对象存到老年代。
>
> 新生代对象生命周期短的特点，完美契合了复制算法。



### 标记-整理算法

标记-整理(Mark-Compact)算法有效预防了标记-清除算法中可能产生过多内存碎片的问题。在标记需要回收的对象以后，它会将所有存活的对象空间挪到一起，然后再执行清理。示例图如下所示：

![标记-整理算法图解](./Java_GC/show-mark-compact-algorithm.png)

> 标记-整理通常会在标记-清除算法里面作为备选方案，为了防止标记-清除后产生大量内存碎片而无法为大对象分配足够内存的情况，如后面所讲的Serial Old收集器(基于标记-整理算法实现)将会作为CMS收集器(基于标记-清除算法实现)的备选方案。



## 新生代收集器

### Serial收集器（单线程，复制算法）

Serial收集器作用于新生代，是一个单线程收集器，基于复制算法实现。在进行垃圾回收的时候仅使用单线程并且在回收的过程中会挂起所有的用户线程(Stop The World)。

Serial收集器是**JVM client模式下默认的新生代收集器。**

因为在用户的桌面应用场景中，往往分配给 JVM的内存有限，这样 STW 造成的停顿时间往往控制在几百毫秒以内，这个场景中使用 Serial 收集器是不错的选择。

![](./Java_GC/serial.jpeg)

> 特别注意，Stop-The-World会挂起应用线程，造成应用的停顿。



### ParNew收集器（多线程的 Serial）

**ParNew**收集器就是Serial收集器的多线程版本，它也是一个**新生代收集器**。除了使用多线程进行垃圾收集外，其余行为包括Serial收集器可用的所有控制参数、收集算法（复制算法）、Stop The World、对象分配规则、回收策略等与Serial收集器完全相同，两者共用了相当多的代码。

ParNew收集器的工作过程如下图（老年代采用Serial Old收集器）：

![](./Java_GC/parnew.jpeg)

ParNew 垃圾收集器是很多 Java 虚拟机在 **Server 模式下的默认垃圾收集器**，因为**除了Serial收集器外，目前只有它能和CMS收集器（Concurrent Mark Sweep）配合工作**

> ​	ParNew 收集器默认开启和 CPU 数目相同的线程数，可以通过-XX:ParallelGCThreads 参数来限 制垃圾收集器的线程数





### Parallel Scavenge收集器(多线程复制算法，关注吞吐量)

Parallel Scavenge收集器同样作用于新生代，并且也是采用多线程和复制算法来进行垃圾回收。Parallel Scavenge收集器关注的是**吞吐量**，即使得应用能够充分使用CPU。它与ParNew收集器一样，在回收过程会挂起所有的用户线程，造成应用停顿。

**停顿时间越短就越适合需要与用户交互的程序**，良好的响应速度能提升用户体验。而**高吞吐量**则可以高效率地利用CPU时间，尽快完成程序的运算任务，主要适合**在后台运算而不需要太多交互的任务**。

Parallel Scavenge收集器除了会显而易见地提供可以精确控制吞吐量的参数，还提供了一个参数**-XX:+UseAdaptiveSizePolicy**，这是一个开关参数，打开参数后，就不需要手工指定新生代的大小（-Xmn）、Eden和Survivor区的比例（-XX:SurvivorRatio）、晋升老年代对象年龄（-XX:PretenureSizeThreshold）等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量，这种方式称为**GC自适应的调节策略（GC Ergonomics）**。自适应调节策略也是Parallel Scavenge收集器与ParNew收集器的一个重要区别。

另外值得注意的一点，Parallel Scavenge收集器无法与CMS收集器配合使用。

> 所谓吞吐量就是CPU用于运行用户代码的时间与CPU总消耗时间的比值。即吞吐量 = 运行用户代码时间 / (运行用户代码时间 + 垃圾收集时间)。如果虚拟机运行了100分钟，其中垃圾收集花了1分钟，那么吞吐量就是99%。



## 老年代收集器

### Serial Old收集器（单线程标记整理）

Serial Old收集器作用于老年代，采用单线程和**标记-整理算法**来实现垃圾回收。在回收垃圾的时候同样会挂起所有用户线程，造成应用的停顿。

> Serial Old收集器还有一个重要的用途是作为CMS收集器的后备方案，在并发收集发生Concurrent Mode Failure的时候使用，进行内存碎片的整理。

###  Parallel Old收集器（多线程标记整理）

Parallel Old收集器是Parallel Scavenge收集器的老年代版本，采用**多线程和标记-整理**算法来实现老年代的垃圾回收。这个收集器主要是为了配合**Parallel Scavenge收集器的使用**，即当新生代选择了Parallel Scavenge收集器的情况下，老年代可以选择Parallel Old收集器。

> 在JDK1.6以前并没有提供Parallel Scavenge收集器，所以在1.6版本以前，Parallel Scavenge收集器只能与Serial Old收集器搭配使用。

![](./Java_GC/parrel_old.jpeg)

### CMS收集器（多线程标记清除）

CMS(Concurrent Mark Sweep)收集器是一款真正实现了并发收集的老年代收集器。CMS收集器以获取**最短回收停顿时间**为目标，采用**多线程并发以及标记-清除算法**来实现垃圾回收。CMS收集器可以最大程度地减少因垃圾回收而造成应用停顿的时间。

CMS垃圾收集分为以下几个阶段：

* **初始标记**，标记GCRoots能直接关联到的对象，时间很短。

* **并发标记**，进行GCRoots Tracing（可达性分析）过程，时间很长。

* **重新标记**，修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，时间较长。

* **并发清除**，回收内存空间，时间很长。

其中，并发标记与并发清除两个阶段耗时最长，但是可以与用户线程并发执行。![](./Java_GC/cms.jpeg)

**优点**：CMS 是一款优秀的收集器，**并发收集，低停顿**

**缺点**：

1. **对CPU资源非常敏感**，可能会导致应用程序变慢，吞吐率下降。在并发阶段，因为占用了一部分线程（或者说CPU资源）而导致应用程序变慢，总吞吐量会降低。尤其是 CPU 核数低的场景中，对用户程序的影响会变得很明显。
2. **无法处理浮动垃圾**，因为在并发清理阶段用户线程还在运行，自然就会产生新的垃圾，而在此次收集中无法收集他们，只能留到下次收集，这部分垃圾为浮动垃圾，同时，由于用户线程并发执行，所以需要预留一部分老年代空间提供并发收集时程序运行使用。
3. 由于采用的标记 - 清除算法，会产生大量的**内存碎片**，不利于大对象的分配，可能会提前触发一次Full GC。

虚拟机提供了-XX:+UseCMSCompactAtFullCollection参数来进行碎片的合并整理过程，这样会使得停顿时间变长，

虚拟机还提供了一个参数配置，-XX:+CMSFullGCsBeforeCompaction，用于设置执行多少次不压缩的Full GC后，接着来一次带压缩的GC。

> 1. 通常我们把发生在新生代的垃圾回收称为Minor GC，而把发生在老年代的垃圾回收称为Major GC，而FullGC是指整个堆内存的垃圾回收，包括对新生代、老年代和持久代的回收。一般情况下应用程序发生Minor GC的次数要远远大于Major GC和Full GC的次数。 
> 2. 在讲解GC的时候会涉及到**并行**和**并发**两个概念。在这里，并行指的是多个GC收集线程之间并行进行垃圾回收。而并发指的是多个GC收集线程与所有的用户线程能够交替执行

## G1（Garbage-First）

**G1**收集器是当今收集器技术发展最前沿的成果之一，它是一款**面向服务端应用**的垃圾收集器，HotSpot开发团队赋予它的使命是（在比较长期的）未来可以替换掉JDK 1.5中发布的CMS收集器。与其他GC收集器相比，G1具备如下特点：

- **并行与并发** G1 能充分利用多CPU、多核环境下的硬件优势，使用多个CPU来缩短“Stop The World”停顿时间，部分其他收集器原本需要停顿Java线程执行的GC动作，G1收集器仍然可以通过并发的方式让Java程序继续执行。
- **分代收集** 与其他收集器一样，分代概念在G1中依然得以保留。虽然G1可以不需要其他收集器配合就能独立管理整个GC堆，但它能够采用不同方式去处理新创建的对象和已存活一段时间、熬过多次GC的旧对象来获取更好的收集效果。
- **空间整合** G1从整体来看是基于**“标记-整理”**算法实现的收集器，从局部（两个Region之间）上来看是基于**“复制”**算法实现的。这意味着G1运行期间不会产生内存空间碎片，收集后能提供规整的可用内存。此特性有利于程序长时间运行，分配大对象时不会因为无法找到连续内存空间而提前触发下一次GC。
- **可预测的停顿** 这是G1相对CMS的一大优势，降低停顿时间是G1和CMS共同的关注点，但G1除了降低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为M毫秒的时间片段内，消耗在GC上的时间不得超过N毫秒。

**横跨整个堆内存**

在G1之前的其他收集器进行收集的范围都是整个新生代或者老生代，而G1不再是这样。G1在使用时，Java堆的内存布局与其他收集器有很大区别，它**将整个Java堆划分为多个大小相等的独立区域（Region）**，虽然还保留新生代和老年代的概念，但**新生代和老年代不再是物理隔离的了，而都是一部分Region（不需要连续）的集合**。

**建立可预测的时间模型**

G1收集器之所以能建立可预测的停顿时间模型，是因为它可以**有计划地避免在整个Java堆中进行全区域的垃圾收集**。G1跟踪各个Region里面的垃圾堆积的价值大小（回收所获得的空间大小以及回收所需时间），**在后台维护一个优先列表**，每次根据允许的收集时间，**优先回收价值最大的Region（这也就是Garbage-First名称的来由）**。这种使用Region划分内存空间以及有优先级的区域回收方式，保证了G1收集器在有限的时间内可以获取尽可能高的收集效率。

**避免全堆扫描——Remembered Set**

G1把Java堆分为多个Region，就是“化整为零”。但是Region不可能是孤立的，一个对象分配在某个Region中，可以与整个Java堆任意的对象发生引用关系。在做可达性分析确定对象是否存活的时候，需要扫描整个Java堆才能保证准确性，这显然是对GC效率的极大伤害。

为了避免全堆扫描的发生，虚拟机**为G1中每个Region维护了一个与之对应的Remembered Set**。虚拟机发现程序在对Reference类型的数据进行写操作时，会产生一个Write Barrier暂时中断写操作，检查Reference引用的对象是否处于不同的Region之中（在分代的例子中就是检查是否老年代中的对象引用了新生代中的对象），如果是，便通过CardTable**把相关引用信息记录到被引用对象所属的Region的Remembered Set之中**。当进行内存回收时，在GC根节点的枚举范围中加入Remembered Set即可保证不对全堆扫描也不会有遗漏。

------

如果不计算维护Remembered Set的操作，G1收集器的运作大致可划分为以下几个步骤：

- **初始标记（Initial Marking）** 仅仅只是标记一下GC Roots 能直接关联到的对象，并且修改**TAMS（Nest Top Mark Start）**的值，让下一阶段用户程序并发运行时，能在正确可以的Region中创建对象，此阶段需要**停顿线程**，但耗时很短。
- **并发标记（Concurrent Marking）** 从GC Root 开始对堆中对象进行**可达性分析**，找到存活对象，此阶段耗时较长，但**可与用户程序并发执行**。
- **最终标记（Final Marking）** 为了修正在并发标记期间因用户程序继续运作而导致标记产生变动的那一部分标记记录，虚拟机将这段时间对象变化记录在**线程的Remembered Set Logs**里面，最终标记阶段需要**把Remembered Set Logs的数据合并到Remembered Set中**，这阶段需要**停顿线程**，但是**可并行执行**。
- **筛选回收（Live Data Counting and Evacuation）** 首先对各个Region中的回收价值和成本进行排序，根据用户所期望的GC 停顿是时间来制定回收计划。此阶段其实也可以做到与用户程序一起并发执行，但是因为只回收一部分Region，时间是用户可控制的，而且停顿用户线程将大幅度提高收集效率。

通过下图可以比较清楚地看到G1收集器的运作步骤中并发和需要停顿的阶段（Safepoint处）：

![img](https://pic.yupoo.com/crowhawk/53b7a589/0bce1667.png)

## 垃圾收集器汇总

|        收集器         | 串行、并行or并发 | 新生代/老年代 |        算法        |     目标     |                 适用场景                  |
| :-------------------: | :--------------: | :-----------: | :----------------: | :----------: | :---------------------------------------: |
|      **Serial**       |       串行       |    新生代     |      复制算法      | 响应速度优先 |          单CPU环境下的Client模式          |
|    **Serial Old**     |       串行       |    老年代     |     标记-整理      | 响应速度优先 |  单CPU环境下的Client模式、CMS的后备预案   |
|      **ParNew**       |       并行       |    新生代     |      复制算法      | 响应速度优先 |    多CPU环境时在Server模式下与CMS配合     |
| **Parallel Scavenge** |       并行       |    新生代     |      复制算法      |  吞吐量优先  |     在后台运算而不需要太多交互的任务      |
|   **Parallel Old**    |       并行       |    老年代     |     标记-整理      |  吞吐量优先  |     在后台运算而不需要太多交互的任务      |
|        **CMS**        |       并发       |    老年代     |     标记-清除      | 响应速度优先 | 集中在互联网站或B/S系统服务端上的Java应用 |
|        **G1**         |       并发       |     both      | 标记-整理+复制算法 | 响应速度优先 |        面向服务端应用，将来替换CMS        |

本文通过详细介绍HotSpot虚拟机的7种垃圾收集器回答了上一篇文章开头提出的三个问题中的第三个——“如何回收”，在下一篇文章中，我们将回答最后一个未被解答的问题——“什么时候回收”。



## JVM参数使用

### 堆内存相关

- **-Xms 与 -Xmx**

`-Xms`用于指定Java应用使用的最小堆内存，如`-Xms1024m`表示将Java应用最小堆设置为1024M。`-Xmx`用于指定Java应用使用的最大堆内存，如`-Xmx1024m`表示将Java应用最大堆设置为1024m。过小的堆内存可能会造成程序抛出OOM异常，所以正常发布的应用应该明确指定这两个参数。并且，一般会选择将`-Xms`与`-Xmx`设置成一样大小，防止JVM动态调整堆内存容量对程序造成性能影响。

- **-Xmn**

通过`-Xmn`可以设置堆内存中新生代的容量，以此达到间接控制老年代容量的作用，因为没有JVM参数可以直接控制老年代的容量。如`-Xmn256m`表示将新生代容量设置为256M。假如这个时候额外指定了`-Xms1024m -Xmx1024m`，那么老年代的容量为768M(1024-256=768)。

- **-XX:PermSize 与 -XX:MaxPermSize**

`-XX:PermSize`和`-XX:MaxPermSize`分别用于设置永久代的容量和最大容量。如`-XX:PermSize=64m -XX:MaxPermSize=128m`表示将永久代的初始容量设置为64M，最大容量设置为128M。

- **-XX:SurvivorRatio**

这个参数用于设置新生代中Eden区和Survivor(S0、S1)的容量比值。默认设置为`-XX:SurvivorRatio=8`表示Eden区与Survivor的容量比例为8:1:1。假设`-Xmn256m -XX：Survivor=8`，那么Eden区容量为204.8M(256M / 10 * 8)，S0和S1区的容量大小均为25.6M(256M / 10 * 1)。

### GC收集器相关

- **-XX:+UseSerialGC**

虚拟机运行在client模式下的默认值，使用这个参数表示虚拟机将使用Serial + Serial Old收集器组合进行垃圾回收。

> -XX:+UseSerialGC表示使用这个设置，而-XX:-UseSerialGC表示禁用这个设置。

- **-XX:+UseParNewGC**

使用这个设置以后，虚拟机将使用ParNew + Serial Old收集器组合进行垃圾回收。

- **-XX:+UseConcMarkSweepGC**

使用这个设置以后，虚拟机将使用ParNew + CMS + Serial Old的收集器组合进行垃圾回收。注意Serial Old收集器将作为CMS收集器出现Concurrent Mode Failure失败后的后备收集器来进行回收(将会整理内存碎片)。

- **-XX:+UseParallelGC**

虚拟机运行在server模式下的默认值。使用这个设置，虚拟机将使用Parallel Scavenge + Serial Old(PS MarkSweep)的收集器组合进行垃圾回收。

- **-XX:+UseParallelOldGC**

使用这个设置以后，虚拟机将使用Parallel Scavengen + Parallel Old的收集器组合进行垃圾回收。

- **-XX:PretenureSizeThreshold**

设置直接晋升到老年代的对象大小，大于这个参数的对象将直接在老年代分配，而不是在新生代分配。注意这个值只能设置为字节，如`-XX:PretenureSizeThreshold=3145728`表示超过3M的对象将直接在老年代分配。

- **-XX:MaxTenuringThreshold**

设置晋升到老年代的对象年龄。每个对象在坚持过一次Minor GC之后，年龄就会加1，当超过这个值时就进入老年代。默认设置为`-XX:MaxTenuringThreshold=15`。

- **-XX:ParellelGCThreads**

设置并行GC时进行内存回收的线程数。只有当采用的垃圾回收器是采用多线程模式，包括ParNew、Parallel Scavenge、Parallel Old、CMS，这个参数的设置才会有效。

- **-XX:CMSInitiatingOccupancyFraction**

设置CMS收集器在老年代空间被使用多少(百分比)后触发垃圾收集。默认设置`-XX:CMSInitiatingOccupancyFraction=68`表示老年代空间使用比例达到68%时触发CMS垃圾收集。仅当老年代收集器设置为CMS时候这个参数才有效。

- **-XX:+UseCMSCompactAtFullCollection**

设置CMS收集器在完成垃圾收集后是否要进行一次内存碎片整理。仅当老年代收集器设置为CMS时候这个参数才有效。

- **-XX:CMSFullGCsBeforeCompaction**

设置CMS收集器在进行多少次垃圾收集后再进行一次内存碎片整理。如设置`-XX:CMSFullGCsBeforeCompaction=2`表示CMS收集器进行了2次垃圾收集之后，进行一次内存碎片整理。仅当老年代收集器设置为CMS时候这个参数才有效。

### GC日志相关

- **-XX:+PrintGCDetails**

表示输出GC的详细情况。

- **-XX:+PrintGCDateStamps**

指定输出GC时的时间格式，比指定`-XX:+PrintGCTimeStamps`可读性更高。

- **-Xloggc**

指定gc日志的存放位置。如`-Xloggc:/var/log/myapp-gc.log`表示将gc日志保存在磁盘`/var/log/`目录，文件名为`myapp-gc.log`。

## 总结

**Minor GC**:当Eden区满时，触发Minor GC,通过复制算法迅速清理死亡对象，若S1区域满，则将对象存放到持久区。

**Major GC\Full GC**:

（1）调用System.gc时，系统建议执行Full GC，但是不必然执行

（2）老年代空间不足

（3）持久区空间不足

（4）通过Minor GC后进入老年代的平均大小大于老年代的可用内存**（上升对象过多，而老年代并没有足够空间）**

（5）由Eden区、From Space区向To Space区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小**（上升对象过大，而老年代并没有对应空间）**

**OOM的触发条件**：

* Full GC时，如果老年区空间不足，则会报OOM异常。
* gc与非gc时间耗时超过了GCTimeRatio的限制引发OOM

**降低GC的调优的策略：** 

* 通过NewRatio控制新生代老年代比例
* 通过MaxTenuringThreshold控制进入老年前生存次数等



## Reference

[深入理解JVM(3)——7种垃圾收集器](https://crowhawk.github.io/2017/08/15/jvm_3/) 
