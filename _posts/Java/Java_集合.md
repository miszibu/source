---
title: Java_集合
date: 2018-03-29 15:03:04
tags: [Java]

---

## 一.容器与数组

数组：可以存放**基本数据类型和对象**，但是长度需声明，不适合动态场景

容器  : 只能存放**对象**，也唤作集合，可动态变长，JVM预设一个长度，然后如果容器类大小超额，就将其扩大两倍，内存重新寻址，做个Deep Copy。

<!--more-->

## 二.层次关系

如图所示：实线是实现类，虚线是抽象类或接口

![](../img/Java容器类.png)

**Collection**接口是集合类的根接口，Java中没有提供这个接口的直接实现类，而是继承该接口实现了，Set和List。**Set**中**不能包含重复**的元素，**List**是一个**有序**的集合。**可以包含重复**的元素，提供了按**索引访问**的方式。

**Map**是Java.util包的另一个接口，与Collection独立但同属于集合类。Map保存**Key-value**对。Map不能包含重复的key，但是可以包含相同的value。

**Iterator**，所有的**集合类，都实现了Iterator接口**，用于在未知容器类型的情况下遍历集合中元素的接口，主要包含以下三种方法：

*  **hasNext()**是否还有下一个元素。


*  **next()**返回下一个元素。


* **remove()**删除当前元素。



##三、几种重要的接口和类简介

1. **Collection**：接口，List,Set,Queue继承了Collection接口
2. **List（有序、可重复）**

* **ArrayList**：类似于动态数组，适用于大量随机访问的场景。
* **LinkedList**：使用双向链表实现List接口，插入删除操作更为方便，但相对数组实现，随机读取代价高昂。

3. **Set（不能重复）**

* **HashSet**:使用**散列函数**实现，极大提高了访问速度。
* **TreeSet**:使用**红黑树**来实现存储元素，红黑树优点在于插入后能够维持集合的有序性。
* **LinkedHashSet:**使用了Hash来提高查询速度，用链表来保持插入顺序的一种集合。

4. **Queue(队列FIFO)**
   - **LinkedList**同时也实现了Queue接口，因此可以当做队列使用
   - **PriorityQueue**:优先级队列，每次弹出优先最高的元素，通过Comparator来修改优先级判定。

   5.**Map（键值对、键唯一、值不唯一）**

* **HashMap** :查询数据优先判定数据的**HashCode**，再比较value，因此访问速度快。遍历时，取得数据的顺序是完全随机的。因为键对象不可以重复，所以HashMap最多**只允许一条记录的键为Null**。
* **LinkedHashMap**:记录存取顺序的HashMap,因此性能较HashMap差。相当于队列
* **Hashtable**:Hashtable与HashMap类似，是**HashMap的线程安全版**，它支持线程的同步，即任一时刻只有一个线程能写Hashtable，因此也导致了Hashtale在写入时会比较慢，它继承自Dictionary类，不同的是它**不允许记录的键或者值为null，同时效率较低。**
* **ConcurrentHashMap**:线程安全，并且锁分离。ConcurrentHashMap内部使用段(Segment)来表示这些不同的部分，**每个段其实就是一个小的hash table**，它们有自己的锁。只要多个修改操作发生**在不同的段上**，它们**就可以并发进行**。
* **TreeMap**:TreeMap实现SortMap接口，能够把它保存的记录根据键排序，默认是按键值的升序排序，也可以指定排序的比较器，当用Iterator遍历TreeMap时，得到的记录是排过序的。不允许key值为空，非同步的。

6. **Iterator**:用于在不清楚容器对象类型的时候进行迭代遍历

   7.**Comparable**:用于比较

## 四、遍历

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

##五、主要实现类区别小结  

Vector和ArrayList
1.Vector**线程同步**，因此是**线程安全**，

​    ArrayList**线程异步**，不安全。

​    若不考虑到线程的安全因素，采用**ArrayList效率比较高**。
2.Vector增长率为目前数组长度的**100%**

   ArrayList增长率为目前数组长度的**50%**。

​    因此当在集合中使用数据量比较大的数据，用Vector更合适。
3.ArrayList 和Vector是采用数组方式存储数据，此数组元素数大于实际存储的数据以便增加和插入元素，都允许直接序号索引元素。

   Vector由于使用了synchronized方法（线程安全）所以性能上比ArrayList要差，	   	     

   LinkedList使用双向链表实现存储，按序号索引数据需要进行向前或向后遍历，但是插入数据时只需要记录本项的前后项即可，所以插入数度较快。

HashMap与TreeMap

1. HashMap通过**hashcode**对其内容进行快速查找

​       TreeMap是有序的HashMap

2. 两个map中的元素一样，但顺序不一样，导致hashCode()不一样。

* HashMap中，同样的值的map,顺序不同，equals时，false;
* TreeMap中，同样的值的map,顺序不同, equals时，true，说明，treeMap在equals()时是整理了顺序了的。

## 六.Java容器类源码学习

将会有新的博文，深入探讨Java容器类的实现。

## 相关引用

https://www.cnblogs.com/leeplogs/p/5891861.html

https://www.cnblogs.com/ACFLOOD/p/5555555.html

http://www.cnblogs.com/CarpenterLee/p/5545987.html

<<Thinking In Java>>

[集合及concurrent并发包总结](https://my.oschina.net/zhupanxin/blog/269037)

