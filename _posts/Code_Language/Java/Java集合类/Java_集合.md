---
title: Java_集合
date: 2018-03-29 15:03:04
tags: [Java]
---

数组：可以存放**基本数据类型和对象**，但是在使用上需要动态扩容。

容器  : 只能存放**对象**，也唤作集合，容器类是个大家族，常用的有ArrayList，LinkedList，HashMap等等，理解熟悉容器类才能更好的使用他们。

<!--more-->

## 层次关系

如图所示：实线是实现类，虚线是抽象类或接口

![](Java_集合/Java容器类.png)

**Collection**接口是集合类的根接口，Java中没有提供这个接口的直接实现类，而是继承该接口实现了，Set和List。**Set**中**不能包含重复**的元素，**List**是一个**有序**的集合。**可以包含重复**的元素，提供了按**索引访问**的方式。

**Map**是Java.util包的另一个接口，与Collection独立但同属于集合类。Map保存**Key-value**对。Map不能包含重复的key，但是可以包含相同的value。

**Iterator**，所有的**集合类，都实现了Iterator接口**，用于在未知容器类型的情况下遍历集合中元素的接口，主要包含以下三种方法：

*  **hasNext()**是否还有下一个元素。


*  **next()**返回下一个元素。


* **remove()**删除当前元素。

------



## 几种重要的接口和类简介

* **Collection**：接口，List，Set，Queue继承了Collection接口

* **List（有序、可重复）**
  * **ArrayList**：Object[]，默认长度10，1.5倍扩容机制。
  * **LinkedList**：使用双向链表实现List接口。
* **Set（不能重复）**
  * **HashSet（无序）**：继承HashMap实现，极大提高了访问速度。
  * **TreeSet**：使用**红黑树**（自平衡的排序二叉树）来实现存储元素
  * **LinkedHashSet**：继承自HashSet，使用了Hash来提高查询速度，用链表来保持插入顺序的一种集合。

* **Queue(队列FIFO)**
  * **LinkedList** ：同时也实现了Queue接口，因此可以当做队列使用

  * **PriorityQueue** ：优先级队列，每次弹出优先最高的元素，通过Comparator来修改优先级判定。

