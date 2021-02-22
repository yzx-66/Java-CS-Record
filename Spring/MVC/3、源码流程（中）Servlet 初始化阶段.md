## 流程
下面进入正文，我们的思路是首先找到 DispatcherServlet 这个类，然后寻找init()方法。我们发现其 init 方法其实在父类 HttpServletBean中，源码如下： 
### init()

```java
@Override
public final void init() throws ServletException {
   if (logger.isDebugEnabled()) {
      logger.debug("Initializing servlet '" + getServletName() + "'");
   }

   // Set bean properties from init parameters.
   PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
   if (!pvs.isEmpty()) {
      try {
         // 定位资源
         BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
         // 加载配置信息
         ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
         bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
         initBeanWrapper(bw);
         bw.setPropertyValues(pvs, true);
      }
      catch (BeansException ex) {
         if (logger.isErrorEnabled()) {
            logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
         }
         throw ex;
      }
   }

   // Let subclasses do whatever initialization they like.
   // 让子类进行初始化
   initServletBean();

   if (logger.isDebugEnabled()) {
      logger.debug("Servlet '" + getServletName() + "' configured successfully");
   }
}
```

我们看到在这段代码中，又调用了一个重要的 initServletBean() 方法。进入 initServletBean() 方法看到以下源码： 

### initServletBean()

```java
@Override
protected final void initServletBean() throws ServletException {
   getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
   if (this.logger.isInfoEnabled()) {
      this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
   }
   long startTime = System.currentTimeMillis();

   try {
	  // 初始化IOC容器
      this.webApplicationContext = initWebApplicationContext();
      initFrameworkServlet();
   }
   catch (ServletException ex) {
      this.logger.error("Context initialization failed", ex);
      throw ex;
   }
   catch (RuntimeException ex) {
      this.logger.error("Context initialization failed", ex);
      throw ex;
   }

   if (this.logger.isInfoEnabled()) {
      long elapsedTime = System.currentTimeMillis() - startTime;
      this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
            elapsedTime + " ms");
   }
}
```
### initWebApplicationContext()
IOC 容器初始化之后，最后又调用了 onRefresh() 方法。这个方法最终是在 DisptcherServlet 中实现，来看源码： 
```java
protected WebApplicationContext initWebApplicationContext() {

   // 先从ServletContext中获得父容器 WebAppliationContext
   WebApplicationContext rootContext =
         WebApplicationContextUtils.getWebApplicationContext(getServletContext());
   // 声明子容器
   WebApplicationContext wac = null;

   // 建立父、子容器之间的关联关系
   if (this.webApplicationContext != null) {
      // A context instance was injected at construction time -> use it
      wac = this.webApplicationContext;
      if (wac instanceof ConfigurableWebApplicationContext) {
         ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
         if (!cwac.isActive()) {
            // The context has not yet been refreshed -> provide services such as
            // setting the parent context, setting the application context id, etc
            // 上下文尚未刷新，提供诸如设置父上下文，设置应用上下文，id等
            if (cwac.getParent() == null) {
               // The context instance was injected without an explicit parent -> set
               // the root application context (if any; may be null) as the parent
               cwac.setParent(rootContext);
            }
            // 这个方法里面调用了AbatractApplication的 refresh()模板方法，规定IOC初始化基本流程
            configureAndRefreshWebApplicationContext(cwac);
         }
      }
   }
   // 先去ServletContext中查找Web容器的引用是否存在，并创建好默认的空IOC容器
   if (wac == null) {
      // No context instance was injected at construction time -> see if one
      // has been registered in the servlet context. If one exists, it is assumed
      // that the parent context (if any) has already been set and that the
      // user has performed any initialization such as setting the context id
      wac = findWebApplicationContext();
   }
   // 给上一步创建好的IOC容器赋值
   if (wac == null) {
      // No context instance is defined for this servlet -> create a local one
      wac = createWebApplicationContext(rootContext);
   }

   // 触发onRefresh方法
   if (!this.refreshEventReceived) {
      // Either the context is not a ConfigurableApplicationContext with refresh
      // support or the context injected at construction time had already been
      // refreshed -> trigger initial onRefresh manually here.
      onRefresh(wac);
   }

   if (this.publishContext) {
      // Publish the context as a servlet context attribute.
      String attrName = getServletContextAttributeName();
      getServletContext().setAttribute(attrName, wac);
      if (this.logger.isDebugEnabled()) {
         this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
               "' as ServletContext attribute with name [" + attrName + "]");
      }
   }

   return wac;
}
```
这段代码中最主要的逻辑就是初始化IOC容器，最终会调用 refresh() 方法。

