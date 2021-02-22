# 一、请求的分发过程

  首先简单描述一下，请求的分发过程。一个请求到来，会走到DispatcherServlet的doDispatch方法。这个方法非常重要，封装了整个请求的分发过程，其中有一段代码如下：


```java
//根据请求找到对应的handler       
mappedHandler = getHandler(processedRequest, false);
if (mappedHandler == null || mappedHandler.getHandler() == null) {
   noHandlerFound(processedRequest, response);
   return;
}

// Determine handler adapter for the current request.
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
......  调用拦截器等   ......
// Actually invoke the handler.
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

 `mappedHandler = getHandler(processedRequest, false)` 这个方法根据request得到的是一个HandlerExecutionChain对象，他包含了mvc模块的拦截器即handlerInterceptor和真正处理请求的handler。这个方法最终调用的是下面的这个方法，它也在ispatcherServlet中：

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   for (HandlerMapping hm : this.handlerMappings) {
      if (logger.isTraceEnabled()) {
         logger.trace(
               "Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
      }
      HandlerExecutionChain handler = hm.getHandler(request);
      if (handler != null) {
         return handler;
      }
   }
   return null;
}
```



  终于今天要详细说明的主角来了，我们看到 HandlerExecutionChain 是从 handlerMappings中取得的，但是 handlerMappings是怎么来的呢？

# 二、DispatcherServlet 中 handlerMapping的初始化

## initHandlerMappings
在DispatcherServlet类下有这样一个方法


```java
private void initHandlerMappings(ApplicationContext context) {
   this.handlerMappings = null;

   if (this.detectAllHandlerMappings) {
      // Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
      Map<String, HandlerMapping> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
      if (!matchingBeans.isEmpty()) {
         this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
         // We keep HandlerMappings in sorted order.
         OrderComparator.sort(this.handlerMappings);
      }
   }
   else {
      try {
         HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
         this.handlerMappings = Collections.singletonList(hm);
      }
      catch (NoSuchBeanDefinitionException ex) {
         // Ignore, we'll add a default HandlerMapping later.
      }
   }

   // Ensure we have at least one HandlerMapping, by registering
   // a default HandlerMapping if no other mappings are found.
   if (this.handlerMappings == null) {
      this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
      if (logger.isDebugEnabled()) {
         logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
      }
   }
}
```

根据方法的名字可以猜出，它就是初始化handermapping的地方。而这个方法的逻辑也不是很复杂，就是向容器索要HandlerMapping的实现类。 可知这个时候handermapper的bean的定义已经在容器中了。但是，这段初始化的代码是怎么调起的呢？


# RequestMappingHandlerMapping
* <mvc:annotation-driven /> 默认注入的 handlerMapping



## handleMapping在容器中的初始化过程

  我们在spring的配置文件中通常会加入这样一个标签 <mvc:annotation-driven />，用来支持基于注解的映射。
  * 而它的实现类是AnnotationDrivenBeanDefinitionParser,这个类又向容器中注册了RequestMappingHandlerMapping，他就是一个handlerMapping，dispatchServelet中用到的handlerMapping。

AnnotationDrivenBeanDefinitionParser的parse方法中的代码片段：

```java
RootBeanDefinition methodMappingDef = new RootBeanDefinition(RequestMappingHandlerMapping.class);
methodMappingDef.setSource(source);
methodMappingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
methodMappingDef.getPropertyValues().add("order", 0);
String methodMappingName = parserContext.getReaderContext().registerWithGeneratedName(methodMappingDef);
```

  RequestMappingHandlerMapping的继承关系如下 
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210222174716869.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70) 


## 回调 afterPropertiesSet 给 HandlerMapping 注册 Handler
afterPropertiesSet(),这就是HandlerMapping真正初始化的触发点（具体参见IOC实现源码）:

```java
public void afterPropertiesSet() {
   initHandlerMethods();
}
protected void initHandlerMethods() {   if (logger.isDebugEnabled()) {
      logger.debug("Looking for request mappings in application context: " + getApplicationContext());
   }

   String[] beanNames = (this.detectHandlerMethodsInAncestorContexts ?
         BeanFactoryUtils.beanNamesForTypeIncludingAncestors(getApplicationContext(), Object.class) :
         getApplicationContext().getBeanNamesForType(Object.class));

   for (String beanName : beanNames) {
      if (isHandler(getApplicationContext().getType(beanName))){
         detectHandlerMethods(beanName);
      }
   }
   handlerMethodsInitialized(getHandlerMethods());
}
```

​上面的代码就是处理controller中的各个方法的逻辑。首先得到所有的handler，对应开发者写的controller；

