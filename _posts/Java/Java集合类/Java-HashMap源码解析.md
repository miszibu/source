---
title: Java_HashMap源码解析
date: 2018-12-27 09:53:43
tags: [Java]
---

> HashMap是Java.util包下极为重要的高频数据结构，本文以JDK1.8的HashMap出发，对HashMap的结构和链表延长方式等集中讲解。提高对HashMap的理解。
>
> 1. HashMap**非线程安全**，可以用**Collection的synchronized**方法来使HashMap线程安全，或者使**用ConcurrentHashMap**。
> 2. HashTable：HashTable是遗留类，实现了Map类，因此具备了许多和HashMap相同操作。但是它继承自Dictionary类，并且是线程安全的。任意时间，只允许一个写进程。**ConcurrentHashMap引入了分段锁**机制，效率更高。不建议使用HashTable，线程安全场景下使用ConcurrentHashMap，其他场景HashMap。
> 3. LinkedHashMap继承自HashMap，记录了元素插入链表的顺序。
> 4. TreeMap实现了SortedMap接口，能够把保存的数据根据键排序，默认按照键升序排。当用Iterator遍历TreeMap时，得到的记录是排过序的。如果使用排序的映射，建议使用TreeMap。在使用TreeMap时，key必须实现Comparable接口或者在构造TreeMap传入自定义的Comparator，否则会在运行时抛出java.lang.ClassCastException类型的异常。
>
>

<!--more-->

## 类签名

```java
// HashMap定义
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable 
// AbstractMap定义
public abstract class AbstractMap<K,V> implements Map<K,V>
```

注意一点，Java是**单继承(extends)多实现(implements)**，不同于C++的多重继承。

这里有个问题，AbstractMap既然已经实现了Map接口。为什么HashMap还要实现Map接口呢。

我在网络上找到的答案是：原作者认为这样做可能会有意义，但是实际上并无作用。

> I've asked Josh Bloch, and he informs me that it was a mistake. He used to think, long ago, that there was some value in it, but he since "saw the light". Clearly JDK maintainers haven't considered this to be worth backing out later.—[注1]
>
> 注：Josh Bloch The Writer Of Java Collection Framework。

------



## 类成员变量

![](HashMap.jpg)

```java
	// 序列号
    private static final long serialVersionUID = 362498820763181265L;    
    // 默认的初始容量是16 
    // 提个问题：默认16不是更直观更快吗 为什么要移位计算16，虽然很快还是多了计算 
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;   
    // 最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30; 
    // 默认的填充因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    // 当桶(bucket)上的结点数大于这个值时会转成红黑树
    static final int TREEIFY_THRESHOLD = 8; 
    // 当桶(bucket)上的结点数小于这个值时树转链表
    static final int UNTREEIFY_THRESHOLD = 6;
    // 桶中结构转化为红黑树对应的table的最小大小
    static final int MIN_TREEIFY_CAPACITY = 64;
    // 存储元素的数组，总是2的幂次倍
    transient Node<k,v>[] table; 
    // 存放具体元素的集
    transient Set<map.entry<k,v>> entrySet;
    // 存放元素的个数，注意这个不等于数组的长度。
    transient int size;
    // 每次扩容和更改map结构的计数器
    transient int modCount;   
    // 临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容
    int threshold;
    // 填充因子
    final float loadFactor;
```

* **loadFactor 加载因子**

  loadFactor加载因子控制数组存放数据的**疏密程度**，loadFactor越**趋近1**，那么在达到阈值前，Hash碰撞的概率越大，**链表越长，查找效率低**。loadFactor越**趋近0**，**空间利用率低**。默认设置0.75f是个相对平衡的点。

* **threshold 阈值 评价数组是否需要扩容的标准**

  **threshold = capacity * loadFactor**， 当size>=threshold时，就需要考虑扩增数组。

------



## Node节点类源码

