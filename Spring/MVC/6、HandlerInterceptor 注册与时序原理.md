# 前言
## HandlerInterceptor 定义

```java
public interface HandlerInterceptor {

    /**
     * 预处理回调方法，实现处理器的预处理，第三个参数为响应的处理器，自定义Controller
     * 返回值：true表示继续流程（如调用下一个拦截器或处理器）；false表示流程中断
     * 不会继续调用其他的拦截器或处理器，此时我们需要通过response来产生响应
     */
    default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {

        return true;
    }

    /**
     * 后处理回调方法，实现处理器的后处理（渲染视图之前）
     * 此时我们可以通过modelAndView（模型和视图对象）对模型数据进行处理或对视图进行处理
     * modelAndView也可能为null，如API接口返回JSON数据时
     */
    default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
            @Nullable ModelAndView modelAndView) throws Exception {
    }

    /**
     * 整个请求处理完毕回调方法，即在视图渲染完毕时回调
     * 如性能监控中我们可以在此记录结束时间并输出消耗时间
     * 还可以进行一些资源清理，类似于try-catch-finally中的finally，但仅调用处理器执行链中
     */
    default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
            @Nullable Exception ex) throws Exception {
    }

}
```
## 重要接口及类
AbstractHandlerMapping
* 实际上拦截器都是初始化并保存在 AbstractHandlerMapping 里，直到 getHandle -> getHandleExcuteChain 时会使用 HandlerMapping 里面的拦截器去初始化 HandlerExcuteChain

<img src="https://img-blog.csdnimg.cn/20210222190731106.png?"  width="65%"/>

HandlerExecutionChain
* handlerMethod 和 handlerExcuteChain 的包装类

<img src="https://img-blog.csdnimg.cn/20210222190929577.png?"  width="65%"/>

 MappedInterceptor
 * 一个包括includePatterns和excludePatterns字符串集合并带有HandlerInterceptor的类。 很明显，就是对于某些地址做特殊包括和排除的拦截器。

<img src="https://img-blog.csdnimg.cn/20210222191229580.png"  width="45%"/>

# HandlerInterceptor 解析与初始化

## 解析
SpringMVC 配置拦截器

```
<mvc:interceptors>
  <mvc:interceptor>
    <mvc:mapping path="/**"/>
    <mvc:exclude-mapping path="/login"/> 　　
    <mvc:exclude-mapping path="/index"/>
    <bean class="package.interceptor.XXInterceptor"/>
  </mvc:interceptor>
</mvc:interceptors>
```

这里配置的每个\<mvc:interceptor>都会被解析成MappedInterceptor。

* 其中子标签\<mvc:mapping path="/**"/>会被解析成MappedInterceptor的includePatterns属性；
* <mvc:exclude-mapping path="/**"/>会被解析成MappedInterceptor的excludePatterns属性；
* <bean/>会被解析成MappedInterceptor的interceptor属性。

\<mvc:interceptors>这个标签是被InterceptorsBeanDefinitionParser类解析。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210222191645184.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)


## 初始化
interceptor的初始化是在初始化 HandlerMapping 时完成的，HandlerMapping 多数是通过继承 AbstractHandlerMapping 实现的
* 因为 AbstractHandlerMapping 实现 **ApplicationContextAware** 接口，bean 初始化完成后会执行 **setApplicationContext 方法**，通过源码看到最终执行如下方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210222174716869.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)

```java
protected void initApplicationContext() throws BeansException {
	//空实现，用于扩展拦截器
    extendInterceptors(this.interceptors);
    //直接加载spring 容器中所有实现了MappedInterceptor接口的拦截器
    detectMappedInterceptors(this.adaptedInterceptors);
    //把extendInterceptors中的扩展方法加入到adaptedInterceptors集合中
    initInterceptors();
}
```

