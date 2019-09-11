---
title: SpringIoc
date: 2018-03-23 17:20:34
tags: [Spring]
---

## Ioc

### 综述

Inverse of Control ，控制反转。是Spring容器的内核，AOP,声明式事务等功能的基础。涉及代码解耦，设计模式，代码优化等问题

有关于IOC的理解，这个回答下的第一篇文章非常清楚，示例代码配上解答令人很快理解。故不再赘述，

来源：https://www.zhihu.com/question/23277575

<!-- more -->

### Ioc的类

- 1.**构造函数注入**，将接口实现类通过构造函数变量传入。
- 2.**属性注入**，set方法注入，部分时候不需要构造函数注入时，选择set方法注入。
- 3.**接口注入**，将调用类所有依赖注入方法抽取到接口中，通过类实现接口来提供注入。(需要额外声明接口，且实际作用与属性注入无本质区别，**不建议**)

## Java反射机制相关基础知识

Java允许通过程序化的方法间接对Class进行操作。Class文件由类装载器装载后，在JVM中形成一份描述Class结构的元信息对象。通过该对象可以获知Class的结构信息，即构造函数，属性，方法等。Java允许使用该元信息对象来间接调用Class对象的功能。

### 实例

```java
public class Car {//这是普通的一个car的方法
	private String brand;
	private String color;
	private int maxSpeed;
	public Car(){System.out.println("init car!!");}//一个无参构造
	public Car(String brand,String color,int maxSpeed){//一个有参构造
		this.brand = brand;
		this.color = color;
		this.maxSpeed = maxSpeed;
	}
	public void introduce() {
       System.out.println("brand:"+brand+";color:"+color+";maxSpeed:"+maxSpeed);
	}
	public String getBrand() {
		return brand;
	}
	public void setBrand(String brand) {
		this.brand = brand;
	}
	public String getColor() {
		return color;
	}
	public void setColor(String color) {
		this.color = color;
	}
	public int getMaxSpeed() {
		return maxSpeed;
	}
	public void setMaxSpeed(int maxSpeed) {
		this.maxSpeed = maxSpeed;
	}
}
```

```java
public static Car initByDefaultConst() throws Throwable{//execption的父类
		ClassLoader loader = Thread.currentThread().getContextClassLoader();//获取根类加载器
		Class clazz = loader.loadClass("com.smart.reflect.Car");//类加载器 加载 Car类
		Constructor cons = clazz.getDeclaredConstructor((Class[])null);//获取类的默认构造器对象
		Car car = (Car)cons.newInstance();//并通过它实例化Car
		Method setBrand = clazz.getMethod("setBrand",String.class);//通过反射方法设置属性
		setBrand.invoke(car,"宝马");
		Method setColor = clazz.getMethod("setColor",String.class);
		setBrand.invoke(car,"黑色");
		return car;		
	}
	public static void main(String[] args) throws  Throwable{
		Car car = initByDefaultConst();
		car.introduce();
	}
```

可以通过编程方式调用Class的各项功能，相比于构造函数的间接调用，方法直接调用更为直接。具体查看上述注释。

### 类装载器

类装载器：就是寻找类的字节码文件并构造出类在JVM内部表示对象的组件。在Java中，类装载器通过以下步骤载入JVM。

**工作机制**:

- 1. 装载：查找和导入class文件 
- 1. 连接：执行校验，准备和解析步骤
- 校验：检查文件正确性。 准备：为类的静态变量分配存储空间  解析：将符号引用转为直接引用
- 3.初始化：对类的静态变量，静态代码块执行初始化工作。 

类装载工作有ClassLoader及其子类负责。JVM运行时产生三个ClassLoader，ClassLoader(根装载器)，ExtClassLoader(扩展类装载器)，AppClassLoader(应用类装载器)。

ClassLoader->ExtClassLoader->AppClassLoader(三者的继承关系)

**全盘负责委托机制**：除非显示调用一个ClassLoader，否则所有JVM类都由一个ClassLoader载入。而且只有当父类装载器无法找到目标类时，才会调用子类装载器从自身类路径下寻找装载目标类。

每个类在JVM中都拥有一个对应的java.lang.Class对象，提供了类结构信息的描述。所有java类包括void都有对应的Class对象。Class没有public的构造方法。Class对象实在装载类时由JVM通过调用类装载器中的defineClass()方法自动构造。

## Java反射机制

所谓的反射机制就是java语言在运行时拥有一项自观的能力。通过这种能力可以彻底的了解自身的情况为下一步的动作做准备。

下面具体介绍一下java的反射机制。 

```
Java的反射机制的实现要借助于4个类：class，Constructor，Field，Method;
```

其中class代表的类对象，Constructor－类的构造器对象，Field－类的属性对象，Method－类的方法对象。通过这四个对象我们可以粗略的看到一个类的各个组 成部分。

Java反射的作用：

在Java运行时环境中，对于任意一个类，可以知道这个类有哪些属性和方法。对于任意一个对象，可以调用它的任意一个方法。这种动态获取类的信息以及动态调用对象的方法的功能来自于Java 语言的反射（Reflection）机制。

Java 反射机制主要提供了以下功能

```
在运行时判断任意一个对象所属的类。
在运行时构造任意一个类的对象。
在运行时判断任意一个类所具有的成员变量和方法。
在运行时调用任意一个对象的方法
```

Class类

1、得到构造器的方法

```
Constructor getConstructor(Class[] params) -- 获得使用特殊的参数类型的公共构造函数， 
 
Constructor[] getConstructors() -- 获得类的所有公共构造函数 
 
Constructor getDeclaredConstructor(Class[] params) -- 获得使用特定参数类型的构造函数(与接入级别无关) 
 
Constructor[] getDeclaredConstructors() -- 获得类的所有构造函数(与接入级别无关) 
```

2、获得字段信息的方法

```
Field getField(String name) -- 获得命名的公共字段 
 
Field[] getFields() -- 获得类的所有公共字段 
 
Field getDeclaredField(String name) -- 获得类声明的命名的字段 
 
Field[] getDeclaredFields() -- 获得类声明的所有字段 
```

3、获得方法信息的方法

```
Method getMethod(String name, Class[] params) -- 使用特定的参数类型，获得命名的公共方法 
 
Method[] getMethods() -- 获得类的所有公共方法 
 
Method getDeclaredMethod(String name, Class[] params) -- 使用特写的参数类型，获得类声明的命名的方法 
 
Method[] getDeclaredMethods() -- 获得类声明的所有方法 
```

在程序开发中使用反射并结合属性文件，可以达到程序代码与配置文件相分离的目的

如果我们想要得到对象的信息，一般需要“引入需要的‘包.类’的名称——通过new实例化——取得实例化对象”这样的过程。使用反射就可以变成“实例化对象——getClass()方法——得到完整的‘包.类’名称”这样的过程。

正常方法是通过一个类创建对象，反射方法就是通过一个对象找到一个类的信息。