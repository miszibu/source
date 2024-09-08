---
title: Java8 新特性
date: 2018-04-01 16:23:45
tags: [Java]
---

​	Java8由Oracle公司于2014年3月18号正式推出，JDK1.8中新增了许多全新的特性，支持函数式编程。

* **Lambda表达式**：Lambda允许把**函数作为**一个方法的**参数**
* **方法引用**：可以直接引用已有Java类或对象（实例）的方法或构造器。与lambda联合使用，方法引用可以使语言的构造更紧凑简洁，减少冗余代码
* **默认方法**：一个在接口里面有了一个实现的方法
* **新的编译工具**：如：Nashorn引擎 jjs、 类依赖分析器jdeps
* **Stream API**：新添加的Stream API（java.util.stream） 把真正的函数式编程风格引入到Java中。
* **Date Time API** − 加强对日期与时间的处理。
* **Optional 类** − Optional 类已经成为 Java 8 类库的一部分，用来解决空指针异常。
* **Nashorn, JavaScript 引擎** − Java 8提供了一个新的Nashorn javascript引擎，它允许我们在JVM上运行特定的javascript应用。

<!--more-->

## Lambda表达式

​	Lambda表达式，又可称作闭包。它允许将一个函数作为参数传递到方法中，从而使代码变的更为简洁紧凑。

​	**语法**:Lambda 表达式的语法格式如下：

​	(parameters) -> expression

​	(parameters) ->{ statements; }

以下是lambda表达式的重要特征:

- **可选类型声明：**不需要声明参数类型，编译器可以**统一识别参数值**。
- **可选的参数圆括号：**一个参数无需定义圆括号，但**多个参数**需要**定义**圆括号。
- **可选的大括号：**如果主体包含了一个语句，就不需要使用大括号。
- **可选的返回关键字：**如果主体只有一个表达式，则编译器会自动返回该表达式的值，若多个表达式，则需要指固定返回了一个数值。

**Java8Tester.java 文件**

~~~java
public class Java8Tester {
    public static void main(String args[]){
        Java8Tester tester = new Java8Tester();//Main是静态方法，可以声明自己
        // 类型声明
        MathOperation addition = (int a, int b) -> a + b;
        // 不用类型声明
        MathOperation subtraction = (a, b) -> a - b;
        // 大括号中的返回语句
        MathOperation multiplication = (int a, int b) -> { return a * b; };
        // 没有大括号及返回语句
        MathOperation division = (int a, int b) -> a / b;
        System.out.println("10 + 5 = " + tester.operate(10, 5, addition));
        System.out.println("10 - 5 = " + tester.operate(10, 5, subtraction));
        System.out.println("10 x 5 = " + tester.operate(10, 5, multiplication));
        System.out.println("10 / 5 = " + tester.operate(10, 5, division));
        // 不用括号
        GreetingService greetService1 = message -> System.out.println("Hello " + message);
        // 用括号
        GreetingService greetService2 = (message) -> System.out.println("Hello " + message);
        greetService1.sayMessage("Runoob");
        greetService2.sayMessage("Google");
    }
    interface MathOperation {
        int operation(int a, int b);
    }
    interface GreetingService {
        void sayMessage(String message);
    }
    private int operate(int a, int b, MathOperation mathOperation){
        return mathOperation.operation(a, b);
    }
}
~~~

输出结果为：

```
10 + 5 = 15
10 - 5 = 5
10 x 5 = 50
10 / 5 = 2
Hello Runoob
Hello Google
```

使用 Lambda 表达式需要注意以下两点：

- Lambda 表达式主要用来定义行内执行的方法类型接口，例如，一个简单方法接口。在上面例子中，我们使用各种类型的Lambda表达式来定义MathOperation接口的方法。然后我们定义了sayMessage的执行。
- Lambda 表达式免去了使用匿名方法的麻烦，并且给予Java简单但是强大的函数化的编程能力。

**变量作用域**

不能在 lambda 内部修改定义在域外的局部变量，否则会编译错误。

在 Lambda 表达式当中不允许声明一个与局部变量同名的参数或者局部变量。

```Java
String first = "";  
Comparator<String> comparator = (first, second) -> Integer.compare(first.length(), second.length());  //编译会出错 
```

## 函数式接口





## 方法引用



## 默认方法：接口

简单说，默认方法就是接口可以有实现方法，而且不需要实现类去实现其方法。

我们只需在方法名前面加个default关键字即可实现默认方法。

> **为什么要有这个特性？**
>
> ​	首先，之前的接口是个双刃剑，好处是面向抽象而不是面向具体编程，缺陷是，当需要修改接口时候，需要修改全部实现该接口的类，目前的java 8之前的集合框架没有foreach方法，通常能想到的解决办法是在JDK里给相关的接口添加新的方法及实现。然而，对于已经发布的版本，是没法在给接口添加新方法的同时不影响已有的实现。所以引进的默认方法。他们的目的是为了解决接口的修改与现有的实现不兼容的问题。
>
> ​	但是JAVA8中即使接口有了default method,但是他和抽象类是不同的。因为Java8的接口不能有状态，只是提供公有虚方法的默认实现。

**语法**

默认方法语法格式如下：

~~~java
public interface vehicle {   
    default void print(){      
        System.out.println("我是一辆车!");   
    }
}
~~~

**多个默认方法**

一个接口有默认方法，考虑这样的情况，一个类实现了多个接口，且这些接口有相同的默认方法，以下实例说明了这种情况的解决方法：

~~~java
public interface vehicle {   
    default void print(){      
        System.out.println("我是一辆车!");   
    }
}
public interface fourWheeler {   
    default void print(){      
        System.out.println("我是一辆四轮车!");   
    }
}



~~~



第一个解决方案是创建自己的默认方法，来覆盖重写接口的默认方法：

public class car implements vehicle, fourWheeler {   default void print(){      System.out.println("我是一辆四轮汽车!");   }}

第二种解决方案可以使用 super 来调用指定接口的默认方法：

public class car implements vehicle, fourWheeler {   public void print(){      vehicle.super.print();   }}

------

## 静态默认方法

Java 8 的另一个特性是接口可以声明（并且可以提供实现）静态方法。例如：

public interface vehicle {   default void print(){      System.out.println("我是一辆车!");   }    // 静态方法   static void blowHorn(){      System.out.println("按喇叭!!!");   }}

------

## 默认方法实例

我们可以通过以下代码来了解关于默认方法的使用，可以将代码放入 Java8Tester.java 文件中：

## Java8Tester.java 文件

```java
public class Java8Tester {   
  public static void main(String args[]){      
    Vehicle vehicle = new Car();      
    vehicle.print();   
  }
} 
interface Vehicle {   
  default void print(){      
    System.out.println("我是一辆车!");   
  }       
  static void blowHorn(){      
    System.out.println("按喇叭!!!");   
  }
} 
interface FourWheeler {   
  default void print(){      
    System.out.println("我是一辆四轮车!");   
  }
} 
class Car implements Vehicle, FourWheeler {   
  public void print(){      
    Vehicle.super.print();      
    FourWheeler.super.print();      
    Vehicle.blowHorn();      
    System.out.println("我是一辆汽车!");   
  }
}



```



执行以上脚本，输出结果为：

```
$ javac Java8Tester.java 
$ java Java8Tester
我是一辆车!
我是一辆四轮车!
按喇叭!!!
我是一辆汽车!
```



## 引用资料

[菜鸟教程 JDK1.8新特性](http://www.runoob.com/java/java8-default-methods.html)