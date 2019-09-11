---
title: SpringBoot启动类源码——refresh方法剖析
date: 2019-01-07 15:09:27
tags: [Spring]
---

> 在SpringBoot启动类源码分析一文中，我们已经看到了SpringBoot的启动过程。
>
> 首先创建SpringApplication对象，获取配置文件中定义的initializer和listener
>
> 再执行run方法，初始化ApplicationContext，再执行initializer方法，设置环境变量，
>
> refresh Spring容器，再执行AfterRefresh方法。
>
> 其中refresh方法至关重要，本文将进入该方法，一起来看看它都做了什么事情。

<!--more-->

## SpringApplication的refresh

```java
private void refreshContext(ConfigurableApplicationContext context) {
		refresh(context);
    	// 一个Hook bind jvm and SpringApp
    	// when jvm shutdown, close the springApplication
		if (this.registerShutdownHook) {
			try {
				context.registerShutdownHook();
			}
			catch (AccessControlException ex) {
				// Not allowed in some environments.
			}
		}
	}

protected void refresh(ApplicationContext applicationContext) {
    	// 断言判定是否为AbstractApplicationContext子类
		Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    	// 调用Spring容器的refresh
		((AbstractApplicationContext) applicationContext).refresh();
	}
```

这里主要是启用了Spring容器的refresh方法

本文案例是web程序，那么对应的Spring容器为**AnnotationConfigEmbeddedWebApplicationContext**。

它的refresh方法调用了**父类AbstractApplicationContext的refresh**方法：

------



## ApplicaitonContext的refresh

```java
public void refresh() throws BeansException, IllegalStateException {
    	// destory和refresh方法共用的类成员变量对象锁
		synchronized (this.startupShutdownMonitor) {
            
			// Prepare this context for refreshing.
            // 准备context refresh所需要的参数环境 
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
            // 告知子类刷新内部bean factory
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}

/** Synchronization monitor for the "refresh" and "destroy" */
// refresh和destory方法共用的 成员变量对象锁
private final Object startupShutdownMonitor = new Object();
```

------



### prepareRefresh()

```java
protected void prepareRefresh() {
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isInfoEnabled()) {
			logger.info("Refreshing " + this);
		}

		// Initialize any placeholder property sources in the context environment
		initPropertySources();

		// Validate that all properties marked as required are resolvable
		// see ConfigurablePropertyResolver#setRequiredProperties
		getEnvironment().validateRequiredProperties();

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
	}
```

1. 设置Spring容器的启动时间，撤销关闭状态，开启活跃状态。
2. 初始化属性源信息(Property)
3. 验证环境信息里一些必须存在的属性

------

### obtainFreshBeanFactory()

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   		// 刷新beanFactroy 根据父类的类不同 实现不同
		refreshBeanFactory();
    	// 获取BeanFactory 并且返回
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```

本文是web环境，因此加载了**AnnotationConfigEmbeddedWebApplicationContext**。

该ApplicaitonContext继承自GenericApplicationContext，在创建ApplicationContext时，就生成了内部类**DefaultListableBeanFactory**，刷新时不会摧毁并重建。

详情见[ApplicationContext类继承设计一文]

### prepareBeanFactory方法

```java
/**
	 * Configure the factory's standard context characteristics,
	 * such as the context's ClassLoader and post-processors.
	 * @param beanFactory the BeanFactory to configure
	 */
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
        // 配置DefaultBeanFactory相关参数 如类加载器，Bean解析器，属性注册器等
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
        // 添加BeanPostProcessor 并 忽略无用依赖
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new 	ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}

```

从Spring容器获取BeanFactory(Spring Bean容器)并进行相关的设置为后续的使用做准备：

1. 设置classloader(用于加载bean)，设置beanExpressionResolver表达式解析器(解析bean定义中的一些表达式)，添加属性编辑注册器(注册属性编辑器)

2. 添加ApplicationContextAwareProcessor这个BeanPostProcessor。

   取消ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware、EnvironmentAware这5个接口的自动注入。因为ApplicationContextAwareProcessor把这5个接口的实现工作做了

3. 设置特殊的类型对应的bean。BeanFactory对应刚刚获取的BeanFactory；ResourceLoader、ApplicationEventPublisher、ApplicationContext这3个接口对应的bean都设置为当前的Spring容器

4. 注入一些其它信息的bean，比如environment、systemProperties等

------

### postProcessBeanFactory(beanFactory)

BeanFactory设置之后再进行后续的一些BeanFactory操作。

不同的Spring容器做不同的操作。比如GenericWebApplicationContext容器会在BeanFactory中添加ServletContextAwareProcessor用于处理ServletContextAware类型的bean初始化的时候调用setServletContext或者setServletConfig方法(跟ApplicationContextAwareProcessor原理一样)。

AnnotationConfigEmbeddedWebApplicationContext对应的postProcessBeanFactory方法：

```java
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  // 调用父类EmbeddedWebApplicationContext的实现
  super.postProcessBeanFactory(beanFactory);
  // 查看basePackages属性，如果设置了会使用ClassPathBeanDefinitionScanner去扫描basePackages包下的bean并注册
  if (this.basePackages != null && this.basePackages.length > 0) {
    this.scanner.scan(this.basePackages);
  }
  // 查看annotatedClasses属性，如果设置了会使用AnnotatedBeanDefinitionReader去注册这些bean
  if (this.annotatedClasses != null && this.annotatedClasses.length > 0) {
    this.reader.register(this.annotatedClasses);
  }
}
```

父类EmbeddedWebApplicationContext的实现：

```java
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  beanFactory.addBeanPostProcessor(
      new WebApplicationContextServletContextAwareProcessor(this));
  beanFactory.ignoreDependencyInterface(ServletContextAware.class);
}
```



**未完待续** 