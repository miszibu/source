---
title: Java_动态代理
date: 2018-12-08 16:23:45
tags: [Java]
---

> Java动态代理是什么，优点在哪里，Spring AOP是如何通过动态代理实现AOP的，本文从代理的设计模式出发，引入静态代理，动态代理和CGLIB代理。最终以AOP对动态代理的应用结束文章。

<!--more-->

------



## 代理：设计模式

代理是一种常用的设计模式，其目的就是为其他对象提供一个代理以**控制对某个对象的访问**。

为了保持行为的一致性，代理类和委托类通常会**实现相同的接口**，所以在访问者看来两者没有丝毫的区别。通过代理类这中间一层，能有效控制对委托类对象的直接访问，也可以很好地隐藏和保护委托类对象，同时也为实施不同控制策略预留了空间，从而在设计上获得了更大的灵活性。Java 动态代理机制以巧妙的方式近乎完美地实践了代理模式的设计理念。

* 通过实现了相同接口，让访问变成相同的调用。(在多态的帮助下，极大减小了对原代码的入侵性)
* 有效控制了对委托类对象的直接访问
  * 隐藏和保护委托类对象
  * 灵活的为不同控制策略预留了空间

![](Java_动态代理/image001.png)



------

## 静态代理---理解代理的优点， 发现静态代理的不足

在切入本文的正题之前，我们先来实现对一个接口的静态代理，来理解代理模式的具体应用。

```java
public interface Humen{
  void eat(String food);
}

// HumanImpl实现Human接口 
public class HumenImpl implements Humen{
 @Override
 public void eat(String food){
 	System.out.println("eat " + food);
 }
}
```

**问题场景：**在某处eat()方法调用时，我们不仅仅要实现吃，还要实现做菜和打扫卫生这么两个逻辑。那么有几种方式来实现呢。

1. 修改Human类的eat()方法，增加新的逻辑。
2. 增加Human类新的方法，如dinner()来实现需求。
3. 新增HumanDinner类，同样继承Human，来重新实现eat方法。

好，那我们来分析下这集中实现的利弊。

首先，第一种，直接排除，eat()方法是个被多处调用的方法，直接侵入原有代码，会出现很多问题。

第二种，增加新的方法，实际上可以实现，但是侵入性还是太强，而且做菜和打扫卫生的方法不可复用。况且原有的代码调用的就是eat()方法，最好不对方法调用进行修改。

第三种：最接近于代理类的想法，对原有代码侵入性小，但是多实现了一个类，而且这个类不能被复用，实际上不如静态代理实现。

那么说了这么多，什么是最好的方案呢，静态代理。

**静态代理实现：**

```java
// 实现相同接口，可以利用多态。
public class HumenProxy implements Humen{

 private Humen humen;

 public HumenProxy(){
 	humen = new HumenImpl();
 }

 @Override
 public void eat(String food){
	 before();
	 humen.eat(food);
	 after();
 }

 private void before(){
	 System.out.println("cook");
 }
 private void after(){
	 System.out.println("swap");
 }
}
```

**方法调用**

```java
public static void main(String[] args){
	Humen humenProxy = new HumenProxy();
 	humenProxy.eat("rice");
}
// cook
// eat
// swap
```

使用静态代理方法，对原代码的侵入性最小化，只需要修改一处。

而且控制了对Hemen类的访问，也实现了业务逻辑。是最佳的解决方案。

**静态代理的缺点：**倘若我们需要将包装好的cook()和swap()逻辑引入到别的地方时，就需要再写一套静态代理，而且一但接口被修改后，静态代理类也需要被修改。然而这两个问题都可以被动态代理解决

------

## 动态代理

### 动态代理实现

```java
public class DynamicProxy implements InvocationHandler {

    private Object target;

    public DynamicProxy(Object target){
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(target,args);
        after();
        return result;
    }
}
```

* 将委托类设置为Object，利用多态实现复用；
* 使用反射method.invoke()来调用方法，从而实现了不依赖于具体实现接口的具体方法，便成功调用委托类方法的效果。

```java
   Humen humen = new HumenImpl();
   DynamicProxy dynamicProxy = new DynamicProxy(humen);

   Humen humenProxy = (Humen) Proxy.newProxyInstance(humen.getClass().getClassLoader(), humen.getClass().getInterfaces(), dynamicProxy);
   humenProxy.eat("chicken");

   // eat chicken

```

