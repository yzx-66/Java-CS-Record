# 进入SpringApplication

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources,
      String[] args) {
   return new SpringApplication(primarySources).run(args);
}
```

我们根据DemoApplication跟进代码，发现其调用的SpringApplication类的run方法。这个方法就干了2件事：
* 一是创建SpringApplication对象
* 二是启动SpringApplication。

# SpringApplication 构造器分析

## **1.构造器**

```java
public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
}

/**
* Create a new {@link SpringApplication} instance. The application context will load
* beans from the specified primary sources
*/
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
   this.resourceLoader = resourceLoader;
   Assert.notNull(primarySources, "PrimarySources must not be null");
   this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //根据应用是否存在某些类推断应用类型，分为响应式web应用，servlet类型web应用和非web应用，
    // 在后面用于确定实例化applicationContext的类型
   this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //设置初始化器，读取spring.factories文件key ApplicationContextInitializer对应的value并实例化
    //ApplicationContextInitializer接口用于在Spring上下文被刷新之前进行初始化的操作
   setInitializers((Collection) getSpringFactoriesInstances(
         ApplicationContextInitializer.class));

    //设置监听器，读取spring.factories文件key ApplicationListener对应的value并实例化
    // interface ApplicationListener<E extends ApplicationEvent> extends EventListener
    //ApplicationListener继承EventListener，实现了观察者模式。对于Spring框架的观察者模式实现，它限定感兴趣的事件类型需要是ApplicationEvent类型事件
   setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    //没啥特别作用，仅用于获取入口类class对象
   this.mainApplicationClass = deduceMainApplicationClass();
}
```

在构造器里主要干了件事
* 设置主类
* 推断应用类型
* 一个设置初始化器
* 二是设置监听器。

## 推断应用类型
```java
	static WebApplicationType deduceFromClasspath() {
		if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
			return WebApplicationType.REACTIVE;
		}
		for (String className : SERVLET_INDICATOR_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return WebApplicationType.NONE;
			}
		}
		return WebApplicationType.SERVLET;
	}
```

## 设置初始化器

```java
setInitializers((Collection) getSpringFactoriesInstances(
      ApplicationContextInitializer.class));
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
   return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
      Class<?>[] parameterTypes, Object... args) {
   ClassLoader classLoader = getClassLoader();
   // Use names and ensure unique to protect against duplicates
   Set<String> names = new LinkedHashSet<>(
    //从类路径的META-INF处读取相应配置文件spring.factories，然后进行遍历，读取配置文件中Key(type)对应的value
         SpringFactoriesLoader.loadFactoryNames(type, classLoader));
   //将names的对象实例化
   List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
         classLoader, args, names);
   AnnotationAwareOrderComparator.sort(instances);
   return instances;
}
```

根据入参type类型ApplicationContextInitializer.class从类路径的META-INF处读取相应配置文件spring.factories并实例化对应Initializer。上面这2个函数后面会反复用到。

```text
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
```

## 设置监听器

```java
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
```

和设置初始化器一个套路，通过getSpringFactoriesInstances函数实例化监听器。

```text
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
```