然后查找每个handler中映射请求的方法；最后初始化这些映射方法。接下来看看是怎么查找并处理这些映射方法的。
```java
​protected void detectHandlerMethods(final Object handler) { 	
	Class<?> handlerType = (handler instanceof String) ? 			
			getApplicationContext().getType((String) handler) : handler.getClass(); 	
				
	final Class<?> userType = ClassUtils.getUserClass(handlerType); 	
		
	Set<Method> methods = HandlerMethodSelector.selectMethods(userType, new MethodFilter() { 		
					public boolean matches(Method method) { 			
						return getMappingForMethod(method, userType) != null; 		
					} 	
	}); 		
	
	for (Method method : methods) { 		//对于每个方法，通过注解等信息解析出映射信息，然后进行注册
		T mapping = getMappingForMethod(method, userType);
		registerHandlerMethod(handler, method, mapping);
	}
}
```

把 beanName 、method 、 RequestMappingInfo 注册进去

```java
protected void registerHandlerMethod(Object handler, Method method, T mapping) {   
	HandlerMethod handlerMethod;   
	if (handler instanceof String) {      
		String beanName = (String) handler;      
		handlerMethod = new HandlerMethod(beanName, getApplicationContext(), method);   
	} else {      
		handlerMethod = new HandlerMethod(handler, method);   
	}    
	
	HandlerMethod oldHandlerMethod = handlerMethods.get(mapping);   
	if (oldHandlerMethod != null && !oldHandlerMethod.equals(handlerMethod)) {      
		throw new IllegalStateException("Ambiguous mapping found. Cannot map '" + handlerMethod.getBean() 
				+ "' bean method \n" + handlerMethod + "\nto " + mapping + ": There is already '"            
				+ oldHandlerMethod.getBean() + "' bean method\n" + oldHandlerMethod + " mapped.");   
	}   
	 
	this.handlerMethods.put(mapping, handlerMethod);   
	if (logger.isInfoEnabled()) {      
		logger.info("Mapped \"" + mapping + "\" onto " + handlerMethod);   
	}    
	
	Set<String> patterns = getMappingPathPatterns(mapping);   
	for (String pattern : patterns) {      
		if (!getPathMatcher().isPattern(pattern)) {

        //添加到urlMap中，查找时也会通过这个map查询
        this.urlMap.add(pattern, mapping);
​      	}   
	}
	
}
```