> 关于IOC容器的初始化细节可以参考：
> * [【Spring】IOC：初始化流程源码分析（上）定位 Resource](https://blog.csdn.net/weixin_43935927/article/details/110489146)
> * [【Spring】IOC：初始化流程源码分析（中）加载 BeanDefinition](https://blog.csdn.net/weixin_43935927/article/details/110492100)
> * [【Spring】IOC：初始化流程源码分析（下）注册 BeanDefinition](https://blog.csdn.net/weixin_43935927/article/details/110492145)
>

另外，这里还又涉及了 Spring 容器与 SpringMVC 容器的关系：
* 相同：Spring 容器与 SpringMVC 容器都是 IOC 容器实例，都是走 refresh 流程创建的
* 不同：Spring 容器是 SpringMVC 容器的父容器（`cwac.setParent(parent)`），会先于 SpringMVC 容器创建。所以 SpringMVC 容器能获取到 Spring 容器的 bean，而 Spring 容器获取不到 SpringMVC 容器的 bean
* 总结：我们在平时开发中，一般会使用两个配置文件去明确这两个容器注册不同的 bean
	* applicationContext.xml：配置 Spring 容器，去管理那些通用 bean（service，dao等）
	* applicationContext-mvc.xml：配置 SpringMVC 容器，专门管理 controller 


再看一次 web.xml 的配置：
```xml
<!-- 1.启动spring容器 -->
<context-param>
   <param-name>contextConfigLocation</param-name>
   <param-value>classpath:applicationContext.xml</param-value>
</context-param>

<listener>
   <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>

<!-- 2.springmvc前端控制器 ， 拦截所有请求 -->
<servlet>
   <servlet-name>dispatcherServlet</servlet-name>
   <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
   <init-param>
       <param-name>contextConfigLocation</param-name>
       <param-value>classpath:applicationContext-mvc.xml</param-value>
   </init-param>
   <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
   <servlet-name>dispatcherServlet</servlet-name>
   <url-pattern>/</url-pattern>
</servlet-mapping>
```
> 关于 Spring 容器与 SpringMVC 容器的关系再放几个参考链接， [参考链接1](https://juejin.cn/post/6844903537416077320)，[参考链接2](https://blog.csdn.net/justloveyou_/article/details/74295728?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.control)，[参考链接3](https://www.cnblogs.com/brolanda/p/4265597.html)，[参考链接4](https://blog.csdn.net/u012060033/article/details/105670409)...
### onRefresh()

```java
@Override
protected void onRefresh(ApplicationContext context) {
   initStrategies(context);
}

// 初始化策略
protected void initStrategies(ApplicationContext context) {
   // 多文件上传的组件
   initMultipartResolver(context);
   // 初始化本地语言环境
   initLocaleResolver(context);
   // 初始化模板处理器
   initThemeResolver(context);
   // handlerMapping
   initHandlerMappings(context);
   // 初始化参数适配器
   initHandlerAdapters(context);
   // 初始化异常拦截器
   initHandlerExceptionResolvers(context);
   // 初始化视图预处理器
   initRequestToViewNameTranslator(context);
   // 初始化视图转换器
   initViewResolvers(context);
   //
   initFlashMapManager(context);
}
```

到这一步就完成了SpringMVC的九大组件的初始化。 

## 总结

这里给出一个简洁的文字描述版`SpringMVC启动过程`:

tomcat web容器启动时会去读取`web.xml`这样的`部署描述文件`，相关组件启动顺序为: `解析<context-param>` => `解析<listener>` => `解析<filter>` => `解析<servlet>`，具体初始化过程如下:

- 1、解析`<context-param>`里的键值对。

- 2、创建一个`application`内置对象即`ServletContext`，servlet上下文，用于全局共享。

- 3、将`<context-param>`的键值对放入`ServletContext`即`application`中，`Web应用`内全局共享。

- 4、读取`<listener>`标签创建监听器，一般会使用`ContextLoaderListener类`，如果使用了`ContextLoaderListener类`，`Spring`就会创建一个`WebApplicationContext类`的对象，`WebApplicationContext类`就是`IoC容器`，`ContextLoaderListener类`创建的`IoC容器`是`根IoC容器`为全局性的，并将其放置在`appication`中，作为应用内全局共享，键名为`WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`，可以通过以下两种方法获取

  

  ```undefined
  WebApplicationContext applicationContext = (WebApplicationContext) application.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
  
  WebApplicationContext applicationContext1 = WebApplicationContextUtils.getWebApplicationContext(application);
  ```

	这个全局的`根IoC容器`只能获取到在该容器中创建的`Bean`不能访问到其他容器创建的`Bean`，也就是读取`web.xml`配置的`contextConfigLocation`参数的`xml文件`来创建对应的`Bean`。

- 5、`listener`创建完成后如果 web.xml 有 `<filter>`则会去创建`filter`。
- 6、初始化创建`<servlet>`，一般使用`DispatchServlet类`。
- 7、`DispatchServlet`的父类`FrameworkServlet`会重写其父类的`initServletBean`方法，并调用`initWebApplicationContext()`以及`onRefresh()`方法。
- 8、`initWebApplicationContext()`方法会创建一个当前`servlet`的一个`IoC子容器`，如果存在上述的全局`WebApplicationContext`则将其设置为`父容器`，如果不存在上述全局的则`父容器`为null。
- 9、读取`<servlet>`标签的`<init-param>`配置的`xml文件`并加载相关`Bean`。
- 10、`onRefresh()`方法创建`Web应用`相关组件。

