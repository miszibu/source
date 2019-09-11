---
title: SpringBoot启动类源码分析
date: 2019-01-06 14:23:10
tags: [Spring]
---

> 本文从SpringBoot启动类进入分析，看看SpringBoot项目启动时都进行了哪些操作
>
> 版本：SpringBoot 1.5.8

<!--more-->

## SpringBoot启动类

```java
@SpringBootApplication
public class MainApplication{
	public static void main(String[] args) {
		SpringApplication.run(MainApplication.class, args);
	}
}
```

------



## @SpringBootApplication注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    //...
}
```

这是一个集成注解，集成了三个注解

1. **@SpringBootConfiguration**

   该注解其实就是@Configuration，标注了该注解的类具备了以Java代码的形式，编写JavaConfig的能力。

2. **@EnableAutoConfiguration**

   该注解让SpringBoot具备了自动化注入配置的能力，比较复杂，本文暂且略过。

3. **@ComponentScan**

   该注解实现了自动扫描的能力，类似于XML配置中的`<context:component-scan>`

   可以指定basePackages属性指定要扫描的包，以及扫描的条件。

   **默认扫描@ComponentScan注解所在类的同级类和同级目录下的所有类**，所以对于一个Spring Boot项目，一般会把入口类放在顶层目录中，这样就能够保证源码目录下的所有类都能够被扫描到。

------



## 启动流程

```java
// 启动类调用run静态方法，传入启动类class和启动参数
public static void main(String[] args) {
   SpringApplication.run(MainApplication.class, args);
}

// new一个SpringApplication类 然后 执行其run方法
// object[] sources是 传入的入口类.class
public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
		return new SpringApplication(sources).run(args);
}
```



------



## SpringApplicationn构造函数

```java
// 调用initialize方法
public SpringApplication(Object... sources) {
		initialize(sources);
}

private void initialize(Object[] sources) {
    	// 判断启动类Class数组 若正常 则加入source
		if (sources != null && sources.length > 0) {
			this.sources.addAll(Arrays.asList(sources));
		}
    	// 判断当前是否为Web环境
    	// 通过class.forName{ "javax.servlet.Servlet","org.springframework.web.context.ConfigurableWebApplicationContext" }
		this.webEnvironment = deduceWebEnvironment();
    	// 设置初始化类：从所有配置文件spring.factories中查找所有的key=org.springframework.context.ApplicationContextInitializer的类【加载，初始化，排序】
        // SpringFactoriesLoader:工厂加载机制
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
    	// 设置Listeners：从所有配置文件spring.factories中查找所有的key=org.springframework.context.ApplicationListener的类.【加载，初始化，排序】
        setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	    // 从当前调用栈中，查找主类
		this.mainApplicationClass = deduceMainApplicationClass();
}

// source类 存储了当前的启动类.class
private final Set<Object> sources = new LinkedHashSet<Object>();
// 记录当前是否为Web环境
private boolean webEnvironment;
// SpringApplication的初始化器列表
private List<ApplicationContextInitializer<?>> initializers;
// SpringApplication事件监听器列表
private List<ApplicationListener<?>> listeners;


