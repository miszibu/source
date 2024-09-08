---
layout: java
title: Java_反射
date: 2018-12-07 10:41:11
tags: Java
---

> 反射(Reflecting)是Java的特征之一，它允许Java程序在运行期间，操作类或对象的内部属性。
>
> Oracle官方对反射的解释是：
>
> Reflection enables Java code to discover information about the fields, methods and constructors of loaded classes, and to use reflected fields, methods, and constructors to operate on their underlying counterparts, within security restrictions.
> The API accommodates applications that need access to either the public members of a target object (based on its runtime class) or the members declared by a given class. It also allows programs to suppress default reflective access control.
>
> 反射使Java代码具有在安全限制下发现装载类字段，方法和构造器并且使用反射字段，方法和构造来操作类的的底层对象。
>
> API容纳需要访问目标对象的公共成员（基于其运行时类）或给定类声明的成员的应用程序。它还允许程序抑制默认的反射访问控制。

<!--more-->

## 反射的用途

反射使得Java程序能在**运行时**，动态获取创建参数并获取其属性。

Java 反射主要提供以下功能：

- 在运行时判断任意一个对象所属的类；
- 在运行时构造任意一个类的对象；
- 在运行时判断任意一个类所具有的成员变量和方法（通过反射甚至可以调用private方法）；
- 在运行时调用任意一个对象的方法

**具体应用**：在Spring系列框架中，大量使用了反射，比如判断应用参数动态加载类的内容。比如类的序列化中，获取对象的readObject\writeObject方法等等。

## 反射的具体运用

Java反射机制主要通过**class，Constructor，Field，Method**四个类来实现。

反射的相关内容基本都在**java.util.relfect**包下。

### 获得Class对象

```java
    Class<?> a = new Person().getClass();
    Class<?> b = Person.class;
    Class<?> c = Class.forName("bean.Person");
    System.out.println(a.equals(b) && a.equals(c));
	//True 获取的类Class对象 都为同一个实例
```

### 判断是否为某个类的实例

```java
System.out.println(new Person() instanceof Father);
System.out.println(Father.class.isInstance(new Person()));
// true true
```

* 使用`instanceof`关键字来判断是否为某个类的实例
* 也可使用反射中Class对象的`isInstance()`方法来判断是否为某个类的实例（native方法）

### 创建实例

```java
// 获取类class对象 调用newInstance()方法 生成实例对象
// 调用无参构造器
Person reflectPerson0 = Person.class.newInstance();
Person reflectPerson1 = new Person().getClass().newInstance();
Person reflectPerson2 = (Person) Class.forName("bean.Person").newInstance();

// 获取构造器 获取类class对象 调用getConstructor方法获取类构造器
// 调用构造器的newInstance() 方法调用有参构造器
Constructor<Person> constructor = Person.class.getConstructor(String.class,Integer.class,Gender.class);
Person reflectPerson3 = constructor.newInstance("zibu",new Integer(11),Gender.MALE);
```

### 获取方法

获取某个class对象的方法集合，有以下三种方法

`getDeclardMethods`可以**获取私有方法**，但**不能访问继承方法**

```java
//getDeclaredMethods 方法返回类或接口声明的所有方法，包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法。
public Method[] getDeclaredMethods() throws SecurityException


//getMethods 方法返回某个类的所有公用（public）方法，包括其继承类的公用方法。
public Method[] getMethods() throws SecurityException

//getMethod 方法返回一个特定的方法，其中第一个参数为方法名称，后面的参数为方法的参数对应Class的对象。(只能获取类的公有方法)
public Method getMethod(String name, Class<?>... parameterTypes)

```

