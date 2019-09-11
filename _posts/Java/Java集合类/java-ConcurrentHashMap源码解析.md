---
title: java_ConcurrentHashMap源码解析
date: 2019-01-08 12:26:23
tags: [Java]
---

> 从源码角度分析ConcurrentHashMap是如何实现线程安全的

<!--more-->

## HashMap 多线程下的问题

HashMap限制了数组的Length为2的幂次，是为了实现计算Hash值所处数组位置时，能正确计算。

举例：如果设置length为5，那么实际计算的值为4，二进制为100。100与Hash值进行位|运算，结果只有0和4，不能散列到1，2，3三个数组位置上，因此length只能为2的幂次。

### resize死循环

当HashMap数组size>Length*LoadFactor时，就会执行resize方法扩容HashMap。

1. 创建一个新的，长度为原来Capacity两倍的数组，保证新的Capacity仍为2的N次方，从而保证上述寻址方式仍适用。
2. 通过如下transfer方法将原来的所有数据全部重新插入（rehash）到新的数组中。

```java
void transfer(Entry[] newTable, boolean rehash) {
  int newCapacity = newTable.length;
  for (Entry<K,V> e : table) {
    while(null != e) {
      Entry<K,V> next = e.next;
      if (rehash) {
        e.hash = null == e.key ? 0 : hash(e.key);
      }
      int i = indexFor(e.hash, newCapacity);
      e.next = newTable[i];
      newTable[i] = e;
      e = next;
    }
  }
}
```

这里如果同时有多个线程，来同时put数据，触发了resize方法，可能会导致数据链表产生死循环。



### Fast-fail机制

父类声明了**modCount**成员变量，每当HashMap数据发生变化时，就会把modCount++。

在迭代获取数据，序列化数据等时候，都会获取modCount，在结束时再次获取modCount，进行比对。如果数据发生变化则抛出`ConcurrentModificationException`。



### 解决方式

1. 使用`Collections.synchronizedMap`方法构造出一个同步Map
2. 直接使用ConcurrentHashMap替代HashMap

------



## Collection.synchronizedMap

```java
public static <K,V> Map<K,V> synchronizedMap(Map<K,V> m) {
        return new SynchronizedMap<>(m);
}
private static class SynchronizedMap<K,V>
        implements Map<K,V>, Serializable {
        private static final long serialVersionUID = 1978198479659022715L;

        private final Map<K,V> m;     // Backing Map
        final Object      mutex;        // Object on which to synchronize

        SynchronizedMap(Map<K,V> m) {
            this.m = Objects.requireNonNull(m);
            mutex = this;
        }

        SynchronizedMap(Map<K,V> m, Object mutex) {
            this.m = m;
            this.mutex = mutex;
        }

        public int size() {
            synchronized (mutex) {return m.size();}
        }
		...
}
```

SynchronizedMap内部封装了一个Map类型的成员变量，所有可能会产生并发问题的方法，都增加了同步代码块。

默认情况下，同步块对该**Map对象加锁，类对象锁**。

由此可知，这样的实现是比较粗糙的，因此不建议使用，我更推荐的是ConcurrentHashMap。

------



## ConcurrentHashMap

```java
// 内部table 增加volatile标注
transient volatile Node<K,V>[] table;

// 内部静态node类 修改val和next为volatile
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;
    	...
}
```



### 初始化

```java
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            // 如果table为null 且 sizeControl为负数(标识有线程正在进行初始化操作)
            if ((sc = sizeCtl) < 0)
                // 则暂时让出CPU时间片(yield函数不一定会让出时间片)
                Thread.yield(); // lost initialization race; just spin
            // 否则就完成初始化操作
            // 先把sizeControl值 使用CAS操作设为-1 防止并发init
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    // 为什么这里又判断一词是否为空呢 因为
                    // (sc = sizeCtl) < 0 该语句不保证原子性 
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }

	 /**
     * Table initialization and resizing control.  When negative, the
     * table is being initialized or resized: -1 for initialization,
     * else -(1 + the number of active resizing threads).  Otherwise,
     * when table is null, holds the initial table size to use upon
     * creation, or 0 for default. After initialization, holds the
     * next element count value upon which to resize the table.
     */
	// 注意sizeControl符合 volatile使用条件
	// 需要并发操作，且该值不受原参数影响 可以使用volatile
    private transient volatile int sizeCtl;
```



### put操作

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 获取第n-1个数组的node 如果为空则使用casTabAt设定值
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 这里之所以使用CAS set 是为了在合理的逻辑下提高并发效率
            // 成功设置：则返回
            // 失败设置：有其他线程设置了值，自旋重新尝试在相同位置插入
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
         // 如果插入节点的hash值为-1 则标识正在扩容 一起加入扩容操作
         else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
         else {
                V oldVal = null;
                synchronized (f) {
                    // 同步操作
                }
        ...
    }
}
    
// 获取指定数组位置的元素 确保获取主内存中最新值
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
    
// 使用CAS操作 设置值
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