```java
protected void detectMappedInterceptors(List<HandlerInterceptor> mappedInterceptors) {
 	// 加载所有的 MappedInterceptor
    mappedInterceptors.addAll(
            BeanFactoryUtils.beansOfTypeIncludingAncestors(
                    obtainApplicationContext(), MappedInterceptor.class, true, false).values());
}
```
```java
protected void initInterceptors() {
    if (!this.interceptors.isEmpty()) {
        for (int i = 0; i < this.interceptors.size(); i++) {
            Object interceptor = this.interceptors.get(i);
            if (interceptor == null) {
                throw new IllegalArgumentException("Entry number " + i + " in interceptors array is null");
            }
            //List<MappedInterceptor>
            if (interceptor instanceof MappedInterceptor) {
                this.mappedInterceptors.add((MappedInterceptor) interceptor);
            }
            //添加到List<HandlerInterceptor>
            else {
                this.adaptedInterceptors.add(adaptInterceptor(interceptor));
            }
        }
    }
}
```

注意：
* Aware 接口的回调在 InitlizeBean 会调用 afterPropertySet 前面，所以初始化拦截在初始化 handlerMap 前面。


#### 拦截器时序原理

当请求进来后，最终执行DispatcherServlet的doDispatch方法开始处理请求，如下

```java
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        HttpServletRequest processedRequest = request;
        HandlerExecutionChain mappedHandler = null;
        boolean multipartRequestParsed = false;

        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

        try {
            ModelAndView mv = null;
            Exception dispatchException = null;

            try {
                processedRequest = checkMultipart(request);
                multipartRequestParsed = (processedRequest != request);
				
				// 初始化 HandlerExecutionChain 与添加拦截器
                // 此处开始遍历HandlerMapping，获取HandlerExecutionChain，这里方法执行完后，针对此次请求，已经确定有几个拦截器可以被执行了
                mappedHandler = getHandler(processedRequest);
                
                HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

                //开始这行preHandler的预处理方法，会遍历HandlerExecutionChain 中的所有拦截器，并执行器preHandler方法，如果有方法返回false，则直接返回，不在继续执行
                if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                    return;
                }

                //开始真正执行Handler方法
                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

                

                applyDefaultViewName(processedRequest, mv);
				//具体Controler中的方法执行完成后开始执行拦截器的postHandler方法，注意此时是和preHandler的执行顺序是相反的
                mappedHandler.applyPostHandle(processedRequest, response, mv);
            }
            //开始遍历并执行拦截器的AfterCompletion方法，注意此时是根据handlerIndex反向执行的
            processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
        }
        catch (Exception ex) {
            triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
        }
        catch (Throwable err) {
            triggerAfterCompletion(processedRequest, response, mappedHandler,
                    new NestedServletException("Handler processing failed", err));
        }
        finally {
            if (asyncManager.isConcurrentHandlingStarted()) {
                // Instead of postHandle and afterCompletion
                if (mappedHandler != null) {
                    mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
                }
            }
            else {
                // Clean up any resources used by a multipart request.
                if (multipartRequestParsed) {
                    cleanupMultipart(processedRequest);
                }
            }
        }
    }
```

AbstractHandlerMapping中获取HandlerExecutionChain的过程如下
* 请求进来后，在HandlerMapping中会根据具体的请求path来选择合适的拦截器
* 因为在初始化HandlerMapping的时候，已经把所有的HandlerInterceptor全部加载完成了

```java
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
        HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
                (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

        String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
        for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
            if (interceptor instanceof MappedInterceptor) {
                MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
                // 如果匹配上就放入 HandlerExecutionChain  的 interceptors 中
                if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
                    chain.addInterceptor(mappedInterceptor.getInterceptor());
                }
            }
            else {
                chain.addInterceptor(interceptor);
            }
        }
        return chain;
}
```

#### 拦截器和Filter的区别

1. Filter是web容器定义的，由servlet容器来加载并执行，基本可以拦截所有请求，interceptor是spring mvc这种web框架定义的，用于在handler执行前后执行的，仅针对handler进行拦截，且是spring 来加载并执行的
2. 针对执行顺序，自然是filter先被执行，然后是拦截器的执行，拦截器是按照顺序执行prehandler，然后按照相反的顺序执行postHandler的，且不论方法执行成功与否，会按照相反的和preHander相反的顺序执行afterCompletion【仅执行已经执行过preHander方法的拦截器】