```

### getSpringFactoriesInstance()

Initialize方法中 ，Initializers和Listeners的加载过程都是使用到了`SpringFactoriesLoader`工厂加载机制。我们进入到`getSpringFactoriesInstances`这个方法中：

```java
// 典型的重载 写一个全参方法 其余重载方法调用全参方法
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type) {
		return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
    	// 获取当前类加载器
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
		// Use names and ensure unique to protect against duplicates
	    // 获取META-INF/spring.factories文件中key为type类型的所有的类全限定名。注意是所有jar包内的。
		Set<String> names = new LinkedHashSet<String>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
	    // 通过上面获取到的类的全限定名，这里将会使用Class.forName加载类，并调用构造方法实例化类
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
	    // 根据类上的org.springframework.core.annotation.Order注解，排序
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
}
```

### ApplicationContextInitializer，应用程序初始化器

Spring有多个initializer，通过在框架启动过程中，遍历调用initialize方法完成初始化工作：

```java
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {
	void initialize(C applicationContext);
}
```

### ApplicationListener，应用程序事件(ApplicationEvent)监听器：

```java
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
	void onApplicationEvent(E event);
}
```

应用程序事件(ApplicationEvent)有应用程序启动事件(ApplicationStartedEvent)，失败事件(ApplicationFailedEvent)，准备事件(ApplicationPreparedEvent)等。

**ApplicationListener与ApplicationEvent相绑定**。

* ConfigServerBootstrapApplicationListener只跟ApplicationEnvironmentPreparedEvent事件绑定，
* LiquibaseServiceLocatorApplicationListener只跟ApplicationStartedEvent事件绑定，
* LoggingApplicationListener跟所有事件绑定等。

### initialize流程

1. 将启动类加入this.source(LinkedHashSet)
2. 通过反射判断是否是Web环境
3. 从所有类中查找META-INF/spring.factories文件，加载其中的初始化类和监听类。通过Key查找
   * ApplicationContextInitializer.class
   * ApplicationListener.class

4. 查找运行的主类 默认初始化Initializers都继承自ApplicationContextInitializer。

------



## Run方法

### 前置知识: SpringApplicaitonRunListener

在分析run方法前，先来了解SpringApplication事件和监听器概念。

这里出现了两个类

* **SpringApplicationRunListener**：一个interface定义了5个方法
  * void starting(); 在run方法启动时调用，对应事件的类型是ApplicationStartedEvent
  * void environmentPrepared(ConfigurableEnvironment environment); 在环境变量配置完成后，在ApplicationContext创建前，对应事件的类型是ApplicationEnvironmentPreparedEvent
  * void contextPrepared(ConfigurableApplicationContext context); ApplicationContext创建好并且在source加载之前调用一次；没有具体的对应事件
  * void contextLoaded(ConfigurableApplicationContext context); ApplicationContext创建并加载之后并在refresh之前调用；对应事件的类型是ApplicationPreparedEvent
  * void finished(ConfigurableApplicationContext context, Throwable exception);run方法结束之前调用；对应事件的类型是ApplicationReadyEvent或ApplicationFailedEvent

* **SpringApplicationRunListener：**持有SpringApplicationRunListener集合和1个Log日志类。用于SpringApplicationRunListener监听器的批量执行
  * **private final Log log;**
  * **private final List<SpringApplicationRunListener> listeners;**

* **EventPublishingRunListener：**SpringApplicationRunListener唯一实现类

  * **private final SimpleApplicationEventMulticaster initialMulticaster;**  利用组合的方式，作为内部成员变量，用于将监听的过程封装成的SpringApplicationEvent事件，广播出去，广播出去的事件，最终会被ApplicationListener监听。

  所以说SpringApplicationRunListener和ApplicationListener之间的关系是通过ApplicationEventMulticaster广播出去的SpringApplicationEvent所联系起来的。

![](startup2.jpg)

### Run方法

```java
public ConfigurableApplicationContext run(String... args) {
    	// StopWatch 用于监听run启动过程
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		FailureAnalyzers analyzers = null;
		configureHeadlessProperty();
	    // 获取监听器 成员变量监听器的List实际上只有一个EventPublishingRunListener
		SpringApplicationRunListeners listeners = getRunListeners(args);
    	// 遍历监听器集合 调用其starting方法
    	// 包装ApplicationEvent然后广播  详情见前置知识
		listeners.starting();
		try {
            // 加载属性配置，执行完成后，所有的environment的属性都会加载进来，包括application.properties和外部的属性配置。
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
            
            // 打印Banner
			Banner printedBanner = printBanner(environment);
            
            // 创建spring容器 根据之前判断的webEnvironment加载不同的SpirngApplicationContext
            context = createApplicationContext();
           
            // 错误分析器
            analyzers = new FailureAnalyzers(context);
            
            // 主要是调用所有初始化类的initialize方法
            prepareContext(context, environment, listeners, applicationArguments,
                    printedBanner);
            
            // 初始化spring容器【后面写到】
            refreshContext(context);
            // 主要是执行ApplicationRunner和CommandLineRunner的实现类【后面会写到】
            afterRefresh(context, applicationArguments);
            // 通知监听器
            listeners.finished(context, null);
            stopWatch.stop();
            if (this.logStartupInfo) {
                new StartupInfoLogger(this.mainApplicationClass)
                        .logStarted(getApplicationLog(), stopWatch);
            }
            return context;
		}
		catch (Throwable ex) {
            // 执行异常判断、然后广播出ApplicationFailedEvent事件给相应的监听器执行
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}
```

#### createApplicationContext()

```java
 // 使用反射实例化ApplicationContext
 // web应用程序时使用 AnnotationConfigEmbeddedWebApplicationContext 
 // 否则为 AnnotationConfigApplicationContext。          
protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
		if (contextClass == null) {
			try {
				contextClass = Class.forName(this.webEnvironment
						? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Unable create a default ApplicationContext, "
								+ "please specify an ApplicationContextClass",
						ex);
			}
		}
		return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
	}
```

#### prepareContext()

```java
	private void prepareContext(ConfigurableApplicationContext context,
            ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
            ApplicationArguments applicationArguments, Banner printedBanner) {
	    // 设置环境变量
        context.setEnvironment(environment);
        
        // 将SpringApplication的一些相关设置参数同步给Spring容器(ApplicationContext)
        postProcessApplicationContext(context);
        
        // SpringApplication的的初始化器开始工作。主要是循环遍历调用所有Initializers的initialize方法
        applyInitializers(context);
        
        // 广播Context相关必要操作完成 目前为空实现
        listeners.contextPrepared(context);
        if (this.logStartupInfo) {
            logStartupInfo(context.getParent() == null);
            logStartupProfileInfo(context);
        }

        // 注册单例：applicationArguments
        context.getBeanFactory().registerSingleton("springApplicationArguments",
                applicationArguments);
        if (printedBanner != null) {
            context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
        }

        // Load the sources: 只有一个主类：MainApplication
        Set<Object> sources = getSources();
        Assert.notEmpty(sources, "Sources must not be empty");
        // 加载bean，目前只有一个主类：MainApplication
        load(context, sources.toArray(new Object[sources.size()]));
        // 广播容器加载完成事件
        listeners.contextLoaded(context);
    }
