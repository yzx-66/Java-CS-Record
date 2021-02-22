## singleton 三层缓存
```java
/** Cache of singleton objects: bean name to bean instance. */
/** 缓存单例对象， K-V -> BeanName - Bean 实例 */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of early singleton objects: bean name to bean instance. */
/** 缓存早期单例对象 */
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

/** Cache of singleton factories: bean name to ObjectFactory. */
/** 缓存 Bean 工厂 */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

## doGetBean 里的两种 getSingleton()
doGetBean 一进去调用的：

<img src="https://img-blog.csdnimg.cn/20210221021102954.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70"  width="70%"/>

```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  
  // 尝试从缓存中获取成品的目标对象，如果存在，则直接返回
  Object singletonObject = this.singletonObjects.get(beanName);
  
  // 如果缓存中不存在目标对象，则判断当前对象是否已经处于创建过程中，在前面的讲解中，第一次尝试获取A对象
  // 的实例之后，就会将A对象标记为正在创建中，因而最后再尝试获取A对象的时候，这里的if判断就会为true
  if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    
    synchronized (this.singletonObjects) {
      singletonObject = this.earlySingletonObjects.get(beanName);
      if (singletonObject == null && allowEarlyReference) {
        
        // 这里的singletonFactories是一个Map，其key是bean的名称，而值是一个ObjectFactory类型的
        // 对象，这里对于A和B而言，调用图其getObject()方法返回的就是A和B对象的实例，无论是否是半成品
        ObjectFactory singletonFactory = this.singletonFactories.get(beanName);
        if (singletonFactory != null) {
          
          // 获取目标对象的实例
          singletonObject = singletonFactory.getObject();
          this.earlySingletonObjects.put(beanName, singletonObject);
          this.singletonFactories.remove(beanName);
        }
      }
    }
  }
  return singletonObject;
}
```

上一个 getSingleton 返回值为 null，并且 beanDefition 为 singleton 的作用域时调用

<img src="https://img-blog.csdnimg.cn/20210221021141803.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70"  width="70%"/>


```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    // 加锁
    synchronized (this.singletonObjects) {
        // 检查 singletonObjects 缓存中是否有
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            // 检查是否在执行销毁
            if (this.singletonsCurrentlyInDestruction) {
                throw new BeanCreationNotAllowedException(beanName,
                        "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                        "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
            }
            // 将 Bean 添加到 singletonsCurrentlyInCreation 集合中, 表示正在创建
            beforeSingletonCreation(beanName);
            boolean newSingleton = false;
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
            if (recordSuppressedExceptions) {
                this.suppressedExceptions = new LinkedHashSet<>();
            }
            try {
                // 调用工厂方法
                // 也就是调用 createBean(beanName, mbd, args)
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            catch (IllegalStateException ex) {
                // Has the singleton object implicitly appeared in the meantime ->
                // if yes, proceed with it since the exception indicates that state.
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    throw ex;
                }
            }
            catch (BeanCreationException ex) {
                if (recordSuppressedExceptions) {
                    for (Exception suppressedException : this.suppressedExceptions) {
                        ex.addRelatedCause(suppressedException);
                    }
                }
                throw ex;
            }
            finally {
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = null;
                }
                // 创建成功, 从 singletonsCurrentlyInCreation 移除
                afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                // 将给定的单例对象添加到该工厂的单例缓存中
                //     this.singletonObjects.put(beanName, singletonObject);
                //     this.singletonFactories.remove(beanName);
                //     this.earlySingletonObjects.remove(beanName);
                //     this.registeredSingletons.add(beanName);
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}
```

## doCretaeBean 里放入 singletonFactories


<img src="https://img-blog.csdnimg.cn/2021022102121133.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70"  width="70%"/>

##  循环依赖总结
解决方法：singleton 家族，三层 cache
* 第一层 singletonObjects（已经初始化完，成品 bean ，通过 getSingleton 的重载方法放入，该重载方法的入参是 beanName 和 ObjectFactory 的未实现方法 getObject，该 ObjectFactory 使用 lamda 形式创建，在其唯一的 getObject 方法中会调用 createBean 创建 bean，然后用 addSingleton 方法放入 singletonObjects ）
* 第二层 earlySingletonOjbects（半成品 bean，不会直接在某个步骤直接放入，在 doGetBean 方法一进去就调用的 getSingleton（beanName) 中，如果 singletonObjects 中没有，但是 singletonFactores 中有的话，会把 singletonFactories 中该半成品 bean 放入 earlySingleton ，代表未初始化完成但是被依赖）
* 第三层 singletonFactoies （半成品 bean，在 docreateBean 的 createBeanInstance 完成后调用 addSingletonFactories 放入，也正是因为在这个时机放入，所以如果因为构造方法出现循环依赖无法解决，因为 createBeanInstance 最后会调用 SimpleInitalizeStagy 的 instant 方法，该方法在 beanDefition 不存在覆盖方法时就会用 JDK 的反射创建对象，创建的方法就是获得 constructor 然后 newInstance，所以如果在解析 constructor 的参数有循环依赖，那么当前的半成品还未放入 singletonFactoies）
