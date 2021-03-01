## 1.内嵌Tomcat--jar包启动原理
内嵌 tomcat 的启动流程大致如下：

1. org.springframework.boot.SpringApplication#refreshContext
2. org.springframework.context.support.AbstractApplicationContext#refresh
3. org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#onRefresh
4. org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#createWebServer
5. org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory#getWebServer

### refreshContext()
这个方法会在 SpringBoot 启动的 run 方法中调用，是初始化 IOC 容器的入口
```java
private void refreshContext(ConfigurableApplicationContext context) {
   // 调用 AbstractApplication#refresh 方法，去初始化IOC容器
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
```

### refresh()

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      // 1.调用容器准备刷新的方法，获取容器的当时时间，同时给容器设置同步标识
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      // 2.告诉子类启动refreshBeanFactory()方法，Bean定义资源文件的载入从子类的refreshBeanFactory()方法启动
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
      // 3.为BeanFactory配置容器特性，例如类加载器、事件处理器等
      prepareBeanFactory(beanFactory);

      try {
         // Allows post-processing of the bean factory in context subclasses.
         // 4.为容器的某些子类指定特殊的BeanPost事件处理器
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         // 5.调用所有注册的BeanFactoryPostProcessor的Bean
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         // 6.为BeanFactory注册BeanPost事件处理器.BeanPostProcessor是Bean后置处理器，用于监听容器触发的事件
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         // 7.初始化信息源，和国际化相关.
         initMessageSource();

         // Initialize event multicaster for this context.
         // 8.初始化容器事件传播器.
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         // 9.调用子类的某些特殊Bean初始化方法
         onRefresh();

         // Check for listener beans and register them.
         // 10.为事件传播器注册事件监听器.
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         // 11.初始化所有剩余的单例Bean
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         // 12.初始化容器的生命周期事件处理器，并发布容器的生命周期事件
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         // 13.销毁已创建的Bean
         destroyBeans();

         // Reset 'active' flag.
         // 14.取消refresh操作，重置容器的同步标识。
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         // 15.重设公共缓存
         resetCommonCaches();
      }
   }
}
```

这里我们继续看 onRefresh 方法

### onRefresh()

```java
protected void onRefresh() throws BeansException {
   // For subclasses: do nothing by default. 
   // 给子类去实现
}
```

下面，我们打开实现类 ServletWebServerApplicationContext 的 onRefresh 方法

```java
protected void onRefresh() {
   super.onRefresh();
   try {
     // 创建嵌入式Servlet服务器
     // 注：到这里时已经创建好了SpringBoot应用上下文
      createWebServer();
   }
   catch (Throwable ex) {
      throw new ApplicationContextException("Unable to start web server", ex);
   }
}
```

### createWebServer()

```java
private void createWebServer() {
   // 获取当前的WebServer
   WebServer webServer = this.webServer;
   // 获取当前的ServletContext
   ServletContext servletContext = getServletContext();
   // 第一次进来，webServer和servletContext 默认都为null，会进入这里
   if (webServer == null && servletContext == null) {
       // 获取Servlet服务器工厂
      StartupStep createWebServer = this.getApplicationStartup().start("spring.boot.webserver.create");
      // 工厂方法，获取Servlet服务器，并作为AbstractApplicationContext的一个属性进行设置。
      // 该会创建DispatcherServlet对象，并添加到beanFactory中去，对应的beanName为dispatcherServlet
      ServletWebServerFactory factory = getWebServerFactory();
      createWebServer.tag("factory", factory.getClass().toString());
      // 这个方法为wrapper设置了servletClass为DispatcherServlet
      this.webServer = factory.getWebServer(getSelfInitializer());
      createWebServer.end();
      getBeanFactory().registerSingleton("webServerGracefulShutdown",
            new WebServerGracefulShutdownLifecycle(this.webServer));
      getBeanFactory().registerSingleton("webServerStartStop",
            new WebServerStartStopLifecycle(this, this.webServer));
   }
   else if (servletContext != null) {
      try {
         getSelfInitializer().onStartup(servletContext);
      }
      catch (ServletException ex) {
         throw new ApplicationContextException("Cannot initialize servlet context", ex);
      }
   }
   // 初始化一些ConfigurableEnvironment中的 ServletContext信息
   initPropertySources();
}
```

### getWebServer()

获取 webServer 其实有多种选择，SpringBoot 不止内嵌了 Tocmat，还内嵌了 Jetty 等。
<img src="https://img-blog.csdnimg.cn/20201211010758134.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNTkyNw==,size_16,color_FFFFFF,t_70#pic_center" width="70%">


这里我们只看内嵌 tomcat，源码如下：

```java
public WebServer getWebServer(ServletContextInitializer... initializers) {
    // 创建内嵌tomcat，直接new出来的
    Tomcat tomcat = new Tomcat();
    // 设置工作目录
    File baseDir = (this.baseDirectory != null) ? this.baseDirectory
        : createTempDir("tomcat");
    // 设置安装目录
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    // 初始化tomcat的连接器
    Connector connector = new Connector(this.protocol);
    tomcat.getService().addConnector(connector);
    customizeConnector(connector);
    tomcat.setConnector(connector);
    // 设置自动部署为false
    tomcat.getHost().setAutoDeploy(false);
    // 配置引擎
    configureEngine(tomcat.getEngine());
    for (Connector additionalConnector : this.additionalTomcatConnectors) {
        tomcat.getService().addConnector(additionalConnector);
    }
    // 准备context
    prepareContext(tomcat.getHost(), initializers);
    return getTomcatWebServer(tomcat);
}


protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {  
    // 端口大于0启动启动  
    return new TomcatWebServer(tomcat, getPort() >= 0); 
}   
```

```java
public TomcatWebServer(Tomcat tomcat, boolean autoStart) {
    Assert.notNull(tomcat, "Tomcat Server must not be null");
    // 维护了一个tomcat的实例
    this.tomcat = tomcat;
    this.autoStart = autoStart;
    // 初始化方法，启动tomcat
    initialize();
}

// 初始化方法，启动tomcat
private void initialize() throws WebServerException {
    logger.info("Tomcat initialized with port(s): " + getPortsDescription(false));
    synchronized (this.monitor) {
        try {
            addInstanceIdToEngineName();

            Context context = findContext();
            context.addLifecycleListener((event) -> {
                if (context.equals(event.getSource())
                    && Lifecycle.START_EVENT.equals(event.getType())) {
                    // Remove service connectors so that protocol binding doesn't
                    // happen when the service is started.
                    removeServiceConnectors();
                }
            });

            // Start the server to trigger initialization listeners
            // 启动tomcat
            // 这里面会为Wrapper设置servletClass为dispatcherServlet
            this.tomcat.start();

            // We can re-throw failure exception directly in the main thread
            rethrowDeferredStartupExceptions();

            try {
                ContextBindings.bindClassLoader(context, context.getNamingToken(),
                                                getClass().getClassLoader());
            }
            catch (NamingException ex) {
                // Naming is not enabled. Continue
            }

            // Unlike Jetty, all Tomcat threads are daemon threads. We create a
            // blocking non-daemon to stop immediate shutdown
            // 启动一个守护进程进行等待，以免程序直接停止结束
            startDaemonAwaitThread();
        }
        catch (Exception ex) {
            stopSilently();
            throw new WebServerException("Unable to start embedded Tomcat", ex);
        }
    }
}
```
可以看到，内嵌 Tomcat 已经 start 了。

那么，我们把 SpringBoot 的程序打成 war 的时候，是怎么样的原理了？(tomcat 启动带动 IOC 容器的启动)

