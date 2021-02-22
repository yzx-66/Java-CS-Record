当SpringIOC 容器完成了Bean定义资源的定位、载入和解析注册以后，IOC 容器中已经管理了Bean定义的相关数据，但是此时 IOC容器还没有对所管理的Bean进行依赖注入，依赖注入在以下两种情况发生：
1. 用户第一次调用 getBean() 方法时，IOC 容器触发依赖注入
2. 当用户在配置文件中将<bean>元素配置了lazy-init=false属性，即让容器在解析注册 Bean 定义时进行预实例化，触发依赖注入

>DI 大致可以两步：**实例化Bean => 依赖注入**

<img src="https://img-blog.csdnimg.cn/20201209005143410.png?" width="75%">

## 1.寻找获取Bean的入口 (AbstractBeanFactory)

### getBean()

实现了BeanFactory的getBean()方法

```java
// 获取IOC容器中指定名称的Bean
@Override
public Object getBean(String name) throws BeansException {
	// doGetBean才是真正向IoC容器获取被管理Bean的过程
	return doGetBean(name, null, null, false);
}
// 获取IOC容器中指定名称和类型的Bean
@Override
public <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException {
	// doGetBean才是真正向IoC容器获取被管理Bean的过程
	return doGetBean(name, requiredType, null, false);
}

// 获取IOC容器中指定名称和参数的Bean
@Override
public Object getBean(String name, Object... args) throws BeansException {
	// doGetBean才是真正向IoC容器获取被管理Bean的过程
	return doGetBean(name, null, args, false);
}
```

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

通过上面对向 IOC 容器获取 Bean方法的分析，我们可以看到在 Spring 中

* 若 Bean 定义的单例模式(Singleton)，则容器在创建之前先从缓存中查找，以确保整个容器中只存在一个实例对象。
* 若 Bean定义的是原型模式(Prototype)，则容器每次都会创建一个新的实例对象。


除此之外，Bean定义还可以扩展为指定其生命周期范围。

上面的源码只是定义了根据 Bean 定义的模式，采取的不同创建 Bean实例对象的策略，具体的 Bean 实例对象的创建过程由实现了 ObjectFactory 接口的匿名内部类的 createBean()方法完成，ObjectFactory 使用委派模式，具体的 Bean 实例创建过程交由其实现类AbstractAutowireCapableBeanFactory完成。

## 2.开始实例化 (AbstractAutowireCapableBeanFactory)

AbstractAutowireCapableBeanFactory 类实现了ObjectFactory 接口，创建容器指定的 Bean实例对象，同时还对创建的Bean实例对象进行初始化处理。其创建Bean实例对象的方法源码如下：

### createBean()

创建Bean实例对象

```java
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		// 判断需要创建的Bean是否可以实例化，即是否可以通过当前的类加载器加载
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		// 校验和准备Bean中的方法覆盖
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),beanName, 
						"Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			// 如果Bean配置了初始化前和初始化后的处理器，则试图返回一个需要创建Bean的代理对象
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			// 创建Bean的入口
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isDebugEnabled()) {
				logger.debug("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException ex) {
			// A previously detected exception with proper bean creation context already...
			throw ex;
		}
		catch (ImplicitlyAppearedSingletonException ex) {
			// An IllegalStateException to be communicated up to DefaultSingletonBeanRegistry...
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```

### doCreateBean()

真正创建Bean的方法

```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		// 封装被创建的Bean对象
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		// 获取实例化对象的类型
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		// 调用PostProcessor后置处理器
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		// 向容器中缓存单例模式的Bean对象，以防循环引用
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences 
											&& isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +"' to allow for resolving potential circular references");
			}
			// 这里是一个匿名内部类，为了防止循环引用，尽早持有对象的引用
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		// Bean对象的初始化，依赖注入在此触发
		// 这个exposedObject在初始化完成之后返回作为依赖注入完成后的Bean
		Object exposedObject = bean;
		try {
			// 将Bean实例对象封装，并且Bean定义中配置的属性值赋值给实例对象
			populateBean(beanName, mbd, instanceWrapper);
			// 初始化Bean对象
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			// 获取指定名称的已注册的单例模式Bean对象
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				// 根据名称获取的已注册的Bean和正在实例化的Bean是同一个
				if (exposedObject == bean) {
					// 当前实例化的Bean初始化完成
					exposedObject = earlySingletonReference;
				}
				// 当前Bean依赖其他Bean，并且当发生循环引用时不允许新创建实例对象
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) 
                {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>
                        (dependentBeans.length);
					// 获取当前Bean所依赖的其他Bean
					for (String dependentBean : dependentBeans) {
						// 对依赖Bean进行类型检查
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,"Bean with name '" + beanName 
							+ "' has been injected into other beans [" 
							+ StringUtils.collectionToCommaDelimitedString(actualDependentBeans)
                            + "] in its raw version as part of a circular reference, but has  eventually been "  
                            + "wrapped. This means that said other beans do not use the final version of the " 
                            + "bean. This is often the result of over-eager type matching -consider using "
                            + "'getBeanNamesOfType' with the 'allowEagerInit' flag turned  off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		// 注册完成依赖注入的Bean
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, 
					"Invalid destruction signature", ex);
		}
		return exposedObject;
	}
```

通过上面的源码注释，我们看到具体的依赖注入实现其实就在以下两个方法中

* createBeanInstance()方法，生成Bean所包含的java对象实例。
* populateBean()方法，对Bean属性的依赖注入进行处理。

下面继续分析这两个方法的代码实现。

## 3.选择Bean实例化策略 (AbstractAutowireCapableBeanFactory)

### createBeanInstance()

根据指定的初始化策略，使用简单工厂、工厂方法或者容器的自动装配特性生成Java实例对象，创建对象的源码如下：

