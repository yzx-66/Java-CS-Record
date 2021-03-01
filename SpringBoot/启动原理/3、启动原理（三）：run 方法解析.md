# run(String... args)解析
**run函数**

```java
/**
* Run the Spring application, creating and refreshing a new ApplicationContext
*/

public ConfigurableApplicationContext run(String... args) {
   //计时器
   StopWatch stopWatch = new StopWatch();
   stopWatch.start();

   ConfigurableApplicationContext context = null;
   Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();

   //设置java.awt.headless系统属性为true，Headless模式是系统的一种配置模式。
   // 在该模式下，系统缺少了显示设备、键盘或鼠标。但是服务器生成的数据需要提供给显示设备等使用。
   // 因此使用headless模式，一般是在程序开始激活headless模式，告诉程序，现在你要工作在Headless        mode下，依靠系统的计算能力模拟出这些特性来
   configureHeadlessProperty();

   //获取监听器集合对象
   SpringApplicationRunListeners listeners = getRunListeners(args);

   //发出开始执行的事件。
   listeners.starting();

   try {
      //根据main函数传入的参数，创建DefaultApplicationArguments对象
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);
      //根据扫描到的监听器对象和函数传入参数，进行环境准备。
      ConfigurableEnvironment environment = prepareEnvironment(listeners,
            applicationArguments);

      configureIgnoreBeanInfo(environment);
      Banner printedBanner = printBanner(environment);

      context = createApplicationContext();

      //和上面套路一样，读取spring.factories文件
      //key 是 SpringBootExceptionReporter 对应的value
      exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);

      prepareContext(context, environment, listeners, applicationArguments,
            printedBanner);

      //和上面的一样，context准备完成之后，将触发SpringApplicationRunListener的contextPrepared执行
      refreshContext(context);

      //其实啥也没干。但是老版本的callRunners好像是在这里执行的。
      afterRefresh(context, applicationArguments);

      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass)
               .logStarted(getApplicationLog(), stopWatch);
      }
      //发布ApplicationStartedEvent事件，发出结束执行的事件
      listeners.started(context);
      //在某些情况下，我们希望在容器bean加载完成后执行一些操作，会实现ApplicationRunner或者CommandLineRunner接口
      //后置操作，就是在容器完成刷新后，依次调用注册的Runners，还可以通过@Order注解设置各runner的执行顺序。
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, listeners);
      throw new IllegalStateException(ex);
   }

   try {
      listeners.running(context);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, null);
      throw new IllegalStateException(ex);
   }
   return context;
}
```

## **1.获取run listeners**

```java
SpringApplicationRunListeners listeners = getRunListeners(args);
```

和构造器设置初始化器一个套路，根据传入type  SpringApplicationRunListener去扫描spring.factories文件，读取type对应的value并实例化。然后利用实例化对象创建SpringApplicationRunListeners对象。

```java
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```

EventPublishingRunListener的作用是发布SpringApplicationEvent事件。
* 内部有一个 SimpleApplicationEventMulticaster 

```java
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {

	private final SpringApplication application;

	private final String[] args;

	private final SimpleApplicationEventMulticaster initialMulticaster;

	public EventPublishingRunListener(SpringApplication application, String[] args) {
		this.application = application;
		this.args = args;
		this.initialMulticaster = new SimpleApplicationEventMulticaster();
		for (ApplicationListener<?> listener : application.getListeners()) {
			this.initialMulticaster.addApplicationListener(listener);
		}
	}
	
	...
```

## **2.发出开始执行的事件**

```java
listeners.starting();
```

继续跟进starting函数，

```java
public void starting() {
   this.initialMulticaster.multicastEvent(
         new ApplicationStartingEvent(this.application, this.args));
}
```
SimpleApplicationEventMulticaster.multicastEvent
```java
	//获取ApplicationStartingEvent类型的事件后，发布事件
    @Override
    public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
        ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
        for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
            Executor executor = getTaskExecutor();
            if (executor != null) {
                executor.execute(() -> invokeListener(listener, event));
            }
            else {
                invokeListener(listener, event);
            }
        }
    }
	//继续跟进invokeListener方法,最后调用ApplicationListener监听者的onApplicationEvent处理事件
    private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
        try {
            listener.onApplicationEvent(event);
        }
        catch (ClassCastException ex) {
            .....
        }
    }
```

这个后面也会反复遇到，比如listeners.running(context)。

这里是典型的观察者模式。

```java
//观察者：监听<E extends ApplicationEvent>类型事件
ApplicationListener<E extends ApplicationEvent> extends EventListener

//事件类型：
Event extends SpringApplicationEvent  extends ApplicationEvent extends EventObject

//被观察者：发布事件
EventPublishingRunListener implements SpringApplicationRunListener
```

SpringApplication根据当前事件Event类型，比如ApplicationStartingEvent，查找到监听ApplicationStartingEvent的观察者EventPublishingRunListener，调用观察者的onApplicationEvent处理事件。

## **3.环境准备**

```java
//根据main函数传入的参数，创建DefaultApplicationArguments对象
ApplicationArguments applicationArguments = new DefaultApplicationArguments(
      args);
//根据扫描到的listeners对象和函数传入参数，进行环境准备。
ConfigurableEnvironment environment = prepareEnvironment(listeners,
      applicationArguments);
```

ApplicationArguments提供运行application的参数，后面会作为一个Bean注入到容器。这里重点说下prepareEnvironment方法做了些什么。