```

##### postProcessApplicationContext()

```java
protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
		if (this.beanNameGenerator != null) {
            // 如果SpringApplication设置了实例命名生成器，注册到Spring容器(ApplicationContext)中
			context.getBeanFactory().registerSingleton(
					AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
					this.beanNameGenerator);
		}
		if (this.resourceLoader != null) {
            // 如果SpringApplication设置了资源加载器，设置到Spring容器中
			if (context instanceof GenericApplicationContext) {
				((GenericApplicationContext) context)
						.setResourceLoader(this.resourceLoader);
			}
			if (context instanceof DefaultResourceLoader) {
				((DefaultResourceLoader) context)
						.setClassLoader(this.resourceLoader.getClassLoader());
			}
		}
	}
```

##### applyInitializers()

```java
protected void applyInitializers(ConfigurableApplicationContext context) {
    	// 遍历每个初始化器，对调用对应的initialize方法
		for (ApplicationContextInitializer initializer : getInitializers()) {
			Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
					initializer.getClass(), ApplicationContextInitializer.class);
			Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
			initializer.initialize(context);
		}
	}
```

初始化器做的工作，举例如下

* ContextIdApplicationContextInitializer会设置应用程序的id；
* AutoConfigurationReportLoggingInitializer会给应用程序添加一个条件注解解析器报告等：

#### refreshContext()

```java
private void refreshContext(ConfigurableApplicationContext context) {
		// 之前都是相关变量参数的设置
    	// refresh方法完成了许多context上下文数据的工作 单独开一文来分析
		refresh(context);
    	
    	// 注册一个Hook来绑定ApplicationContext和JVM
    	// 当JVM关闭时，关闭ApplicaitonContext
		if (this.registerShutdownHook) {
			try {
				context.registerShutdownHook();
			}
			catch (AccessControlException ex) {
				// Not allowed in some environments.
			}
		}
	}
```

#### afterRefresh()

```java
	protected void afterRefresh(ConfigurableApplicationContext context,
			ApplicationArguments args) {
		callRunners(context, args);
	}

	private void callRunners(ApplicationContext context, ApplicationArguments args) {
		List<Object> runners = new ArrayList<Object>();
        // 找出Spring容器中ApplicationRunner接口的实现类
		runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
        // 找出Spring容器中CommandLineRunner接口的实现类
		runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
		AnnotationAwareOrderComparator.sort(runners);
        // 去重后顺序调用去重后的 class类run方法
		for (Object runner : new LinkedHashSet<Object>(runners)) {
			if (runner instanceof ApplicationRunner) {
				callRunner((ApplicationRunner) runner, args);
			}
			if (runner instanceof CommandLineRunner) {
				callRunner((CommandLineRunner) runner, args);
			}
		}
	}

	private void callRunner(ApplicationRunner runner, ApplicationArguments args) {
		try {
			(runner).run(args);
		}
		catch (Exception ex) {
			throw new IllegalStateException("Failed to execute ApplicationRunner", ex);
		}
	}

	private void callRunner(CommandLineRunner runner, ApplicationArguments args) {
		try {
			(runner).run(args.getSourceArgs());
		}
		catch (Exception ex) {
			throw new IllegalStateException("Failed to execute CommandLineRunner", ex);
		}
	}
```

------



## 总结

SpringBoot启动的时候，会**构造一个SpringApplication的实例，然后调用这个实例的run方法**，这样就表示启动SpringBoot。

**SpringApplication初始化工作**

1. **把参数sources设置到SpringApplication属性中**，这个sources可以是任何类型的参数。本文的例子中这个sources就是MainApplication的class对象
2. **判断是否是web程序**，并设置到webEnvironment这个boolean属性中
3. 找出所有的**初始化器**，默认有5个，设置到initializers属性中
4. 找出所有的**应用程序监听器**，默认有9个，设置到listeners属性中
5. **找出运行的主类(main class)**

**调用SpringApplication  run方法，启动SpringApplication**，run方法执行的时候会做以下几件事：

1. 构造一个StopWatch，观察SpringApplication的执行

2. 找出所有的SpringApplicationRunListener并封装到SpringApplicationRunListeners中，用于监听run方法的执行。监听的过程中会封装成事件并广播出去让初始化过程中产生的应用程序监听器进行监听

3. **构造Spring容器(ApplicationContext)**

    创建Spring容器的判断是否是web环境，是的话构造AnnotationConfigEmbeddedWebApplicationContext，否则构造AnnotationConfigApplicationContext

4. **初始化ApplicationContext容器**

5. **刷新Spring容器**(完成bean的解析、各种processor接口的执行、条件注解的解析等等

6. 从Spring容器中找出ApplicationRunner和CommandLineRunner接口的实现类并排序后依次执行