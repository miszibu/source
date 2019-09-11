---
title: Java_List源码解析
date: 2018-12-25 09:54:14
tags: [Java]

---



> 容器类中List是使用最为广泛的，主要有ArrayList，LinkedList两个实现类。本文将从源码入手，直接分析ArrayList，LinkedList的实现，扩容机制等。
>
> 了解所使用的类库，才能更好的使用它。——《Effective Java》第47条。

<!--more-->

------



## ArrayList

### ArrayList简介

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

ArrayList是由数组实现，并提供了**动态扩容**机制。无需我们去关注扩容数组的问题。

它继承于AbstarctList，实现了List，RandomAccess，Cloneable，Serializable四个接口。

由于数组的实现，ArrayList的增加元素和查询第n个元素都是O(1)，然而删除和新增都是O(N)

* **RandomAccess**，一个空的接口（标记型接口），用于标注一个类是否可以实现随机访问。实现该接口的类。
* **Cloneable**，标记型接口，用于声明接口覆盖了clone()方法，能被克隆。
* **java.io.Serializable**，ArrayList支持序列化，能通过序列化进行传输。

ArrayList不是线程安全的，在多线程场景下，可以使用**Vector**和**CopyOnWriteArrayList**替代。

> Tips: 在传入大量数据前，使用ensureCapacity()修改List长度，来避免接下来频繁的扩容操作。

### ArrayList源码分析

#### 类成员变量

```java
	/**
     * 默认初始容量大小
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 空数组（用于空实例）。
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

     //用于默认大小空实例的共享空数组实例。
      //我们把它从EMPTY_ELEMENTDATA数组中区分出来，以知道在添加第一个元素时容量需要增加多少。
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 保存ArrayList数据的数组
     * 非私有变量，默认protected，方便相关的类进行访问
     * 使用transient标注 不使用自动序列化机制 
     * 因为ArrayList是动态扩展的，实际容量大于成员变量个数。若默认使用Serializable接口来序列化，会反序列化出大量的null,影响效率。
     */
    transient Object[] elementData;

    /**
     * ArrayList 所包含的元素个数
     */
    private int size;
```

### 构造函数

```java
	/**
     * 带初始容量参数的构造函数。
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            // 减少一次堆内存分配 对象引用指向全局空Object[]
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    /**
     * 设置默认函数为空Object[] 
     * 默认初始大小为10 这里却指向空数组，是为了延迟初始化，在真实插入数据时，再分配
     * 延迟初始化
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * 构造一个包含指定集合的元素的列表，按照它们由集合的迭代器返回的顺序。
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        //如果指定集合元素个数不为0
        if ((size = elementData.length) != 0) {
            // c.toArray 可能返回的不是Object类型的数组所以加上下面的语句用于判断，
            //这里用到了反射里面的getClass()方法
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // 用空数组代替
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

```

### 容量相关—动态扩容

