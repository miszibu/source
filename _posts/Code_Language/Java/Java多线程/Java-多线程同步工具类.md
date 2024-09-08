---
title: Java-多线程同步工具类
date: 2020-05-27 00:01:32
tags: [Java, 多线程]
---

# Java同步工具类

基于Java AQS类，JUC提供了以下几个方便实用的工具类，用于管理线程之间同步的关系。

* **CountDownLatch**： 适用于线程间存在先后顺序关系的场景。某个线程需要等待其他线程完成后，才开始的场景。
* **CyclicBarrier**：适用于线程间存在同步的场景，所有线程到达一个检查点就暂停，等待其他线程都到达，再一起继续。
* **Semaphore**： 信号量，适用于资源竞争的场景。抢占到资源的线程继续，没有抢占到的则等待。

<!--more-->

# CountDownLatch

> A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.
>
> 一个同步辅助类，允许多个线程一直等待直至其他线程它们的操作。

它常用的API是: `await()`和`countDown()`

## 使用说明

初始化CountDownLatch时，设置计数器，

* countDown(): 使计数器count值减1
* await(): 若count为0，则继续；否则，加入等待线程队列，等待唤醒

## 例子

```java
import java.util.concurrent.CountDownLatch;

// 假设在一个集群自动Recovery的项目中，我们想要启动Kafka，Kafka依赖于ZK,我们希望等待ZK状态正常后，再启动Kafka.
public class Main {
    public static void main(String[] args) throws InterruptedException {
        final CountDownLatch countDownLatch = new CountDownLatch(1);

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println("Ready to start Zk");
                    Thread.sleep(1000);
                    countDownLatch.countDown();
                    System.out.println("ZKs are working");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        new Thread(()->{
            try {
                System.out.println("Waiting zks status");
                // 这里调用的是await()不是wait()
                countDownLatch.await();
                System.out.println("Trying to start kafka");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}

//输出结果
Ready to start Zk
Waiting zks status
ZKs are working
Trying to start kafka
```



# CyclicBarrier

> A synchronization aid that allows a set of threads to all wait for each other to reach a common barrier point. CyclicBarriers are useful in programs involving a fixed sized party of threads that must occasionally wait for each other. The barrier is called <em>cyclic</em> because it can be re-used after the waiting threads are released.
>
> 简单来说，CyclicBarrier允许一系列线程互相等待，直至所有人都达到一个共同的屏障点。
>
> 之所以叫做cyclic，因为当所有线程都达到屏障点后，在等待线程被释放以后，它可以继续重复使用。

## 使用说明

 CountDownLatch注重的是**等待其他线程完成**，CyclicBarrier注重的是：当线程**到达某个状态**后，暂停下来等待其他线程，**所有线程均到达以后**，继续执行。 

## 例子

```java
import java.util.Random;
import java.util.concurrent.BrokenBarrierException;

import java.util.concurrent.CyclicBarrier;

// 一个开会的场景,所有人都需要到齐才能开会,等所有人都觉得累了,就开始休息。
public class Main {

    public static void main(String[] args) throws InterruptedException {
        final CyclicBarrier cyclicBarrier = new CyclicBarrier(3);
        Random random = new Random();

        for (int i=0; i<3 ;i++){
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        System.out.println("A new colleague come in");
                        // 等待所有人到齐，开始会议
                        cyclicBarrier.await();
                        System.out.println("All teams members are ready. Start the meeting");

                        Thread.sleep(random.nextInt(1000));
                        System.out.println("I am tired");
                        // 等待所有人都累了，就休息
                        cyclicBarrier.await();
                        System.out.println("Have a rest");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
    }
}

//Output
A new colleague come in
A new colleague come in
A new colleague come in
All teams members are ready. Start the meeting
All teams members are ready. Start the meeting
All teams members are ready. Start the meeting
I am tired
I am tired
I am tired
Have a rest
Have a rest
Have a rest
```



# Semaphore

> Semaphores are often used to **restrict the number of threads than can access some (physical or logical) resource**. 
>
> 信号量往往用于限定某个资源的使用量

## 使用说明

设定了Count作为计数器，

- 调用`acquire()`方法时，count--。如果Count<0，就加入到等待线程列表。
- 调用`release()`方法时，count++。

## 例子

```java
import java.util.Random;
import java.util.concurrent.Semaphore;

// 场景：一个只能容纳2人的小餐馆，一个顾客离开，下一个顾客才能进来
public class Main {

    public static void main(String[] args) throws InterruptedException {

        Semaphore semaphore = new Semaphore(2);
        Random random = new Random();
        for (int i = 0; i<4;i++){
            new Thread(()->{
                try {
                    // 获取信号量，进来吃饭
                    semaphore.acquire();
                    System.out.println("Custom come in");
                    // 随机吃饭一段时间
                    Thread.sleep(random.nextInt(3000));
                    System.out.println("Custom leave");
                    // 吃完离开，释放信号量
                    semaphore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}

//Output
Custom come in
Custom come in
Custom leave
Custom come in
Custom leave
Custom come in
Custom leave
Custom leave
```

# Reference

[Java多线程打辅助的三个小伙子](https://zhuanlan.zhihu.com/p/40669746)