```java
private ConfigurableEnvironment prepareEnvironment(
      SpringApplicationRunListeners listeners,
      ApplicationArguments applicationArguments) {

   // Create and configure the environment
   ConfigurableEnvironment environment = getOrCreateEnvironment();

   configureEnvironment(environment, applicationArguments.getSourceArgs());

    //和listeners.starting一样的流程
   listeners.environmentPrepared(environment);

   //上述完成了环境的创建和配置，传入的参数和资源加载到environment

   //绑定环境到SpringApplication
   bindToSpringApplication(environment);
   if (!this.isCustomEnvironment) {
      environment = new EnvironmentConverter(getClassLoader())
            .convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
   }
   ConfigurationPropertySources.attach(environment);
   return environment;
}
```

这段代码核心有3个。

1. configureEnvironment，用于基本运行环境的配置。
2. 发布事件ApplicationEnvironmentPreparedEvent。和发布ApplicationStartingEvent事件的流程一样。
3. 绑定环境到SpringApplication

## **4.创建ApplicationContext**

```java
context = createApplicationContext();
```

传说中的IOC容器终于来了。

在实例化context之前，首先需要确定context的类型，这个是根据应用类型确定的。应用类型webApplicationType在构造器已经推断出来了。

```java
protected ConfigurableApplicationContext createApplicationContext() {
   Class<?> contextClass = this.applicationContextClass;
   if (contextClass == null) {
      try {
         switch (this.webApplicationType) {
         case SERVLET:
            //应用为servlet类型的web应用
            contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
            break;
         case REACTIVE:
            //应用为响应式web应用
            contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
            break;
         default:
            //应用为非web类型的应用
            contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
         }
      }
      catch (ClassNotFoundException ex) {
         throw new IllegalStateException(
               "Unable create a default ApplicationContext, "
                     + "please specify an ApplicationContextClass",
               ex);
      }
   }
   return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

获取context类型后，进行实例化，这里根据class类型获取无参构造器进行实例化。

```java
public static <T> T instantiateClass(Class<T> clazz) throws BeanInstantiationException {
   Assert.notNull(clazz, "Class must not be null");
   if (clazz.isInterface()) {
      throw new BeanInstantiationException(clazz, "Specified class is an interface");
   }
   try {
       //clazz.getDeclaredConstructor()获取无参的构造器，然后进行实例化
      return instantiateClass(clazz.getDeclaredConstructor());
   }
   catch (NoSuchMethodException ex) {
    .......
}
```

比如web类型为servlet类型，就会实例化org.springframework.boot.web.servlet.context.

AnnotationConfigServletWebServerApplicationContext类型的context。

## **5.context前置处理阶段**

```java
private void prepareContext(ConfigurableApplicationContext context,
      ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
      ApplicationArguments applicationArguments, Banner printedBanner) {
   //关联环境
   context.setEnvironment(environment);

   //ApplicationContext预处理，主要配置Bean生成器以及资源加载器
   postProcessApplicationContext(context);
    
   //调用初始化器，执行initialize方法，前面set的初始化器终于用上了
   applyInitializers(context);
   //发布contextPrepared事件，和发布starting事件一样，不多说
   listeners.contextPrepared(context);
   if (this.logStartupInfo) {
      logStartupInfo(context.getParent() == null);
      logStartupProfileInfo(context);
   }

   // Add boot specific singleton beans
   ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
   //bean, springApplicationArguments,用于获取启动application所需的参数
   beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    
   //加载打印Banner的Bean
   if (printedBanner != null) {
      beanFactory.registerSingleton("springBootBanner", printedBanner);
   }
   
   if (beanFactory instanceof DefaultListableBeanFactory) {
      ((DefaultListableBeanFactory) beanFactory)
            .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
   }
   // Load the sources，根据primarySources加载resource。
   // primarySources:一般为主类的class对象
   Set<Object> sources = getAllSources();
   Assert.notEmpty(sources, "Sources must not be empty");
   //构造BeanDefinitionLoader并完成定义的Bean的加载
   load(context, sources.toArray(new Object[0]));
   //发布ApplicationPreparedEvent事件，表示application已准备完成
   listeners.contextLoaded(context);
}
```

## **6.刷新容器**

```java
private void refreshContext(ConfigurableApplicationContext context) {
   refresh(context);
   // 注册一个关闭容器时的钩子函数,在jvm关闭时调用
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
	Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
	((AbstractApplicationContext) applicationContext).refresh();
}
```

调用父类AbstractApplicationContext刷新容器的操作

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
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
       ......
}
```

## **7.后置操作，调用Runners**

后置操作，就是在容器完成刷新后，依次调用注册的Runners，还可以通过@Order注解设置各runner的执行顺序。

Runner可以通过实现ApplicationRunner或者CommandLineRunner接口。

```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {
        List<Object> runners = new ArrayList<>();
        runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
        runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
        AnnotationAwareOrderComparator.sort(runners);
        for (Object runner : new LinkedHashSet<>(runners)) {
            if (runner instanceof ApplicationRunner) {
                callRunner((ApplicationRunner) runner, args);
            }
            if (runner instanceof CommandLineRunner) {
                callRunner((CommandLineRunner) runner, args);
            }
        }
    }
```

根据源码可知，runners收集从容器获取的ApplicationRunner和CommandLineRunner类型的Bean，然后依次执行。

## **8.发布ApplicationReadyEvent事件**

```java
listeners.running(context);
```

应用启动完成，可以对外提供服务了，在这里发布ApplicationReadyEvent事件。流程还是和starting时一样。
