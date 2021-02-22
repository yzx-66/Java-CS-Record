在Spring中，有两个很容易混淆的类 BeanFactory 和 FactoryBean：

BeanFactory：Bean 工厂，是一个工厂(Factory)，我们 Spring IOC 容器的最顶层接口就是这个 BeanFactory。它的作用是管理 Bean，即实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。
```java
public interface BeanFactory {

	<T> T getBean(Class<T> requiredType) throws BeansException;
	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;	
	<T> T getBean(String name, Class<T> requiredType) throws BeansException;
	Object getBean(String name) throws BeansException;
	Object getBean(String name, Object... args) throws BeansException;
}
```

>=> Bean工厂顶层规范，核心是 getBean() 方法

FactoryBean：工厂 Bean，是一个 Bean，作用是产生其他 bean实例。通常情况下，这种 Bean 没有什么特别的要求，仅需要提供一个工厂方法，该方法用来返回其他 Bean实例。
```java
// 工厂 Bean，用于产生其他对象 
public interface FactoryBean<T> {
	// 获取容器管理的对象实例
    @Nullable 
    T getObject() throws Exception;
    
	// 获取 Bean 工厂创建的对象的类型 
    @Nullable
    Class<?> getObjectType();
    
	// Bean 工厂创建的对象是否是单态模式，如果是单态模式，则整个容器中只有一个实例对象，每次请求都返回同一个实例对象 
    default boolean isSingleton() { 
        return true;
    }
}
```
>=> Spring内部的一种`&`开头的 bean，也可以理解成是 Spring 的一个扩展点

## 1.FactoryBean 使用场景

一般情况下，我们有两种方式我去注册 bean，applicationContext.xml 中的 `<bean>` 标签或者 JavaConfig 的 @Bean 注解。但是在某些情况下，实例化Bean过程比较复杂，如果按照传统的方式，则需要在`<bean>`中提供大量的配置信息。配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。

Spring 为此提供了一个org.springframework.bean.factory.FactoryBean 的工厂类接口，用户可以通过实现该接口定制实例化Bean的逻辑。我们需要做的就是把这个 factoryBean 注册到 IOC 容器中，当调用 getBean(factoryBean.class) 时获取 bean 得到的就是 factoryBean#getObject() 方法中创建的实例。

> PS：当用户使用容器本身时，可以使用转义字符”&”来得到 FactoryBean 本身，以区别通过 FactoryBean 产生的实例对象和 FactoryBean 对象本身。在 BeanFactory 中通过如下代码定义了该转义字符：`String FACTORY_BEAN_PREFIX = "&";`。如果 myJndiObject 是一个 FactoryBean，则使用 &myJndiObject 得到的是 myJndiObject 对象，而不是 myJndiObject 产生出来的对象。



FactoryBean接口对于Spring框架来说占用重要的地位，Spring自身就提供了70多个FactoryBean的实现。它们隐藏了实例化一些复杂Bean的细节，给上层应用带来了便利。从Spring3.0开始，FactoryBean开始支持**泛型**，即接口声明改为`FactoryBean<T>`的形式

**FactoryBean 使用示例**

FactoryBean 通常是用来创建比较复杂的 bean，一般的 bean 直接用xml配置即可，但如果一个 bean 的创建过程中涉及到很多其他的 bean  和复杂的逻辑，用xml配置比较困难，这时可以考虑用 FactoryBean。

1）创建一个工厂，实现 FactoryBean

```java
public class TestFactoryBean implements FactoryBean {

    private int listType;
    public void setListType(int listType) {
        this.listType = listType;
    }

    @Override
    public Object getObject() throws Exception {
        if (listType == 1) {
            return new ArrayList();
        } else if (listType == 2) {
            return new LinkedList();
        } else {
            return new CopyOnWriteArrayList(); 
        }
    }

    @Override
    public Class<?> getObjectType() {
        return List.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

2）配置 applicationContext.xml

```xml
<bean id="factoryBean" class="com.xupt.yzh.pkg_04.TestFactoryBean">
      <!-- value=2 按照上面逻辑是返回LinkedList-->
      <property name="listType" value="1"></property>
