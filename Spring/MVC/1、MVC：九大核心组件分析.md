SpringMVC相对于前面的IOC、DI、AOP是比较简单的，Spring MVC的核心组件和大致处理流程如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201209151745493.png?)
1. DispatcherServlet是SpringMVC中的前端控制器(FrontController),负责接收Request并将Request转发给对应的处理组件。
2. HanlerMapping 是 SpringMVC 中 完 成 url 到 Controller 映 射 的 组 件 。DispatcherServlet 接收 Request,然后从 HandlerMapping 查找处理 Request 的Controller。
3. Controller 处理 Request,并返回 ModelAndView 对象,Controller 是 SpringMVC中负责处理 Request 的组件(类似于 Struts2 中的 Action),ModelAndView 是封装结果视图的组件。
4. 4-6 视图解析器解析ModelAndView对象并返回对应的视图给客户端。在前面的文章中我们已经大致了解到，容器初始化时会建立所有url 和 Controller 中的 Method 的对应关系，保存到 HandlerMapping 中，用户请求是根据 Request 请求的url快速定位到Controller 中的某个方法。


>在Spring中先将url和Controller的对应关系保存到 Map<url,Controller>中，Web 容器启动时会通知 Spring 初始化容器(加载Bean的定义信息和初始化所有单例Bean)，然后SpringMVC会遍历容器中的Bean，获取每一个Controller中的所有方法访问的url，然后将url和Controller保存到一个Map中。这样就可以根据 Request 快速定位到 Controller。
><br>因为最终处理 Request 的是 Controller 中的方法，Map 中只保留了 url 和 Controller 中的对应关系，所以要根据Request 的 url 进一步确认 Controller 中的 Method，这一步工作的原理就是拼接Controller 的 url(Controller 上@RequestMapping 的值)和方法的 url(Method 上@RequestMapping的值)，与request的url进行匹配，找到匹配的那个方法。
><br>确定处理请求的Method后，接下来的任务就是参数绑定，把Request中参数绑定到方法的形式参数上，这一步是整个请求处理过程中最复杂的一个步骤。



### 1.HandlerMappings *
HandlerMapping是用来查找Handler的，也就是处理器，具体的表现形式可以是类也可以是方法。比如，标注了@RequestMapping 的每个 method 都可以看成是一个Handler，由 Handler 来负责实际的请求处理。 

HandlerMapping 在请求到达之后，它的作用便是找到请求相应的处理器Handler和Interceptors。 

### 2.HandlerAdapters *
从名字上看，这是一个适配器。因为SpringMVC中Handler可以是任意形式的，只要能够处理请求便行。但是把请求交给 Servlet 的时候，由于 Servlet 的方法结构都是如 doService(HttpServletRequestreq,HttpServletResponseresp) 这样的形式，让固定的Servlet 处理方法调用 Handler 来进行处理，这一步工作便是HandlerAdapter 要做的事。 

### 3.HandlerExceptionResolvers 
从这个组件的名字上看，这个就是用来处理Handler过程中产生的异常情况的组件。 具体来说，此组件的作用是根据异常设 ModelAndView, 之后再交给 render()方法进行渲 染 ， 而 render() 便将 ModelAndView 渲染成页面 。 

不过有一点 ，HandlerExceptionResolver 只是用于解析对请求做处理阶段产生的异常，而渲染阶段的异常则不归他管了，这也是Spring MVC 组件设计的一大原则分工明确互不干涉。 

### 4.ViewResolvers*
视图解析器，相信大家对这个应该都很熟悉了。因为通常在SpringMVC的配置文件中，都会配上一个该接口的实现类来进行视图的解析。 这个组件的主要作用，便是将String类型的视图名和Locale解析为View类型的视图。

这个接口只有一个resolveViewName()方法。从方法的定义就可以看出，Controller层返回的String类型的视图名viewName，最终会在这里被解析成为View。

View 是用来渲染页面的，也就是说，它会将程序返回的参数和数据填入模板中，最终生成 html 文件。ViewResolver在这个过程中，主要做两件大事，即，ViewResolver会找到渲染所用的模板（使用什么模板来渲染？）和所用的技术（其实也就是视图的类型，如JSP啊还是其他什么Blabla的）填入参数。默认情况下，SpringMVC会为我们自动配置一个InternalResourceViewResolver，这个是针对JSP类型视图的。 

### 5.RequestToViewNameTranslator 

这个组件的作用，在于从 Request 中获取 viewName. 因为 ViewResolver 是根据ViewName 查找 View, 但有的 Handler 处理完成之后，没有设置 View 也没有设置ViewName， 便要通过这个组件来从Request中查找viewName。 

### 6.LocaleResolver 

在上面我们有看到 ViewResolver 的resolveViewName()方法，需要两个参数。那么第二个参数Locale是从哪来的呢，这就是LocaleResolver 要做的事了。 LocaleResolver用于从 request 中解析出 Locale, 在中国大陆地区，Locale当然就会是zh-CN之类，用来表示一个区域。这个类也是i18n的基础。

### 7.ThemeResolver 

从名字便可看出，这个类是用来解析主题的。主题，就是样式，图片以及它们所形成的显示效果的集合。Spring MVC中一套主题对应一个properties文件，里面存放着跟当前主题相关的所有资源，如图片，css样式等。

>创建主题非常简单，只需准备好资源，然后新建一个 "主题名.properties" 并将资源设置进去，放在classpath下，便可以在页面中使用了。 

Spring MVC 中跟主题有关的类有 ThemeResolver, ThemeSource 和Theme。 ThemeResolver 负责从request中解析出主题名，ThemeSource则根据主题名找到具体的主题， 其抽象也就是 Theme, 通过Theme来获取主题和具体的资源。 

### 8.MultipartResolver

其实这是一个大家很熟悉的组件，MultipartResolver用于处理上传请求，通过将普通的Request包装成MultipartHttpServletRequest来实现。

MultipartHttpServletRequest可以通过getFile() 直接获得文件，如果是多个文件上传，还可以通过调用getFileMap得到Map<FileName,File> 这样的结构。

>MultipartResolver的作用就是用来封装普通的request，使其拥有处理文件上传的功能。

### 9.FlashMapManager 

说到FlashMapManager，就得先提一下FlashMap。FlashMap用于重定向Redirect时的参数数据传递，比如，在处理用户订单提交时，为了避免重复提交，可以处理完post请求后redirect到一个get请求，这个get请求可以用来显示订单详情之类的信息。

这样做虽然可以规避用户刷新重新提交表单的问题，但是在这个页面上要显示订单的信息，那这些数据从哪里去获取呢，因为redirect重定向是没有传递参数这一功能的，如果不想把参数写进url（其实也不推荐这么做，url有长度限制不说，把参数都直接暴露，感觉也不安全)， 那么就可以通过flashMap来传递。

只需要在 redirect 之前 ， 将要传递的数据写入 request （ 可以通过 ServletRequestAttributes.getRequest() 获得）的属性`OUTPUT_FLASH_MAP_ATTRIBUTE` 中，这样在 redirect 之后的 handler 中 Spring 就会自动将其设置到Model中，在显示订单信息的页面上，就可以直接从Model 中取得数据了。而FlashMapManager就是用来管理FlashMap的。



