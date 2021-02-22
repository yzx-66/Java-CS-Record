我们已经知道 **IOC 容器的初始化过程就是对 Bean 定义资源的定位、载入和注册**。此时容器对Bean的依赖注入并没有发生，依赖注入主要是在应用程序第一次向容器索取Bean时，通过getBean()方法的调用完成。

当Bean定义资源的`<Bean>`元素中配置了 lazy-init=false 属性时，容器将会在初始化的时候对所配置的 Bean 进行预实例化，Bean 的依赖注入在容器初始化的时候就已经完成。这样，当应用程序第一次向容器索取被管理的 Bean时，就不用再初始化和对 Bean进行依赖注入了，直接从容器中获取已经完成依赖注入的现成Bean。可以提高应用第一次向容器获取Bean的性能。

### refresh()

先从IOC 容器的初始化过程开始，我们知道 IOC 容器读入已经定位的 Bean定义资源是从refresh()方法开始的，我们首先从**AbstractApplicationContext**类的refresh()方法入手分析，源码如下：

```java
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			// 1、调用容器准备刷新的方法，获取容器的当时时间，同时给容器设置同步标识
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			// 2、告诉子类启动refreshBeanFactory()方法，Bean定义资源文件的载入从子类的refreshBeanFactory()方法启动
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			// 3、为BeanFactory配置容器特性，例如类加载器、事件处理器等
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				// 4、为容器的某些子类指定特殊的BeanPost事件处理器
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				// 5、调用所有注册的BeanFactoryPostProcessor的Bean
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				// 6、为BeanFactory注册BeanPost事件处理器.
				// BeanPostProcessor是Bean后置处理器，用于监听容器触发的事件
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				// 7、初始化信息源，和国际化相关.
				initMessageSource();

				// Initialize event multicaster for this context.
				// 8、初始化容器事件传播器.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				// 9、调用子类的某些特殊Bean初始化方法
				onRefresh();

				// Check for listener beans and register them.
				// 10、为事件传播器注册事件监听器.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				// 11、初始化所有剩余的单例Bean
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				// 12、初始化容器的生命周期事件处理器，并发布容器的生命周期事件
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				// 13、销毁已创建的Bean
				destroyBeans();

				// Reset 'active' flag.
				// 14、取消refresh操作，重置容器的同步标识。
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				// 15、重设公共缓存
				resetCommonCaches();
			}
		}
}
```
在refresh()方法中 `ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory()`;启动了Bean定义资源的载入、注册过程。而 finishBeanFactoryInitialization 方法是对注册后的Bean定义中的预实例化(lazy-init=false,Spring默认就是预实例化,即为true)的Bean进行处理的地方。

### finishBeanFactoryInitialization()

对配置了lazy-init属性的Bean进行预实例化处理

```java
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		// 这是Spring3以后新加的代码，为容器指定一个转换服务(ConversionService)，在对某些Bean属性进行转换时使用
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) 
        {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		// 为了类型匹配，停止使用临时的类加载器
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		// 缓存容器中所有注册的BeanDefinition元数据，以防被修改
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		// 对配置了lazy-init属性的单态模式Bean进行预实例化处理
		beanFactory.preInstantiateSingletons();
	}
```

### preInstantiateSingletons()

ConfigurableListableBeanFactory 是一个接口， 其 preInstantiateSingletons()方法由其子类**DefaultListableBeanFactory** 提供对配置lazy-init属性单态Bean的预实例化。

```java
	@Override
	public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}

		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			// 获取指定名称的Bean定义
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			// Bean不是抽象的，是单态模式的，且lazy-init属性配置为false
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				// 如果指定名称的bean是创建容器的Bean
				if (isFactoryBean(beanName)) {
					// FACTORY_BEAN_PREFIX=”&”，当Bean名称前面加”&”符号时，获取的是产生容器对象本身，而不是容器产生的Bean.
					// 调用getBean方法，触发容器对Bean实例化和依赖注入过程
					final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
					// 标识是否需要预实例化
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						// 一个匿名内部类
						isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)() ->
								((SmartFactoryBean<?>) factory).isEagerInit(),getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean 
										&& ((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						// 调用getBean方法，触发容器对Bean实例化和依赖注入过程
						getBean(beanName);
					}
				}
				else {
					getBean(beanName);
				}
			}
		}
	}
```

通过对 lazy-init处理源码的分析，我们可以看出，如果设置了 lazy-init 属性，则容器在完成 **Bean 定义的注册之后，会通过getBean方法**，触发对指定Bean的初始化和依赖注入过程，这样当应用第一次向容器索取所需的 Bean时，容器不再需要对 Bean进行初始化和依赖注入，直接从已经完成实例化和依赖注入的Bean中取一个现成的Bean，这样就**提高了第一次获取Bean的性能。** 