</bean>
```

3）测试代码：

```java
public class Main {

    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
        Class<?> factoryBean = context.getBean("factoryBean").getClass();
        System.out.println(factoryBean);
    }
}
```

结果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201221194341683.png#pic_center)
可以看到最后得到的并不是 FactoryBean 本身，而是根据 FactoryBean#getObject() 的逻辑动态控制生成对象，所以我们可以灵活地操控Bean的生成，这就是FactoryBean的作用。

## 2.FactoryBean 流程分析
在前面我们分析 Spring IOC 容器实例化 Bean 并进行依赖注入过程的源码时，提到在 getBean() 方法触发容器实例化 Bean 的时候会调用 **AbstractBeanFactory** 的 doGetBean() 方法来进行实例化的过程，源码如下：

### doGetBean()
```java
	@SuppressWarnings("unchecked")
	// 真正实现向IOC容器获取Bean的功能，也是触发依赖注入功能的地方
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		// 根据指定的名称获取被管理Bean的名称，剥离指定名称中对容器的相关依赖
		// 如果指定的是别名，将别名转换为规范的Bean名称
		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		// 先从缓存中取是否已经有被创建过的单态类型的Bean
		// 对于单例模式的Bean整个IOC容器中只创建一次，不需要重复创建
		Object sharedInstance = getSingleton(beanName);
		// IOC容器创建单例模式Bean实例对象
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				// 如果指定名称的Bean在容器中已有单例模式的Bean被创建直接返回已经创建的Bean
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName 
						+ "' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			// 获取给定Bean的实例对象，主要是完成FactoryBean的相关处理
			// 注意：BeanFactory是管理容器中Bean的工厂，而FactoryBean是创建创建对象的工厂Bean，两者之间有区别
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			// 缓存没有正在创建的单例模式Bean
			// 缓存中已经有已经创建的原型模式Bean，但是由于循环引用的问题导致实例化对象失败
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			// 对IOC容器中是否存在指定名称的BeanDefinition进行检查
			// 首先检查是否能在当前的BeanFactory中获取的所需要的Bean，如果不能则委托当前容器的父级容器去查找
			// 如果还是找不到则沿着容器的继承体系向父级容器查找
			BeanFactory parentBeanFactory = getParentBeanFactory();
			// 当前容器的父级容器存在，且当前容器中不存在指定名称的Bean
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				// 解析指定Bean名称的原始名称
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					// 委派父级容器根据指定名称和显式的参数查找
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					// 委派父级容器根据指定名称和类型查找
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			// 创建的Bean是否需要进行类型验证，一般不需要
			if (!typeCheckOnly) {
				// 向容器标记指定的Bean已经被创建
				markBeanAsCreated(beanName);
			}

			try {
				// 根据指定Bean名称获取其父级的Bean定义。主要解决Bean继承时子类合并父类公共属性问题
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				// 获取当前Bean所有依赖Bean的名称
				String[] dependsOn = mbd.getDependsOn();
				// 如果当前Bean有依赖Bean
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						// 递归调用getBean方法，获取当前Bean的依赖Bean
						registerDependentBean(dep, beanName);
						// 把被依赖Bean注册给当前依赖的Bean
						getBean(dep);
					}
				}

				// Create bean instance.
				// 创建单例模式Bean的实例对象
				if (mbd.isSingleton()) {
					// 这里使用了一个匿名内部类，创建Bean实例对象，并且注册给所依赖的对象
					sharedInstance = getSingleton(beanName, () -> {
						try {
							// 创建一个指定Bean实例对象，如果有父级继承，则合并子类和父类的定义
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							// 显式地从容器单例模式Bean缓存中清除实例对象
							destroySingleton(beanName);
							throw ex;
						}
					});
					// 获取给定Bean的实例对象
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				// IOC容器创建原型模式Bean实例对象
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					// 原型模式(Prototype)是每次都会创建一个新的对象
					Object prototypeInstance = null;
					try {
						// 回调beforePrototypeCreation方法，默认的功能是注册当前创建的原型对象
						beforePrototypeCreation(beanName);
						// 创建指定Bean对象实例
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						// 回调afterPrototypeCreation方法，默认的功能告诉IOC容器指定Bean的原型对象不再创建
						afterPrototypeCreation(beanName);
					}
					// 获取给定Bean的实例对象
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				// 要创建的Bean既不是单例模式，也不是原型模式，则根据Bean定义资源中配置的生命周期范围，选择实例化Bean的合适方法
				// 这种在Web应用程序中比较常用，如：request、session、application等生命周期
				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					// Bean定义资源中没有配置生命周期范围，则Bean定义不合法
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						// 这里又使用了一个匿名内部类，获取一个指定生命周期范围的实例
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						// 获取给定Bean的实例对象
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " 
						+ "defining a scoped proxy for this bean if you intend to referto it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		// 对创建的Bean实例对象进行类型检查
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" 
						+ ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```


### getObjectForBeanInstance()

获取给定Bean的实例对象，主要是完成FactoryBean的相关处理

```java
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd)
    {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		// 容器已经得到了Bean实例对象（这个实例对象可能是一个普通的Bean，也可能是一个工厂Bean）
		// 如果是一个工厂Bean，则使用它创建一个Bean实例对象，
		// 如果调用本身就想获得一个容器的引用，则指定返回这个工厂Bean实例对象
		// 如果指定的名称是容器的解引用(dereference，即是对象本身而非内存地址)，且Bean实例也不是创建Bean实例对象的工厂Bean
		if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the caller actually wants a reference to the factory.
		// 如果Bean实例不是工厂Bean，或者指定名称是容器的解引用，调用者向获取对容器的引用，则直接返回当前的Bean实例
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}

		// 处理指定名称不是容器的解引用，或者根据名称获取的Bean实例对象是一个工厂Bean
		// 使用工厂Bean创建一个Bean的实例对象
		Object object = null;
		if (mbd == null) {
			// 从Bean工厂缓存中获取给定名称的Bean实例对象
			object = getCachedObjectForFactoryBean(beanName);
		}
		// 让Bean工厂生产给定名称的Bean对象实例
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			// 如果从Bean工厂生产的Bean是单态模式的，则缓存
			if (mbd == null && containsBeanDefinition(beanName)) {
				// 从容器中获取指定名称的Bean定义，如果继承基类，则合并基类相关属性
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			// 如果从容器得到Bean定义信息，并且Bean定义信息不是虚构的，则让工厂Bean生产Bean实例对象
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			// 调用FactoryBeanRegistrySupport类的getObjectFromFactoryBean方法，实现工厂Bean生产Bean对象实例的过程
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```

在上面获取给定 Bean 的实例对象的 getObjectForBeanInstance() 方法中 ， 会调用 FactoryBeanRegistrySupport 类的 getObjectFromFactoryBean()方法，该方法实现了 Bean 工厂生产Bean实例对象。

>Dereference(解引用)：一个在C/C++中应用比较多的术语，在C++中，”*”是解引用符号，而”&”是引用符号，解引用是指变量指向的是所引用对象的本身数据，而不是引用对象的内存地址。


### getObjectFromFactoryBean()

Bean工厂生产Bean实例对象,在 **FactoryBeanRegistrySupport** 类中

```java
	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
		// Bean工厂是单态模式，并且Bean工厂缓存中存在指定名称的Bean实例对象
		if (factory.isSingleton() && containsSingleton(beanName)) {
			// 多线程同步，以防止数据不一致
			synchronized (getSingletonMutex()) {
				// 直接从Bean工厂缓存中获取指定名称的Bean实例对象
				Object object = this.factoryBeanObjectCache.get(beanName);
				// Bean工厂缓存中没有指定名称的实例对象，则生产该实例对象
				if (object == null) {
					// 调用Bean工厂的getObject方法生产指定Bean的实例对象
					object = doGetObjectFromFactoryBean(factory, beanName);
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					if (alreadyThere != null) {
						object = alreadyThere;
					}
					else {
						if (shouldPostProcess) {
							try {
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
								"Post-processing of FactoryBean's singleton object failed", ex);
							}
						}
						// 将生产的实例对象添加到Bean工厂缓存中
						this.factoryBeanObjectCache.put(beanName, object);
					}
				}
				return object;
			}
		}
		// 调用Bean工厂的getObject方法生产指定Bean的实例对象
		else {
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			if (shouldPostProcess) {
				try {
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean'sobject failed", ex);
				}
			}
			return object;
		}
	}