```java
 static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;// Hash值，存放元素到hashMap中时，用来与其他元素hash值比较
        final K key;
        V value;
        Node<K,V> next;// 指向下一个节点

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }
		// 重写HashCode方法，异或位运算k/v的Hash值
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }
		// 隐藏真实value地址，新建对象存放value值返回
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
		// 重写equals方法
        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

Node实现了Map.Entry接口，这里注意到Node是HashMap类的静态内部类。

那什么是静态内部类呢，

> 根据Oracle官方的说法：
> Nested classes are divided into two categories: static and non-static. Nested classes that are declared static are called **static nested classes**. Non-static nested classes are called **inner classes**.
> 从字面上看，一个被称为**静态嵌套类**，一个被称为**内部类**。
> 从字面的角度解释是这样的：
>
> * 什么是嵌套？嵌套就是我跟你没关系，自己可以完全独立存在，但是我就想借你的壳用一下，来隐藏一下我自己（为了降低包的深度，方便类的使用。完全不依赖外部类，不用使用外部类的非静态属性和方法）。
> * 什么是内部？内部就是我是你的一部分，我了解你，我知道你的全部，没有你就没有我。（所以内部类对象是以外部类对象存在为前提的，内部类持有对外部类的引用）———[注2]

------



## 树节点类源码

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // 父
        TreeNode<K,V> left;    // 左
        TreeNode<K,V> right;   // 右
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;           // 判断颜色
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
        // 返回根节点
        final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;
       }
```

------

## HashMap构造器

![](HashMap构造器.png)

```java
      // 指定“容量大小”和“加载因子”的构造函数
     public HashMap(int initialCapacity, float loadFactor) {
         if (initialCapacity < 0)
             throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
         if (initialCapacity > MAXIMUM_CAPACITY)
             initialCapacity = MAXIMUM_CAPACITY;
         if (loadFactor <= 0 || Float.isNaN(loadFactor))
             throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
         this.loadFactor = loadFactor;
         this.threshold = tableSizeFor(initialCapacity);
     }

	 // 指定“容量大小”的构造函数
     public HashMap(int initialCapacity) {
         this(initialCapacity, DEFAULT_LOAD_FACTOR);
     }

	 // 默认构造函数。 默认化所有参数
     public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
     }
     
     // 包含另一个“Map”的构造函数
     public HashMap(Map<? extends K, ? extends V> m) {
         this.loadFactor = DEFAULT_LOAD_FACTOR;
         putMapEntries(m, false);//下面会分析到这个方法
     }
  	
```

实际初始化Map