```java
	/**
     * trim List Size to really size 
     */
    public void trimToSize() {
        // AbstractList成员变量 记录List size修改的次数
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }

	//下面是ArrayList的扩容机制 设计合理的扩容机制来避免
    /**
     * 如有必要，增加此ArrayList实例的容量，以确保它至少能容纳元素的数量
     * @param   minCapacity   所需的最小容量
     */
    public void ensureCapacity(int minCapacity) {
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            // any size if not default element table
            ? 0
            // larger than default for default empty table. It's already
            // supposed to be at default size.
            : DEFAULT_CAPACITY;

        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
    }
    //得到最小扩容量
    private void ensureCapacityInternal(int minCapacity) {
		ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
	private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
	//判断是否需要扩容
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
        if (minCapacity - elementData.length > 0)
            //调用grow方法进行扩容，调用此方法代表已经开始扩容了
            grow(minCapacity);
    }

    /**
     * 数组分配的最大size
     * -8是因为，部分虚拟机会在数组前插入数据，因此预留部分
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * ArrayList扩容的核心方法。
     */
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
   		// 计算新容量为旧容量的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 检查新容量是否大于最小需要容量
        // 若扩容后仍小于最小需要容量，那么就把最小需要容量当作数组的新容量，
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //再检查新容量是否超出了ArrayList所定义的最大容量， 
        //由于minCapacity是Integer类型 所以最大值为Integer.MAX_VALUE
        //如果为负值，则超出最大值，则报OOM异常
        //否则就与MAX_ARRAY_SIZE比值，设置为较大值
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    //比较minCapacity和 MAX_ARRAY_SIZE
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

### 查找元素

```java
    /**
     *返回此列表中指定元素的首次出现的索引，如果此列表不包含此元素，则为-1 
     */
    public int indexOf(Object o) {
        // 判断是否为null, 采用不同方法比对 
        // 避免npe问题
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                //equals()方法比较
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

```

### 添加元素

```java
	/**
     * 用指定的元素替换此列表中指定位置的元素。 
     */
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        //返回原来在这个位置的元素
        return oldValue;
    }

    /**
     * 将指定的元素追加到此列表的末尾。 
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

    /**
     * 在此列表中的指定位置插入指定的元素。 
     *先调用 rangeCheckForAdd 对index进行界限检查；然后调用 ensureCapacityInternal 方法保证capacity足够大；
     *再将从index开始之后的所有成员后移一个位置；将element插入index位置；最后size加1。
     */
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //arraycopy() native方法
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```

### 删除元素

```java
	/**
     * 删除该列表中指定位置的元素。 将任何后续元素移动到左侧（从其索引中减去一个元素）。 
     */
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
      	//从列表中删除的元素 
        return oldValue;
    }

```

### 其他

```java
	/**
     * 返回此ArrayList实例的浅拷贝。 （元素本身不被复制。） 
     */
    public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            //Arrays.copyOf功能是实现数组的复制，返回复制后的数组。参数是被复制的数组和复制的长度
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // 这不应该发生，因为我们是可以克隆的
            throw new InternalError(e);
        }
    }

    /**
     *以正确的顺序（从第一个到最后一个元素）返回一个包含此列表中所有元素的数组。 
     *返回的数组将是“安全的”，因为该列表不保留对它的引用。 （换句话说，这个方法必须分配一个新的数组）。
     *因此，调用者可以自由地修改返回的数组。 此方法充当基于阵列和基于集合的API之间的桥梁。
     */
    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }

    /**
     * 以正确的顺序返回一个包含此列表中所有元素的数组（从第一个到最后一个元素）; 
     *返回的数组的运行时类型是指定数组的运行时类型。 如果列表适合指定的数组，则返回其中。 
     *否则，将为指定数组的运行时类型和此列表的大小分配一个新数组。 
     *如果列表适用于指定的数组，其余空间（即数组的列表数量多于此元素），则紧跟在集合结束后的数组中的元素设置为null 。
     *（这仅在调用者知道列表不包含任何空元素的情况下才能确定列表的长度。） 
     */
    @SuppressWarnings("unchecked")
    public <T> T[] toArray(T[] a) {
        if (a.length < size)
            // 新建一个运行时类型的数组，但是ArrayList数组的内容
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
            //调用System提供的arraycopy()方法实现数组之间的复制
        System.arraycopy(elementData, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }
```

### ModCount

```java
	/**
     * 自定义序列化存储方式
     * 根据size，只序列化非null参数 提高效率
     *
     */
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // 记录读前所modCount
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }
		
        // 比对modCount,若变化则标识出现并发问题
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

```

modCount用来**标记数组结构的改变**。每一次数组发生结构改变（比如说增与删）这个变量都会自增。当 List 进行遍历的时候，遍历的前后会检查这个遍历是否被改变，如果有改变则将抛出异常。
这种设计体现了 fail-fast 思想，这是一种编程哲学。尽可能的抛出异常而不是武断的处理一个可能会造成问题的异常。这种思想在很多应用场景的开发（尤其是多线程高并发）都有着普遍的应用。

### 内部类



```java
    (1)private class Itr implements Iterator<E>  
    (2)private class ListItr extends Itr implements ListIterator<E>  
    (3)private class SubList extends AbstractList<E> implements RandomAccess  
    (4)static final class ArrayListSpliterator<E> implements Spliterator<E>  
