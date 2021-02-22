# Advice 拦截器链获取

在为AopProxy代理对象配置拦截器的实现中，有一个取得拦截器的配置过程，这个过程是由 DefaultAdvisorChainFactory 实现的，这个工厂类负责生成拦截器链，在它的 getInterceptorsAndDynamicInterceptionAdvice方法中，有一个适配器和注册过程，通过配置Spring 预先设计好的拦截器，Spring 加入了它对AOP实现的处理。

```java
// AdvisedSupport
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> 
targetClass) {
   MethodCacheKey cacheKey = new MethodCacheKey(method);
   List<Object> cached = this.methodCache.get(cacheKey);
   if (cached == null) {
      cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(this, method, targetClass);
      this.methodCache.put(cacheKey, cached);
   }
   return cached;
}

/**
 * 从提供的配置实例config中获取advisor列表,遍历处理这些advisor.如果是IntroductionAdvisor,
 * 则判断此Advisor能否应用到目标类targetClass上.如果是PointcutAdvisor,则判断
 * 此Advisor能否应用到目标方法method上.将满足条件的Advisor通过AdvisorAdaptor转化成Interceptor列表返回.
 */
@Override
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
      Advised config, Method method, @Nullable Class<?> targetClass) {

   // This is somewhat tricky... We have to process introductions first,
   // but we need to preserve order in the ultimate list.
   List<Object> interceptorList = new ArrayList<>(config.getAdvisors().length);
   Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
   // 查看是否包含IntroductionAdvisor
   boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
   // 这里实际上注册一系列AdvisorAdapter,用于将Advisor转化成MethodInterceptor
   AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
	
  for (Advisor advisor : config.getAdvisors()) {
  	  ...
      MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
	  ...
  }
  

   return interceptorList;
}
```


## GlobalAdvisorAdapterRegistry

GlobalAdvisorAdapterRegistry 负责拦截器的适配和注册过程

```java
public abstract class GlobalAdvisorAdapterRegistry {

   /**
    * Keep track of a single instance so we can return it to classes that request it.
    */
   private static AdvisorAdapterRegistry instance = new DefaultAdvisorAdapterRegistry();

   /**
    * Return the singleton {@link DefaultAdvisorAdapterRegistry} instance.
    */
   public static AdvisorAdapterRegistry getInstance() {
      return instance;
   }

   /**
    * Reset the singleton {@link DefaultAdvisorAdapterRegistry}, removing any
    * {@link AdvisorAdapterRegistry#registerAdvisorAdapter(AdvisorAdapter) registered}
    * adapters.
    */
   static void reset() {
      instance = new DefaultAdvisorAdapterRegistry();
   }

}
```

### DefaultAdvisorAdapterRegistry

而 GlobalAdvisorAdapterRegistry 起到了适配器和单例模式的作用，提供了一个 DefaultAdvisorAdapterRegistry，它用来完成各种通知的适配和注册过程。 

```java
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry, Serializable {

   private final List<AdvisorAdapter> adapters = new ArrayList<>(3);


   /**
    * Create a new DefaultAdvisorAdapterRegistry, registering well-known adapters.
    */
   public DefaultAdvisorAdapterRegistry() {
      registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
      registerAdvisorAdapter(new AfterReturningAdviceAdapter());
      registerAdvisorAdapter(new ThrowsAdviceAdapter());
   }


   @Override
   public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
      if (adviceObject instanceof Advisor) {
         return (Advisor) adviceObject;
      }
      if (!(adviceObject instanceof Advice)) {
         throw new UnknownAdviceTypeException(adviceObject);
      }
      Advice advice = (Advice) adviceObject;
      if (advice instanceof MethodInterceptor) {
         // So well-known it doesn't even need an adapter.
         return new DefaultPointcutAdvisor(advice);
      }
      for (AdvisorAdapter adapter : this.adapters) {
         // Check that it is supported.
         if (adapter.supportsAdvice(advice)) {
            return new DefaultPointcutAdvisor(advice);
         }
      }
      throw new UnknownAdviceTypeException(advice);
   }

   @Override
   public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
      List<MethodInterceptor> interceptors = new ArrayList<>(3);
      Advice advice = advisor.getAdvice();
      if (advice instanceof MethodInterceptor) {
         interceptors.add((MethodInterceptor) advice);
      }
      for (AdvisorAdapter adapter : this.adapters) {
         if (adapter.supportsAdvice(advice)) {
            interceptors.add(adapter.getInterceptor(advisor));
         }
      }
      if (interceptors.isEmpty()) {
         throw new UnknownAdviceTypeException(advisor.getAdvice());
      }
      return interceptors.toArray(new MethodInterceptor[interceptors.size()]);
   }

   @Override
   public void registerAdvisorAdapter(AdvisorAdapter adapter) {
      this.adapters.add(adapter);
   }

}
```

