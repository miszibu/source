---
title: Java_异常
date: 2018-05-19 14:40:02
tags: [Java]
---

### 异常分类

按照异常需要处理的时机分为**编译时异常（强制性异常）**也叫**CheckedException**和**运行时异常（非强制性异常）**也叫**RuntimeException**。只有Java语言提供了Checked异常，Java认为Checked异常都是可以被处理的异常，所以Java程序必须显式处理异常，否则就会编译报错。这就是Java设计哲学之一，没有妥善处理异常的代码根本没有机会被执行。方法有两种

1. 当前方法知道如何处理异常，则用**try...catch**语句处理异常。
2. 当前方法不知道如何处理异常，则**向上抛出**该异常。

运行时异常，只有在当代码运行时才会发现，如除数是0，数组下标越界等，根据具体情况选择是否处理。

<!--more-->

#### Try catch finally

~~~java
public int getNum(){
    try{
        int a = 1/0;
        return 0 ;
    }catch(Exception e){
        return 1;
    }finally{
        return 2;
    }
}
~~~

 	当try语句中遇到异常时，向上抛出异常接下来的语句就不会执行。

 	在Catch中如果遇到return或异常使当前函数无法继续执行时。若存在finally，则执行finally中的语句。

#### Error和Exception的区别

​	Error类跟Exception类都是继承自**Throwable类**，他们的区别如下:

​	**Error类**一般是与**JVM虚拟机相关**的问题，如系统崩溃，虚拟机错误，内存空间不足，方法调用栈溢出等。对于这类错误导致的应用程序中断，仅靠程序本身无法恢复和预防，因此遇到这种问题，建议让程序中止。

​	**Exception类**表示**程序可以处理的异常**，可以捕获且可能恢复。遇到这类异常，应该尽可能处理，是程序运行，而不是随意终止异常。

#### 常见的RuntimeException

1. java.lang.NullPointerException 空指针异常，出现原因：调用了未被初始化的对象或者不存在的对象。
2. java.lang.ClassNotFoundException 指定的类找不到，出现原因：无法找到指定的类
3. java.lang.NumberFormatException 字符串转换为数字异常，出现原因：字符串中包含非数字字符
4. java.lang.IndexOutOfBoundsException 数组下标越界异常
5. java.lang.IllegalArgumentException 方法传递参数错误
6. java.lang.ClassCastException 数据类型转换异常
7. 还有很多 不赘述了

#### Throw 和 Throws的区别

**throw:**

1. **方法体内**，表示抛出异常，由方法体内的异常处理语句处理
2. throw是具体向外抛出异常的动作，所以它抛出的是一个异常实例，执行throw一定是抛出了某种异常。

**throws:**

1. throws语句用在**方法声明后面**，表示如果抛出异常，由调用该方法的调用者来进行异常的处理。
2. throws不一定执行，只是一种抛出异常的可能性。使得调用者知道需要捕获何种异常。

#### final，finally，finalize的区别

1. **final**:声明属性，方法，类为**常量**，分别表示属性不可变，方法不可覆盖，类不可被继承。
2. **finally**：**异常处理语句**结构的一部分，表示总是执行。
3. **finalize**：Object类的一个方法，**当GC回收一个Object类时，会调用finalize方法**，可以重写该方法提供GC时回收其他资源或者打Logger等等。例如关闭文件等。该方法只有当GC回收对象时会被被动调用，当我们主动调用时并不代表着该对象会被回收。