```

　　ArrayList有四个内部类，其中的**Itr是实现了Iterator接口**，同时重写了里面的**hasNext()**，**next()**，**remove()等方法；其中的ListItr**继承**Itr**，实现了**ListIterator接口**，同时重写了**hasPrevious()**，**nextIndex()**，**previousIndex()**，**previous()**，**set(E e)**，**add(E e)**等方法，所以这也可以看出了**Iterator和ListIterator的区别:**ListIterator在Iterator的基础上增加了添加对象，修改对象，逆向遍历等方法，这些是Iterator不能实现的。



## LinkedList

### LinkedList分析

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

实现了List和Deque接口，同时具备了列表和队列的特征。

由于底层是**双向链表**，因此具备了高效的插入和删除的特性。LinkedList**不是线程安全**，可以调用静态类Collections类中的synchronizedList方法，实现线程安全。

```java 
// Collections内部类SynchronizedList 通过Synchronized加锁
List list=Collections.synchronizedList(new LinkedList(...));
```

### 成员变量

```java
	// 
	transient int size = 0;
	// 头结点 尾节点
    transient Node<E> first;
    transient Node<E> last;
	
	// 内部静态类
	private static class Node<E> {
        E item; // 节点值
        Node<E> next; 
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

### 构造函数

```java
	// 调用空构造方法 添加集合成员
	public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
	public LinkedList() {
    }
```

### 添加元素

```java
	// 将元素插入链表尾	
	public boolean add(E e) {
        linkLast(e);
        return true;
    }
	/**
     * 链接使e作为最后一个元素。
     */
    void linkLast(E e) {
        // 记录原末尾元素
        final Node<E> l = last;
        // 新建节点
        final Node<E> newNode = new Node<>(l, e, null);
        // 移动last指针
        last = newNode;
        // 若空链表 则将头指向该节点
        if (l == null)
            first = newNode;
        else
            // 否则将尾节点延长
            l.next = newNode;//指向后继元素也就是指向下一个元素
        size++;
        modCount++;
    }

	// 指定位置插入元素
	public void add(int index, E element) {
        checkPositionIndex(index); //检查索引是否处于[0-size]之间

        if (index == size)//添加在链表尾部
            linkLast(element);
        else//添加在链表中间
            linkBefore(element, node(index));
    }
	void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }

	public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);
    }
	// 指定位置插入集合
	public boolean addAll(int index, Collection<? extends E> c) {
        //1:检查index范围是否在size之内
        checkPositionIndex(index);

        //2:toArray()方法把集合的数据存到对象数组中
        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false;

        //3：得到插入位置的前驱节点和后继节点
        Node<E> pred, succ;
        //如果插入位置为尾部，前驱节点为last，后继节点为null
        if (index == size) {
            succ = null;
            pred = last;
        }
        //否则，调用node()方法得到后继节点，再得到前驱节点
        else {
            succ = node(index);
            pred = succ.prev;
        }

        // 4：遍历数据将数据插入
        for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            //创建新节点
            Node<E> newNode = new Node<>(pred, e, null);
            //如果插入位置在链表头部
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }

        //如果插入位置在尾部，重置last节点
        if (succ == null) {
            last = pred;
        }
        //否则，将插入的链表与先前链表连接起来
        else {
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }    
```

### 根据位置获取元素

```java
	public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }

	// 获取头结点的方法 区别在于头结点为null时，抛出异常还是null
	public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }
	public E element() {
        return getFirst();
    }
	public E peek() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }

	public E peekFirst() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
     }
```

### 根据对象得到索引的方法

```java
public int indexOf(Object o) {
        int index = 0;
    	// 判断是否是null 来避免npe问题
        if (o == null) {
            //从头遍历
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            //从头遍历
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
    }
```

### Node方法

```java
// 获取指定序号的Node
Node<E> node(int index) {
   	// 前一半 正序查 后一半 倒序查
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```