```java
	// 创建Bean的实例对象
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable 
                                             Object[] args) {
		// Make sure bean class is actually resolved at this point.
		// 检查确认Bean是可实例化的
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		// 使用工厂方法对Bean进行实例化
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}

		if (mbd.getFactoryMethodName() != null)  {
			// 调用工厂方法实例化
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		// 使用容器的自动装配方法进行实例化
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
				// 配置了自动装配属性，使用容器的自动装配实例化
				// 容器的自动装配是根据参数类型匹配Bean的构造方法
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				// 使用默认的无参构造方法实例化
				return instantiateBean(beanName, mbd);
			}
		}

		// Need to determine the constructor...
		// 使用Bean的构造方法进行实例化
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR 
						  ||mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
			// 使用容器的自动装配特性，调用匹配的构造方法实例化
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// No special handling: simply use no-arg constructor.
		// 使用默认的无参构造方法实例化
		return instantiateBean(beanName, mbd);
	}
```

### instantiateBean()

使用默认的无参构造方法实例化Bean对象

```java
	protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			final BeanFactory parent = this;
			// 获取系统的安全管理接口，JDK标准的安全管理API
			if (System.getSecurityManager() != null) {
				// 这里是一个匿名内置类，根据实例化策略创建实例对象
				beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
						getInstantiationStrategy().instantiate(mbd, beanName, parent),getAccessControlContext());
			}
			else {
				// 将实例化的对象封装起来
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
			}
			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
		}
	}
```

经过对上面的代码分析，我们可以看出，对使用工厂方法和自动装配特性的Bean的实例化相当比较清楚，调用相应的工厂方法或者参数匹配的构造方法即可完成实例化对象的工作，但是对于我们最常使用的默认无参构造方法就需要使用相应的初始化策略(JDK的反射机制或者CGLib)来进行初始化了，在方法getInstantiationStrategy().instantiate()中就具体实现类使用初始策略实例化对象。 

## 4.执行Bean实例化 (SimpleInstantiationStrategy)

在使用默认的无参构造方法创建Bean的实例化对象时，方法getInstantiationStrategy().instantiate()调用了SimpleInstantiationStrategy类中的实例化Bean的方法，其源码如下：

### instantiate()

使用初始化策略实例化Bean对象

```java
	@Override
	public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner)
    {
		// Don't override the class with CGLIB if no overrides.
		// 如果Bean定义中没有方法覆盖，则就不需要CGLIB父类类的方法
		if (!bd.hasMethodOverrides()) {
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
				// 获取对象的构造方法或工厂方法
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				// 如果没有构造方法且没有工厂方法
				if (constructorToUse == null) {
					// 使用JDK的反射机制，判断要实例化的Bean是否是接口
					final Class<?> clazz = bd.getBeanClass();
					if (clazz.isInterface()) {
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							// 这里是一个匿名内置类，使用反射机制获取Bean的构造方法
							constructorToUse = AccessController.doPrivileged(
									(PrivilegedExceptionAction<Constructor<?>>) () ->  clazz.getDeclaredConstructor());
						}
						else {
							constructorToUse =	clazz.getDeclaredConstructor();
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Throwable ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
			// 使用BeanUtils实例化，通过反射机制调用”构造方法.newInstance(arg)”来进行实例化
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// Must generate CGLIB subclass.
			// 使用CGLIB来实例化对象
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
	}
```

通过上面的代码分析，我们看到了如果Bean有方法被覆盖了，则使用JDK的反射机制进行实例化，否则，使用CGLib进行实例化。

instantiateWithMethodInjection() 方法调用 **SimpleInstantiationStrategy** 子类CGLibSubclassingInstantiationStrategy使用CGLib来进行初始化，其源码如下：

### instantiate()

使用CGLIB进行Bean对象实例化。

CGLib是一个常用的字节码生成器的类库，它提供了一系列API 实现Java字节码的生成和转换功能。而JDK的动态代理只能针对接口，如果一个类没有实现任何接口，要对其进行动态代理只能使用CGLib。
```java
		public Object instantiate(@Nullable Constructor<?> ctor, @Nullable Object... args) {
			// 创建代理子类
			Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
			Object instance;
			if (ctor == null) {
				instance = BeanUtils.instantiateClass(subclass);
			}
			else {
				try {
					Constructor<?> enhancedSubclassConstructor = subclass.getConstructor(ctor.getParameterTypes());
					instance = enhancedSubclassConstructor.newInstance(args);
				}
				catch (Exception ex) {
					throw new BeanInstantiationException(this.beanDefinition.getBeanClass(),
							"Failed to invoke constructor for CGLIB enhanced subclass [" + subclass.getName() + "]", ex);
				}
			}
			// SPR-10785: set callbacks directly on the instance instead of in the
			// enhanced class (via the Enhancer) in order to avoid memory leaks.
			Factory factory = (Factory) instance;
			factory.setCallbacks(new Callback[] {NoOp.INSTANCE,
					new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
					new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)});
			return instance;
		}

	private Class<?> createEnhancedSubclass(RootBeanDefinition beanDefinition) {
			// CGLIB中的类
			Enhancer enhancer = new Enhancer();
			// 将Bean本身作为其基类
			enhancer.setSuperclass(beanDefinition.getBeanClass());
			enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
			if (this.owner instanceof ConfigurableBeanFactory) {
				ClassLoader cl = ((ConfigurableBeanFactory) this.owner).getBeanClassLoader();
				enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(cl));
			}
			enhancer.setCallbackFilter(new MethodOverrideCallbackFilter(beanDefinition));
			enhancer.setCallbackTypes(CALLBACK_TYPES);
			// 使用CGLIB的createClass方法生成实例对象
			return enhancer.createClass();
	}
```