```

### doGetObjectFromFactoryBean()

调用 Bean工厂的 getObject() 方法生产指定Bean的实例对象

```java
	private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)throws BeanCreationException {

		Object object;
		try {
			if (System.getSecurityManager() != null) {
				AccessControlContext acc = getAccessControlContext();
				try {
					// 实现PrivilegedExceptionAction接口的匿名内置类
					// 根据JVM检查权限，然后决定BeanFactory创建实例对象
					object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> factory.getObject(), acc);
				} catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				// 调用BeanFactory接口实现类的创建对象方法
				object = factory.getObject();
			}
		}
		catch (FactoryBeanNotInitializedException ex) {
			throw new BeanCurrentlyInCreationException(beanName, ex.toString());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
		}

		// Do not accept a null value for a FactoryBean that's not fully initialized yet: Many FactoryBeans just return null then.
		// 创建出来的实例对象为null，或者因为单态对象正在创建而返回null
		if (object == null) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(
						beanName, "FactoryBean which is currently in creation returned null from getObject");
			}
			object = new NullBean();
		}
		return object;
	}
```
从上面的源码分析中，我们可以看出，BeanFactory 接口调用其实现类的 getObject 方法来实现创建 Bean实例对象功能。

FactoryBean 的实现类有非常多，比如：Proxy、RMI、JNDI、ServletContextFactoryBean 等等。FactoryBean 接口为 Spring 容器提供了一个很好的封装机制，具体的 getObject()有不同的实现类根据不同的实现策略来具体提供，我们分析一个最简单的AnnotationTestFactoryBean的实现源码：

### AnnotationTestBeanFactory

其他的 Proxy，RMI，JNDI等等，都是根据相应的策略提供getObject()的实现。

```java
public class AnnotationTestBeanFactory implements FactoryBean<FactoryCreatedAnnotationTestBean> {

	private final FactoryCreatedAnnotationTestBean instance = new FactoryCreatedAnnotationTestBean();

	public AnnotationTestBeanFactory() {
		this.instance.setName("FACTORY");
	}

	@Override
	public FactoryCreatedAnnotationTestBean getObject() throws Exception {
		return this.instance;
	}

	// AnnotationTestBeanFactory产生Bean实例对象的实现
	@Override
	public Class<? extends IJmxTestBean> getObjectType() {
		return FactoryCreatedAnnotationTestBean.class;
	}

	@Override
	public boolean isSingleton() {
		return true;
	}
}
```