```java
Method[] methods = Person.class.getMethods();
Method[] declaredMethods = Person.class.getDeclaredMethods();
Method toStringMethod = Person.class.getMethod("toString");
//getMethods()方法获取的所有方法
System.out.println("getMethods获取的方法：");
for(Method m:methods){
    System.out.println(m);
}
//getDeclaredMethods()方法获取的所有方法
System.out.println("getDeclaredMethods获取的方法：");
for(Method m:declaredMethods){
    System.out.println(m);
}
//getMethod()方法获取的指定方法
System.out.println(toStringMethod);

getMethods获取的方法：
public java.lang.String bean.Person.toString()
public final void java.lang.Object.wait(long,int) throws java.lang.InterruptedException
public final native void java.lang.Object.wait(long) throws java.lang.InterruptedException
public final void java.lang.Object.wait() throws java.lang.InterruptedException
public boolean java.lang.Object.equals(java.lang.Object)
public native int java.lang.Object.hashCode()
public final native java.lang.Class java.lang.Object.getClass()
public final native void java.lang.Object.notify()
public final native void java.lang.Object.notifyAll()
getDeclaredMethods获取的方法：
public java.lang.String bean.Person.toString()
private void bean.Person.readObject(java.io.ObjectInputStream) throws java.io.IOException,java.lang.ClassNotFoundException
private void bean.Person.writeObject(java.io.ObjectOutputStream) throws java.io.IOException
public java.lang.String bean.Person.toString()

```

### 获取类的成员变量(字段)信息

* `getField`:访问公有(public)的成员变量
* `getDeclaredField`:所有已声明的成员变量，但不能得到父类的成员变量

使用方法同method, 不再赘述。

### 调用方法

当我们通过反射获取到类的method时，可以通过`invoke()`方法来调用这个方法。

```java
public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
```

```java
// 获取Person类中名为test 且参数签名为String的方法
Method method = Person.class.getMethod("test",String.class);
// 调用该方法，首先传入类对象实例，其次传入所需参数
method.invoke(Person.class.newInstance(),"111111");

//Person中的test方法 
 public void test(String temp){
        System.out.println(temp);
}

//输出：111111
```

### 利用反射创建数组

```java
Class<?> cls = Class.forName("java.lang.String");
Object array = Array.newInstance(cls,25);
//往数组里添加内容
Array.set(array,0,"hello");
Array.set(array,1,"Java");
Array.set(array,2,"fuck");
Array.set(array,3,"Scala");
Array.set(array,4,"Clojure");
//获取某一项的内容
System.out.println(Array.get(array,3));

//输出：Scala
```

`Array`是java.lang.reflect包下的final对象。

```java
// 调用newInstance创建数组实例
public static Object newInstance(Class<?> componentType, int length)
        throws NegativeArraySizeException {
        return newArray(componentType, length);
}
// newArrays是native方法 
private static native Object newArray(Class<?> componentType, int length)
        throws NegativeArraySizeException;
```

源码目录：`openjdk\hotspot\src\share\vm\runtime\reflection.cpp`

Array 类的 `set` 和 `get` 方法都为 native 方法，在 HotSpot JVM 里分别对应 `Reflection::array_set` 和 `Reflection::array_get` 方法。

```c++
arrayOop Reflection::reflect_new_array(oop element_mirror, jint length, TRAPS) {
  if (element_mirror == NULL) {
    THROW_0(vmSymbols::java_lang_NullPointerException());
  }
  if (length < 0) {
    THROW_0(vmSymbols::java_lang_NegativeArraySizeException());
  }
  if (java_lang_Class::is_primitive(element_mirror)) {
    Klass* tak = basic_type_mirror_to_arrayklass(element_mirror, CHECK_NULL);
    return TypeArrayKlass::cast(tak)->allocate(length, THREAD);
  } else {
    Klass* k = java_lang_Class::as_Klass(element_mirror);
    if (k->oop_is_array() && ArrayKlass::cast(k)->dimension() >= MAX_DIM) {
      THROW_0(vmSymbols::java_lang_IllegalArgumentException());
    }
    return oopFactory::new_objArray(k, length, THREAD);
  }
}
```

## 总结

本文主要讲述了反射的基础内容，围绕着**class，Constructor，Field，Method**四个类讲述了反射的使用。关于反射更为具体的实现，原理等内容等待下次完善。