```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        // 判断table是否已经初始化
        if (table == null) { // pre-size
            // 未初始化，s为m的实际元素个数
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                    (int)ft : MAXIMUM_CAPACITY);
            // 计算得到的t大于阈值，则初始化阈值
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        // 已初始化，并且m元素个数大于阈值，进行扩容处理
        else if (s > threshold)
            resize();
        // 将m中的所有元素添加至HashMap中
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

根据给定的容量，获取比该容量大的二的幂次作为实际容量

```java
	/**
     * 根据给定的容量返回大于等于cap容量的2的幂次 作为真实容量
     * 如果给定的cap就是2的幂次 就返回该值
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

假设有三种情况，这里的**位运算使用太巧妙**了，令人不禁击节赞叹。

1. 传入容量为0或负数，那么减1计算后就是负数，进行无符号右移位或计算后。得到的结果肯定是负数，所以最终return值为1。也就是2的零次方。
2. 传入容量为2的幂次，那么在减1计算后，二进制位显示最后n位为1。那么右移位运算的结果就是原来的值，最终返回值就是传入容量
3. 传入其他正数，那么肯定有一个最高位为1，那么对这个位右移1，2，4，8，16。合计31次移位，结果是最高位后的所有位都变成1。再来一个加一计算，就可以得到比传入值大，最接近的2的幂次值。而1-2-4-8-16的移位，在传入容量为Integer，去符号位，实际数字位为31位。肯定满足，可以让最高位与所有后续位进行与运算。

这里是一张例子图，可以帮助理解。

![](tableSizeFor)



------

## Put方法

HashMap提供了三种方法来添加元素，但另外两方法只是调用了putVal()方法。

![](putVal方法.png)

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // table未初始化或者长度为0，进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // (n - 1) & hash 确定元素存放在哪个桶中，桶为空，新生成结点放入桶中(此时，这个结点是放在数组中)
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 桶中已经存在元素
    else {
        Node<K,V> e; K k;
        // 比较桶中第一个元素(数组中的结点)的hash值相等，key相等
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
                // 将第一个元素赋值给e，用e来记录
                e = p;
        // hash值不相等，即key不相等；为红黑树结点
        else if (p instanceof TreeNode)
            // 放入树中
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 为链表结点
        else {
            // 在链表最末插入结点
            for (int binCount = 0; ; ++binCount) {
                // 到达链表的尾部
                if ((e = p.next) == null) {
                    // 在尾部插入新结点
                    p.next = newNode(hash, key, value, null);
                    // 结点数量达到阈值，转化为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    // 跳出循环
                    break;
                }
                // 判断链表中结点的key值与插入的元素的key值是否相等
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 相等，跳出循环
                    break;
                // 用于遍历桶中的链表，与前面的e = p.next组合，可以遍历链表
                p = e;
            }
        }
        // 表示在桶中找到key值、hash值与插入元素相等的结点
        if (e != null) { 
            // 记录e的value
            V oldValue = e.value;
            // onlyIfAbsent为false或者旧值为null
            if (!onlyIfAbsent || oldValue == null)
                //用新值替换旧值
                e.value = value;
            // 访问后回调
            afterNodeAccess(e);
            // 返回旧值
            return oldValue;
        }
    }
    // 结构性修改
    ++modCount;
    // 实际大小大于阈值则扩容
    if (++size > threshold)
        resize();
    // 插入后回调
    afterNodeInsertion(evict);
    return null;
} 
```

------

## Get方法

同put方法，先写一个完整参数签名版本既getNode()方法，其他方法调用该方法。

```java
// 查到返回Node，查不到返回null
final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    	// 当table不为空 table.length>0 且对应数组位置不为空时 进入get
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
           	// 永远首先查询头节点
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            // 是否具有后续节点 根据节点类型 去RBtree中查 或者 链表中查询
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

------

### 扩容方法

用于初始化 或 扩容数组。扩容数组要将元数据 重新Hash到新的位置上。新位置要么不变，要么原先位置*2。

这里有三个概念要区分一下

* **size** ：table中实际存放有存放数据的bin数量
* **table.length**：table数组的实际长度
* **threshold** ：capacity * loadFactor ：阈值跟size进行比对，判断是否需要扩容

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
    	// 记录扩容前参数 初始化新参数
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // HashMap存在最大值 若超过2的30次则不再扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 否则就扩容两倍 因为HashMap的capacity要为2的幂次
            // 如果新capacity 大于2的30次 暂时不赋值新的threshold 下面重新赋值
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; 
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               
            // 若table为空，则初始化参数
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
    	// 从新计算resize上限
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
    	// 堆内存开辟新容量数组空间
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            // 遍历Table的所有bin
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    // 若为单节点 直接reHash到新table
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    // treeNode
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { 
                        // 将链表的数据 重新散列到原位置和新位置上
                        // loHead 为原位置链表
                        // hiHead 为新位置链表
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 由于capacity是2的幂次直接拿来与运算 
                            // 所以capacity最高为1 其他位置为0 
                            // 与运算 0与任何都是0 所以只计算最高位 rehash后的位置
                            // 非常精妙的划分
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
```

------

### Hash方法

```java
 static final int hash(Object key) {
        int h;
        // >>> 无符号右移16位 高位0填充
     	// 高位16位不变 低位与高位进行与运算 一次扰动
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

**计算key位置**

```java
// n为HashMap capacity  所以是2的幂次
// 所以位运算结果 肯定落于0-(capacity-1)中，不会超限
(n-1)&hash 

// 判断 resize后 key是否需要double位置移位
// 原理同上 结果只有0 和 非0 
n&hash == 0
```

![](HashMapHashCode.png)





------



## 相关资料

[注1：Why does LinkedHashSet extend HashSet and implement Set](https://stackoverflow.com/questions/2165204/why-does-linkedhashsete-extend-hashsete-and-implement-sete)

[注2：为什么Java内部类要设计成静态和非静态两种？](https://www.zhihu.com/question/28197253/answer/39814613)

[Java 8系列之重新认识HashMap](https://zhuanlan.zhihu.com/p/21673805)

[JavaGuide HashMap源码分析](https://github.com/Snailclimb/JavaGuide/blob/master/Java%E7%9B%B8%E5%85%B3/%E8%BF%99%E5%87%A0%E9%81%93Java%E9%9B%86%E5%90%88%E6%A1%86%E6%9E%B6%E9%9D%A2%E8%AF%95%E9%A2%98%E5%87%A0%E4%B9%8E%E5%BF%85%E9%97%AE.md)

[HashMap源码注解 之 静态工具方法hash()、tableSizeFor()](https://blog.csdn.net/fan2012huan/article/details/51097331)