* **Map（键值对、键唯一、值不唯一）**
  * **HashMap** ：**数组+(链表/RBTree）**实现，默认初始数组长度16，根据2倍扩容。在链表长度大于8时，将链表转为RBTree。
  * **LinkedHashMap**：LinkedHashMap继承了 HashMap，增加了一条双向链表，用于单独记录数据插入 Map的顺序。
  * **Hashtable** ：数组+链表实现，是**HashMap的线程安全版**，它支持线程的同步，即任一时刻只有一个线程能写Hashtable，因此也导致了Hashtale在写入时会比较慢，它继承自Dictionary类，不同的是它**不允许记录的键或者值为null，同时效率较低。**
  * **ConcurrentHashMap**：通过Synchronize和CAS控制锁，并引入分段锁机制，提高并发读写能力。
  * **TreeMap**：TreeMap实现SortMap接口，能够把它保存的数据根据键排序，默认是按键值的升序排序，也可以指定排序的比较器。当用Iterator遍历TreeMap时，得到的记录是排过序的。不允许key值为空，非线程安全。 在使用TreeMap时，key必须实现Comparable接口或者在构造TreeMap传入自定义的Comparator，否则会在运行时抛出java.lang.ClassCastException类型的异常。

## 遍历

 在类集中提供了以下四种的常见输出方式：

1）Iterator：迭代输出，是使用最多的输出方式。

2）ListIterator：是Iterator的子接口，专门用于输出List中的内容。

3）foreach输出：JDK1.5之后提供的新功能，可以输出数组或集合。

4）for循环

代码示例如下：

~~~java
for（int i=0;i<arr.size();i++）{}
for（int　i：arr）{}
Iterator it = arr.iterator();
while(it.hasNext()){
    object o =it.next(); 
}
~~~

**map的遍历**

第一种：**KeySet()**

```java
Map map = new HashMap();
map.put("key1","lisi1");
map.put("key2","lisi2");
map.put("key3","lisi3");
map.put("key4","lisi4");  
//先获取map集合的所有键的set集合，keyset（）
//然而keyset()方法得到的Key为乱序
//获取set集合后调用set的迭代器
Iterator it = map.keySet().iterator();
 //获取迭代器 根据key值来获取value
while(it.hasNext()){
	Object key = it.next();
	System.out.println(map.get(key));
}
```

第二种：**entrySet()**

```java
Map map = new HashMap();
map.put("key1","lisi1");
map.put("key2","lisi2");
map.put("key3","lisi3");
map.put("key4","lisi4")
//将map集合中的映射关系取出，存入到set集合
//entrySet返回键值对的set集合 再调用集合的迭代器
Iterator it = map.entrySet().iterator();
//实际迭代器中存储的Entry接口
while(it.hasNext()){
	Entry e =(Entry) it.next();
	System.out.println("键"+e.getKey () + "的值为" + e.getValue());
}
```

推荐使用第二种方式，即entrySet()方法，效率较高。
对于keySet其实是遍历了2次，一次是转为iterator，一次就是从HashMap中取出key所对于的value。

entryset只是遍历了第一次，它把key和value都放到了entry中，因此效率更高。

两种遍历的遍历时间相差明显。

## 主要实现类区别小结  

### Vector和ArrayList

1.Vector**线程同步**，因此是**线程安全**，

​    ArrayList**线程异步**，不安全。

​    若不考虑到线程的安全因素，采用**ArrayList效率比较高**。
2.Vector增长率为目前数组长度的**100%**

   ArrayList增长率为目前数组长度的**50%**。

​    因此当在集合中使用数据量比较大的数据，用Vector更合适。
3.ArrayList 和Vector是采用数组方式存储数据，此数组元素数大于实际存储的数据以便增加和插入元素，都允许直接序号索引元素。

   Vector由于使用了synchronized方法（线程安全）所以性能上比ArrayList要差，	   	     

   LinkedList使用双向链表实现存储，按序号索引数据需要进行向前或向后遍历，但是插入数据时只需要记录本项的前后项即可，所以插入数度较快。

### HashMap与TreeMap

1. HashMap通过**hashcode**对其内容进行快速查找

​       TreeMap是有序的HashMap

2. 两个map中的元素一样，但顺序不一样，导致hashCode()不一样。

* HashMap中，同样的值的map,顺序不同，equals时，false;
* TreeMap中，同样的值的map,顺序不同, equals时，true，说明，treeMap在equals()时是整理了顺序了的。

## 集合常见问题

### Treemap和HashMap的区别

* 内部实现的数据结构不同， HashMap 是数据+（链表/红黑树）TreeMap 则是红黑树
* 数据有序性不同，迭代HashMap的输出是无序的，迭代 TreeMap 的输出是按照最右子节点的顺序开始输出
* 构造函数不同，HashMap 的链表长度大于8的时候会将链表转为红黑树。红黑树是一种二叉查找树的变形，因此需要比较节点的大小，hashmap 中是比较Hash 值的大小，在 treemap 中则需要在构造时，传入 key 的**campareto** 方法，否则将会使用 key 默认类型的 compareto 方法。

### HashMap和ConcurrentHashMap的区别

//todo

### HashMap 的 Get, Put 原理

Get方法：使用 `n-1& hashcode`来定位所处数组下标，这就是是为什么HashMap的数组长度是2的幂次的原因了，因为只有`2的幂次-1`才会在二进制上为全1，能够使用位或运算，来得到所有的数组下标。定位到数组元素后，

* 若为空，则返回 null
* 若为链表，则遍历所有元素，若存在相同的 Key 则返回元素
* 若为红黑树，则根据hashCode 来匹配子节点，若存在 key和 HashCode 相同的子节点，则返回元素。

Put方法：同 Get 方法，定位到数组元素。

* 若为空，则新建一个 Node 对象，返回。
* 若为链表，则遍历所有元素，若存在 Key 值相同，则更新该值，若不存在，则新增到链表尾部。若链表元素超过8，则将链表转为红黑树。
* 若为红黑树，则通过比较 HashCode 的方法，将元素加入到正确的子节点，若节点加入后，导致不满足红黑树定义，则通过 recolor 和 rotate 方法来重新平衡树。

## Reference

[JAVA集合类汇总](https://www.cnblogs.com/leeplogs/p/5891861.html)

<<Thinking In Java>>

[集合及concurrent并发包总结](https://my.oschina.net/zhupanxin/blog/269037)