## hm.getHandler 获取 Handler 与 HandlerExecutionChain
回头再来看看dispatchServlet中调用的hm.getHandler方法，它也是在AbstractHandlerMapping中定义的。
```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   Object handler = getHandlerInternal(request);
   if (handler == null) {
      handler = getDefaultHandler();
   }
   if (handler == null) {
      return null;
   }
   // Bean name or resolved handler?
   if (handler instanceof String) {
      String handlerName = (String) handler;
      handler = getApplicationContext().getBean(handlerName);
   }
   return getHandlerExecutionChain(handler, request);
}

// 把 handler 包装成 HandlerExecutionChain  链
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
   HandlerExecutionChain chain = 
      (handler instanceof HandlerExecutionChain) ?
         (HandlerExecutionChain) handler : new HandlerExecutionChain(handler);
         
   chain.addInterceptors(getAdaptedInterceptors());
   
   String lookupPath = urlPathHelper.getLookupPathForRequest(request);
   for (MappedInterceptor mappedInterceptor : mappedInterceptors) {
      if (mappedInterceptor.matches(lookupPath, pathMatcher)) {
         chain.addInterceptor(mappedInterceptor.getInterceptor());
      }
   }
   
   return chain;
}
```
获取对应的 handlerMethod
```java
protected Object getHandlerInternal(HttpServletRequest request) throws Exception {
		String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
        //查找符合匹配规则的handler。可能的结果是HandlerExecutionChain对象或者是null
		Object handler = lookupHandler(lookupPath, request);
        //如果没有找到匹配的handler，则需要处理下default handler
		if (handler == null) {
			// We need to care for the default handler directly, since we need to
			// expose the PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE for it as well.
			Object rawHandler = null;
			if ("/".equals(lookupPath)) {
				rawHandler = getRootHandler();
			}
			if (rawHandler == null) {
				rawHandler = getDefaultHandler();
			}
            //在getRootHandler和getDefaultHandler方法中，可能持有的是bean name。
			if (rawHandler != null) {
				// Bean name or resolved handler?
				if (rawHandler instanceof String) {
					String handlerName = (String) rawHandler;
					rawHandler = getApplicationContext().getBean(handlerName);
				}
				validateHandler(rawHandler, request);
				handler = buildPathExposingHandler(rawHandler, lookupPath, lookupPath, null);
			}
		}
        //如果handler还是为空，则抛出错误。
		if (handler != null && this.mappedInterceptors != null) {
			Set<HandlerInterceptor> mappedInterceptors =
					this.mappedInterceptors.getInterceptors(lookupPath, this.pathMatcher);
			if (!mappedInterceptors.isEmpty()) {
				HandlerExecutionChain chain;
				if (handler instanceof HandlerExecutionChain) {
					chain = (HandlerExecutionChain) handler;
				} else {
					chain = new HandlerExecutionChain(handler);
				}
				chain.addInterceptors(mappedInterceptors.toArray(new HandlerInterceptor[mappedInterceptors.size()]));
			}
		}
		if (handler != null && logger.isDebugEnabled()) {
			logger.debug("Mapping [" + lookupPath + "] to handler '" + handler + "'");
		}
		else if (handler == null && logger.isTraceEnabled()) {
			logger.trace("No handler mapping found for [" + lookupPath + "]");
		}
		return handler;
}

//这个方法可能的返回值是HandlerExecutionChain对象或者是null
//在HandlerExecutionChain对象中的handler，是根据handlerMap中取出来的bean name获得到的bean instance
protected Object lookupHandler(String urlPath, HttpServletRequest request) throws Exception {
		// Direct match?
        //直接匹配
		Object handler = this.handlerMap.get(urlPath);
		if (handler != null) {
			// Bean name or resolved handler?
			if (handler instanceof String) {
				String handlerName = (String) handler;
                //从IoC容器中取出handler
				handler = getApplicationContext().getBean(handlerName);
			}
			validateHandler(handler, request);
            //创建一个HandlerExecutionChain对象并返回
			return buildPathExposingHandler(handler, urlPath, urlPath, null);
		}
		
		// Pattern match?
        //根据一定的模式匹配规则
		List<String> matchingPatterns = new ArrayList<String>();
		for (String registeredPattern : this.handlerMap.keySet()) {
			if (getPathMatcher().match(registeredPattern, urlPath)) {
				matchingPatterns.add(registeredPattern);
			}
		}
		String bestPatternMatch = null;
		if (!matchingPatterns.isEmpty()) {
			Collections.sort(matchingPatterns, getPathMatcher().getPatternComparator(urlPath));
			if (logger.isDebugEnabled()) {
				logger.debug("Matching patterns for request [" + urlPath + "] are " + matchingPatterns);
			}
			bestPatternMatch = matchingPatterns.get(0);
		}
		if (bestPatternMatch != null) {
            //处理最佳匹配
			handler = this.handlerMap.get(bestPatternMatch);
			// Bean name or resolved handler?
			if (handler instanceof String) {
				String handlerName = (String) handler;
				handler = getApplicationContext().getBean(handlerName);
			}
			validateHandler(handler, request);
			String pathWithinMapping = getPathMatcher().extractPathWithinPattern(bestPatternMatch, urlPath);
			Map<String, String> uriTemplateVariables =
					getPathMatcher().extractUriTemplateVariables(bestPatternMatch, urlPath);
            //返回一个HandlerExecutionChain对象
			return buildPathExposingHandler(handler, bestPatternMatch, pathWithinMapping, uriTemplateVariables);
		}
		// No handler found...
		return null;
}
```


## 总结
AbstractHandlerMethodMapping负责注册所有处理器的处理方法，其过程如下：
1. 默认从子web容器中获取所有bean。
2. 找出其中的处理器bean，即bean是否被@Controller注解了。
3. 从处理器bean中找出所有处理方法，即该方法是否被@RequestMapping注解了。发现处理方法后把它的                   @RequestMapping注解解析成RequestMappingInfo对象，再把方法对象包装成HandlerMethod对象。 
4. 把RequestMappingInfo和HandlerMethod对象对象置入缓存handlerMethods，以map形式存放，key为                 RequestMappingInfo，value为HandlerMethod。
5. 从RequestMappingInfo中获取非模式的请求路径集合，把非模式请求路径和RequestMappingInfo置入缓存urlMap。


经过以上解析之后，就可以两种方式获取HandlerMethod，如下：
1. 根据url请求为key尝试从urlMap中获取RequestMappingInfo，在把RequestMappingInfo为key进而从handlerMethods中获取HandlerMethod。
2. 若第一种方式不能获取HandlerMethod，则会遍历handlerMethods中的所有key（key是RequestMappingInfo，RequestMappingInfo中value属性存放着url链接值）进行一一比对，找出最符合的RequestMappingInfo进而找到合适的HandlerMethod。

