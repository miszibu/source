---
title: Java_设计模式
date: 2018-04-12 11:41:38
tags: [Java,设计模式]
---

## Java设计模式简介

本文介绍了常用的工厂模式，抽象工厂模式，单例模式（四种实现），代理模式等常用设计模式，

使用设计模式是为了重用代码、让代码更容易被他人理解、保证代码可靠性。 毫无疑问，设计模式于己于他人于系统都是多赢的，设计模式使代码编制真正工程化，设计模式是软件工程的基石。

总共23种设计模式，这里只介绍了常用的几种，其他的需要时再往文中补充。

<!--more-->

## 创建型模式:对象怎么来

​	这些设计模式提供了一种创建对象的同时，隐藏创建逻辑的方式，而不是使用new运算符直接实例化对象。这使得程序在判断针对某个给定实例需要创建哪些对象时更为灵活。

#### 工厂模式（Factory Pattern）

​	创建一个工厂类，创建对象有实例化工厂类，然后给与变量名来实现。这样就实现了**对客户端隐藏逻辑**的作用，但是却会产生一个额外的工厂类,会**增加**系统的**复杂度**。

~~~java
public class ShapeFactory {
   //使用 getShape 方法获取形状类型的对象
   public Shape getShape(String shapeType){
      if(shapeType == null){
         return null;
      }        
      if(shapeType.equalsIgnoreCase("CIRCLE")){
         return new Circle();
      } else if(shapeType.equalsIgnoreCase("RECTANGLE")){
         return new Rectangle();
      } else if(shapeType.equalsIgnoreCase("SQUARE")){
         return new Square();
      }
      return null;
   }
}
~~~

#### 抽象工厂模式（Abstract Factory Pattern）

抽象工厂模式也就是不仅生产鼠标，同时生产键盘。  
也就是PC厂商是个父类，有生产鼠标，生产键盘两个接口。  
戴尔工厂，惠普工厂继承它，可以分别生产戴尔鼠标+戴尔键盘，和惠普鼠标+惠普键盘。  
创建工厂时，由戴尔工厂创建。  
后续工厂.生产鼠标()则生产戴尔鼠标，工厂.生产键盘()则生产戴尔键盘。 

> **在抽象工厂模式中，假设我们需要增加一个工厂**

假设我们增加华硕工厂，则我们需要增加华硕工厂，和戴尔工厂一样，继承PC厂商。  
之后创建华硕鼠标，继承鼠标类。创建华硕键盘，继承键盘类。  
即可。

> **在抽象工厂模式中，假设我们需要增加一个产品**

假设我们增加耳麦这个产品，则首先我们需要增加耳麦这个父类，再加上戴尔耳麦，惠普耳麦这两个子类。  
之后在PC厂商这个父类中，增加生产耳麦的接口。最后在戴尔工厂，惠普工厂这两个类中，分别实现生产戴尔耳麦，惠普耳麦的功能。    

#### 单例模式（Singleton Pattern）

#####1、懒汉式，线程不安全

**描述：**这种方式是最基本的实现方式，这种实现最大的问题就是不支持多线程。因为没有加锁 synchronized，所以严格意义上它并不算单例模式。

**代码实例：**

```javascript
public class Singleton {  
    private static Singleton instance;  
    private Singleton (){}  //默认实现类设为私有 使其无法被外部访问
  
    public static Singleton getInstance() {  
    if (instance == null) {  //当两个线程同时进入时，线程不安全
        instance = new Singleton();  
    }  
    return instance;  
    }  
}  
```

#####2、懒汉式，线程安全

**描述：**这种方式具备很好的 lazy loading，能够在多线程中很好的工作，但是，效率很低，99% 情况下不需要同步。
优点：第一次调用才初始化，避免内存浪费。
缺点：必须加锁 synchronized 才能保证单例，但**加锁**会**影响效率**。

**代码实例：**

```java
public class Singleton {  
    private static Singleton instance;  
    private Singleton (){}  
    public static synchronized Singleton getInstance() {  //通过加同步锁来实现线程安全
    if (instance == null) {  
        instance = new Singleton();  
    }  
    return instance;  
    }  
} 
```

#####3、饿汉式，通过静态加载实现线程安全

**描述：**这种方式比较常用，但容易产生垃圾对象。
优点：没有加锁，执行效率会提高。
缺点：类加载时就初始化，**浪费内存**。
它基于 classloder 机制避免了多线程的同步问题（优先加载静态变量，静态方法），不过，instance 在类装载时就实例化，虽然导致类装载的原因有很多种，在单例模式中大多数都是调用 getInstance 方法， 但是也不能确定有其他的方式（或者其他的静态方法）导致类装载，这时候初始化 instance 显然没有达到 lazy loading 的效果。

**代码实例：**

```java
public class Singleton {  
    private static Singleton instance = new Singleton();//静态类变量，当类被加载时，优先加载静态变量和静态方法  
    private Singleton (){}  
    public static Singleton getInstance() {  
    return instance;  
    }  
}  
```

#####4、登记式/静态内部类，线程安全

**描述：**这种方式同样利用了 classloder 机制来保证初始化 instance 时只有一个线程，它跟第 3 种方式不同的是：第 3 种方式只要 Singleton 类被装载了，那么 instance 就会被实例化（没有达到 lazy loading 效果），而这种方式是 Singleton 类被装载了，instance 不一定被初始化。因为 SingletonHolder 类没有被主动使用，只有通过显式调用 getInstance 方法时，才会显式装载 SingletonHolder 类，从而实例化 instance。想象一下，如果实例化 instance 很消耗资源，所以想让它延迟加载，另外一方面，又不希望在 Singleton 类加载时就实例化，因为不能确保 Singleton 类还可能在其他的地方被主动使用从而被加载，那么这个时候实例化 instance 显然是不合适的。这个时候，这种方式相比第 3 种方式就显得很合理。

**代码实例：**

```java
public class Singleton {  
    private static class SingletonHolder {  //设置静态内部类 这样就解决了内存浪费的问题
    private static final Singleton INSTANCE = new Singleton();  
    }  
    private Singleton (){}  
    public static final Singleton getInstance() {  
    return SingletonHolder.INSTANCE;  
    }  
}   
```

####代理模式（Proxy Pattern）

所谓代理模式是指客户端并**不直接调用实际的对象**，而是通过调用代理，来**间接的调用**实际的对象。
为什么要采用这种间接的形式来调用对象呢？一般是因为客户端不想直接访问实际的对象，或者访问实际的对象存在困难，因此通过一个代理对象来完成间接的访问。
在现实生活中，这种情形非常的常见，比如火车票代售点

![代理模式](../img/代理模式.jpg)

Subject接口的实现

```java
public interface Subject {
    void visit();
}
```

实现了Subject接口的两个类：

```java
public class RealSubject implements Subject {

    private String name = "byhieg";
    @Override
    public void visit() {
        System.out.println(name);
    }
}
public class ProxySubject implements Subject{

    private Subject subject;

    public ProxySubject(Subject subject) {
        this.subject = subject;
    }

    @Override
    public void visit() {
        subject.visit();
    }
}
//每一个代理类都需要实现一遍委托类，如果接口增加方法，则代理类也要跟着修改。
```

## 相关引用

[菜鸟驿站 JAVA设计模式](http://www.runoob.com/design-pattern/design-pattern-intro.html)

[工厂模式与抽象工厂模式有什么区别](https://www.zhihu.com/question/20367734)

[[Java设计模式之代理模式](http://www.cnblogs.com/qifengshi/p/6566752.html)](https://www.cnblogs.com/qifengshi/p/6566752.html)