---
layout: springboot
title: ApplicationContext类继承设计
date: 2019-01-07 15:56:34
tags: [Spring]
---

> 本文以AnnotationConfigEmbeddedWebApplicationContext为例，使用Idea的类图插件，查看该类的继承图，并对此进行分析。

<!--more-->

![](/Users/ligaofeng/blog/source/_posts/Spring/ApplicationContext类继承设计/AnnotationConfigEmbeddedWebApplicationContext.png)

图中我们可以看到AnnotationConfigEmbeddedWebApplicationContext继承了EmbeddedWebApplicationContext和GenericWebApplicationContext类。



## ApplicationContext实现类设计

ApplicationContext有两大子类

* **GenericApplicationContext**
* **AbstractRefreshableApplicationContext。**

**GenericApplictionContext及其子类**持有一个单例的固定的**DefaultListableBeanFactory**实例，在创建GenericApplicationContext实例的时候就会创建DefaultListableBeanFactory实例。固定的意思就是说，即使调用refresh方法，也不会重新创建BeanFactory实例。

与之对应的就是**AbstractRefreshableApplicationContext**，它实现了所谓的热刷新功能，它内部也持有一个DefaultListableBeanFactory实例，每次**刷新refresh()**时都会**销毁当前的BeanFactory实例并重新创建DefaultListableBeanFactory**

------



## BeanFactory

**BeanFactory**及其实现类是Spring IOC的核心接口，实现了init和get Bean等功能，**负责生产和管理Bean**。(例如DefaultListableBeanFactory)

Spring提供了**ApplicationContext体系，扩展了BeanFactory。**实现了例如：资源加载，消息源处理，事件机制，生命周期管理等等一系列实际开发应用中所需要的高级特性。

* **支持不同的消息源**：ApplicationContext扩展了**MessageSource**,使Spring得以处理国际化信息，为开发多语言版本的应用提供服务。
* **访问资源**： 这个特性体现在对是**ResourceLoader和ResourcePatternResolver**的支持上，这样我们可以从不同的地方得到定义Bean的资源。这样，可以灵活的定义bean定义信息，例如从不通的IO途径或者网络资源来得到Beans定义信息
* **支持应用事件**，获得事件响应功能：**ApplicationEventPublisher**拥有发布事件功能，为Spring引入事件机制。
* **生命周期管理（Lifecycle）**
* **运行环境设置（EnvironmentCapable）。**

---------------------


## 相关资料

[ApplicationContext类继承设计](https://blog.csdn.net/liu_shi_jun/article/details/80173410 )