在调用过程中，我们使用了通用的 DynamicProxy 类包装了 HumenImpl 实例，然后调用了Jdk的代理工厂方法实例化了一个具体的代理类。最后调用代理的 eat() 方法。 

我们可以看到，这个调用虽然足够灵活，可以动态生成一个具体的代理类，而不用自己显示的创建一个实现具体接口的代理类，不过调用这个代理类的过程还是有些略显复杂，与我们减少包装代码的目标不符，所以可以考虑做些小重构来简化调用过程：

```java
public class DynamicProxy implements InvocationHandler {

	...
    public<T> T getProxy(){
        return (T) Proxy.newProxyInstance(target.getClass().getClassLoader(),target.getClass().getInterfaces(),this);
    }
}

  DynamicProxy dynamicProxy = new DynamicProxy(new HumenImpl());

  Humen humenProxy = (Humen) dynamicProxy.getProxy();
  humenProxy.eat("chicken");

```

实现了一个泛型接口，返回其代理动态代理实现。达到了预期目标，对一个函数实现了封装，在其调用前后执行任意函数。而且这个动态代理类，可以注入任何类，来为这个类的方法增加前后的逻辑。

 **优点** ：通过实用 jdk 为我们提供的动态代理实现，达到了我们的 cook() 或者 swap() 方法可以被任意的复用的效果（只要我们在调用代码处使用这个通用代理类去包装任意想要需要包装的被代理类即可）。 当接口改变的时候，虽然被代理类需要改变，但是我们的代理类却不用改变了。

 **缺点**： 我们可以看到，无论是静态代理还是动态代理，它都需要一个接口。动态代理在创建代理类时需要注入该类的加载器和类的接口。那如果我们想要包装的方法，它就没有实现接口怎么办呢？这个时候就需要引入CGLIB,来实现了。

## CGLib 动态代理

CGLIB(*Code Generation Library*)是一个基于ASM的字节码生成库，它允许我们在运行时对字节码进行修改和动态生成。CGLIB**通过继承方式实现代理**

```java
public class CGLibProxy implements MethodInterceptor{ 
    public <T> T getProxy(Class<T> cls){ 
        return (T) Enhancer.create(cls,this); 
    } 
    public Object intercept(Object obj,Method method,Object[] args,MethodProxy proxy) throws Throwable{ 
        before(); 
        Object result = proxy.invokeSuper(obj,args); 
        after(); 
        return result; 
    } 
}
public static void main(String[] args){ 
    CGLibProxy cgLibProxy = new CGLibProxy(); 
    Humen humenProxy = cgLibProxy.getProxy(HumenImpl.class); 
    humenProxy.eat("rice"); 
}
```

**分析：**我们通过CGLIB的`Enhancer`来指定要代理的目标对象、实际处理代理逻辑的对象，最终通过调用`create()`方法得到代理对象，**对这个对象所有非final方法的调用都会转发给MethodInterceptor.intercept()方法**，在`intercept()`方法里我们可以加入任何逻辑，比如修改方法参数，加入日志功能、安全检查功能等；通过调用`MethodProxy.invokeSuper()`方法，我们将调用转发给原始对象，具体到本例，就是`eat`的具体方法。CGLIG中[MethodInterceptor](http://cglib.sourceforge.net/apidocs/net/sf/cglib/proxy/MethodInterceptor.html)的作用跟JDK代理中的`InvocationHandler`很类似，都是方法调用的中转站。

**优点**：JDK动态代理需要提供类的接口信息，好对某个具体的方法进行拦截。因此对于没有接口的类，CGLib就可以对其进行动态代理。

**缺点：**CGLib会对该类所有方法都进行拦截，获取了灵活性，但是效率没有动态代理高。

> Tips: 对于从Object中继承的方法，CGLIB代理也会进行代理，如`hashCode()`、`equals()`、`toString()`等，但是`getClass()`、`wait()`等方法不会，因为它是final方法，CGLIB无法代理。
>
> CGLib是通过继承来实现动态代理的，由于final类不能有子类，因此final类不可静态代理。
>
> final方法不能被重载，因此final方法也不能被CGLib代理。

------



## 相关资料

[Java 动态代理机制分析及扩展](https://www.ibm.com/developerworks/cn/java/j-lo-proxy1/index.html)

[Java Proxy 和 CGLIB 动态代理原理](http://www.importnew.com/27772.html)

[Java 动态代理作用是什么？](https://www.zhihu.com/question/20794107)