#### MethodBeforeAdviceAdapter

DefaultAdvisorAdapterRegistry 设置了一系列的是配置，正是这些适配器的实现，为Spring AOP 提供了编织能力。下面以 MethodBeforeAdviceAdapter为例，看具体的实现： 

```java
class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {

   @Override
   public boolean supportsAdvice(Advice advice) {
      return (advice instanceof MethodBeforeAdvice);
   }

   @Override
   public MethodInterceptor getInterceptor(Advisor advisor) {
      MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
      return new MethodBeforeAdviceInterceptor(advice);
   }

}
```

# Advice 对应的拦截器种类
## MethodBeforeAdviceInterceptor

Spring AOP为了实现advice的织入，设计了特定的拦截器对这些功能进行了封装。我们接着看MethodBeforeAdviceInterceptor如何完成封装的？ 

```java
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {

   private MethodBeforeAdvice advice;


   /**
    * Create a new MethodBeforeAdviceInterceptor for the given advice.
    * @param advice the MethodBeforeAdvice to wrap
    */
   public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
      Assert.notNull(advice, "Advice must not be null");
      this.advice = advice;
   }

   @Override
   public Object invoke(MethodInvocation mi) throws Throwable {
      this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
      return mi.proceed();
   }

}
```

## AfterReturningAdviceInterceptor

可以看到，invoke方法中，首先触发了advice的before回调，然后才是proceed。AfterReturningAdviceInterceptor的源码： 

```java
public class AfterReturningAdviceInterceptor implements MethodInterceptor, AfterAdvice, 
Serializable {

	private final AfterReturningAdvice advice;


	/**
	 * Create a new AfterReturningAdviceInterceptor for the given advice.
	 * @param advice the AfterReturningAdvice to wrap
	 */
	public AfterReturningAdviceInterceptor(AfterReturningAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}

	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		Object retVal = mi.proceed();
		this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
		return retVal;
	}

}
```

## ThrowsAdviceInterceptor

```java
public Object invoke(MethodInvocation mi) throws Throwable {
   try {
      return mi.proceed();
   }
   catch (Throwable ex) {
      Method handlerMethod = getExceptionHandler(ex);
      if (handlerMethod != null) {
         invokeHandlerMethod(mi, ex, handlerMethod);
      }
      throw ex;
   }
}

private void invokeHandlerMethod(MethodInvocation mi, Throwable ex, Method method) throws 
Throwable {
   Object[] handlerArgs;
   if (method.getParameterCount() == 1) {
      handlerArgs = new Object[] { ex };
   }
   else {
      handlerArgs = new Object[] {mi.getMethod(), mi.getArguments(), mi.getThis(), ex};
   }
   try {
      method.invoke(this.throwsAdvice, handlerArgs);
   }
   catch (InvocationTargetException targetEx) {
      throw targetEx.getTargetException();
   }
}
```

至此，我们知道了对目标对象的增强是通过拦截器实现的。
