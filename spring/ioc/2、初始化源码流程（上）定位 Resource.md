![在这里插入图片描述](https://img-blog.csdnimg.cn/20201208152354587.png?)

## 1.寻找入口 (ClassPathXmlApplication)

### ClassPathXmlApplicationContext()

```java
ApplicationContext app = new ClassPathXmlApplicationContext("application.xml");
```

进入ClassPathXMLApplication查看其相应构造函数：
 
```java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
}

// 但实际调用的构造函数是
public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
}
```

## 2.获取资源加载器 (AbstractApplicationContext)

首先，调用父类容器的构造方法super(parent)为容器设置好Bean资源加载器。

### AbstractApplicationContext()

通过追踪 ClassPathXmlApplicationContext 的继承体系 ， 发现其父类的父类**AbstractApplicationContext**中初始化IOC容器所做的主要源码如下：

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {

	// 静态初始化块，在整个容器创建过程中只执行一次
	static {
		// 为了避免应用程序在Weblogic8.1关闭时出现类加载异常加载问题，加载IoC容器关闭事件(ContextClosedEvent)类
		ContextClosedEvent.class.getName();
	}
	
	// 构造器中可以传入父容器
	public AbstractApplicationContext(@Nullable ApplicationContext parent) {
		this();
		setParent(parent);
	}
	// 空参构造
	public AbstractApplicationContext() {
		// 获取一个Spring Source的加载器用于读入Spring Bean定义资源文件，方法如下
		this.resourcePatternResolver = getResourcePatternResolver();
	}
	
	//....
}
```

### getResourcePatternResolver()

```java
protected ResourcePatternResolver getResourcePatternResolver() {
		// AbstractApplicationContext继承DefaultResourceLoader，因此也是一个资源加载器
		// Spring资源加载器，其getResource(String location)方法用于载入资源
		return new PathMatchingResourcePatternResolver(this);
}
```

其中PathMatchingResourcePatternResolver的构造函数如下

```java
public PathMatchingResourcePatternResolver(ResourceLoader resourceLoader) {
		Assert.notNull(resourceLoader, "ResourceLoader must not be null");
		// 设置Spring的资源加载器
		this.resourceLoader = resourceLoader;
}
```

## 3.开始解析配置路径  (AbstractRefreshableConfigApplicationContext)

### setConfigLocations()

获取到资源加载器后， 再调用父类 **AbstractRefreshableConfigApplicationContext** 的 setConfigLocations() 方法设定位配置文件的路径信息

```java
// 处理单个资源文件路径为一个字符串的情况
public void setConfigLocation(String location) {
	// String CONFIG_LOCATION_DELIMITERS = ",; /t/n";
	// 即多个资源文件路径之间用” ,; \t\n”分隔，解析成数组形式
	setConfigLocations(StringUtils.tokenizeToStringArray(location, CONFIG_LOCATION_DELIMITERS));
}

// 处理多个资源文件字符串数组，并解析Bean定义资源文件的路径，
public void setConfigLocations(@Nullable String... locations) {
	if (locations != null) {
		Assert.noNullElements(locations, "Config locations must not be null");
		this.configLocations = new String[locations.length];
		for (int i = 0; i < locations.length; i++) {
			// resolvePath为同一个类中将字符串解析为路径的方法
			this.configLocations[i] = resolvePath(locations[i]).trim();
		}
	}
	else {
		this.configLocations = null;
	}
}

// 子类可以重写路径解析方法
protected String resolvePath(String path) {
	return getEnvironment().resolveRequiredPlaceholders(path);
}	
```

通过这两个方法的源码我们可以看出，我们既可以使用一个字符串来配置多个Spring Bean配置信息，也可以使用字符串数组，即下面两种方式都是可以的：

```java
// 多个资源文件路径之间可以是用” , ; \t\n”等分隔。
ClassPathResource res = new ClassPathResource("a.xml,b.xml");//
ClassPathResource res =new ClassPathResource(new String[]{"a.xml","b.xml"})
```

**至此，SpringIOC 容器在初始化时将配置的Bean配置信息定位为Spring封装的Resource。** 

## 4.开始启动流程 (AbstractApplicationContext)

### refresh()

SpringIOC 容器对 Bean配置资源的载入是从 refresh()函数开始的，**refresh()是一个模板方法**，规定了IOC 容器的**启动流程**，有些逻辑要交给其子类去实现，它定义在**AbstractApplicationContext**中

refresh()方法主要为 IOC 容器 Bean 的**生命周期管理提供条件**。整个 refresh() 中 `ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory()` 这句以后代码的都是注册容器的信息源和生命周期事件，我们前面说的载入就是从这句代码开始启动。Spring IOC 容器载入 Bean 配置信息是从其子类容器的 refreshBeanFactory() 方法启动。
>在创建 IOC 容器前，**如果已经有容器存在，则需要把已有的容器销毁和关闭**，以保证在refresh之后使用的是新建立起来的 IOC容器。它类似于对IOC 容器的重启，在新建立好的容器中对容器进行初始化，对Bean配置资源进行载入。 

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

### obtainFreshBeanFactory()

obtainFreshBeanFactory()方法，调用子类容器的 refreshBeanFactory()方法，启动容器载入Bean配置信息。它也是在AbstractApplicationContext 中

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		// 这里使用了委派设计模式，即父类定义了抽象的refreshBeanFactory()方法，具体实现是调用子类容器的refreshBeanFactory()方法
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
}
```



## 5.创建容器 (AbstractRefreshableApplicationContext)

### refreshBeanFactory()

AbstractApplicationContext 类中只抽象定义了 refreshBeanFactory()方法，容器真正调用的是其子类 **AbstractRefreshableApplicationContext** 实现的 refreshBeanFactory()方法

```java
protected final void refreshBeanFactory() throws BeansException {
		// 如果已经有容器，销毁容器中的bean，关闭容器
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
            
		}
		try {
			// 创建IOC容器
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			// 对IOC容器进行定制化，如设置启动参数，开启注解的自动装配等
			customizeBeanFactory(beanFactory);
             // 装载Bean的定义
			 // 调用载入Bean定义的方法，主要这里又使用了一个委派模式，在当前类中只定义了抽象的loadBeanDefinitions方法，具体的实现调用子类容器
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " 
                                                  + getDisplayName(), ex);
		}
}
```

## 6.载入配置路径 (**AbstractXmlApplicationContext**)

AbstractRefreshableApplicationContext 中只定义了抽象的 loadBeanDefinitions 方法，容器真正调用的是其子类 **AbstractXmlApplicationContext** 对该方法的实现

### loadBeanDefinitions(beanFactory)

```java
// 实现父类抽象的载入Bean定义方法
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws 
        BeansException, IOException {

		// 创建XmlBeanDefinitionReader，即创建Bean读取器，并通过回调设置到容器中去，容器使用该读取器读取Bean定义资源
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// 为Bean读取器设置Spring资源加载器
		// AbstractXmlApplicationContx祖先父类AbstractApplicationContext继承DefaultResourceLoader，因此，容器本身也是一个资源加载器
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		// 为Bean读取器设置SAX xml解析器
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// 当Bean读取器读取Bean定义的Xml资源文件时，启用Xml的校验机制
		initBeanDefinitionReader(beanDefinitionReader);
		// Bean读取器真正实现加载的方法
		loadBeanDefinitions(beanDefinitionReader);
}
```

### initBeanDefinitionReader()

```java
protected void initBeanDefinitionReader(XmlBeanDefinitionReader reader) {
		reader.setValidating(this.validating);
}
```

### loadBeanDefinitions(XmlBeanDefinitionReader)

```java
// Xml Bean读取器加载Bean定义资源
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException,
	IOException {
		// 获取Bean定义资源的定位
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			// Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取定位的Bean定义资源
			reader.loadBeanDefinitions(configResources);
		}
		// 如果子类中获取的Bean定义资源定位为空，则获取FileSystemXmlApplicationContext构造方法中setConfigLocations方法设置的资源
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			// Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取定位的Bean定义资源
			reader.loadBeanDefinitions(configLocations);
		}
}

// 这里又使用了一个委托模式，调用子类的获取Bean定义资源定位的方法，该方法在ClassPathXmlApplicationContext中进行实现
@Nullable
protected Resource[] getConfigResources() {
	return null;
}
```

以 XmlBean 读取器的其中一种策略 XmlBeanDefinitionReader 为例。XmlBeanDefinitionReader调用其父类AbstractBeanDefinitionReader的 reader.loadBeanDefinitions()方法读取Bean配置资源。

由于我们使用ClassPathXmlApplicationContext 作为例子分析，因此getConfigResources 的返回值为null，因此程序执行reader.loadBeanDefinitions(configLocations)分支。 

## 7.分配路径处理策略 (AbstractBeanDefinitionReader)

AbstractRefreshableConfigApplicationContext 的 loadBeanDefinitions(Resource...resources)方法实际上是通过调用AbstractXmlApplication的loadBeanDefinitions来调用XmlBeanDefinitionReader的抽象父类**AbstractBeanDefinitionReader**的loadBeanDefinitions()方法：**定义载入过程**

### loadBeanDefinitions(String)

重载方法，调用下面的loadBeanDefinitions(String, Set<Resource>);方法

```java
@Override
public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(location, null);
}
```

### loadBeanDefinitions(String...)

重载方法，调用loadBeanDefinitions(String);

```java
@Override
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
	Assert.notNull(locations, "Location array must not be null");
	int counter = 0;
	for (String location : locations) {
		counter += loadBeanDefinitions(location);
	}
	return counter;
}
```

### loadBeanDefinitions(String,Resource)

```java
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws 
    BeanDefinitionStoreException {
		// 获取在IoC容器初始化过程中设置的资源加载器
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				// 将指定位置的Bean定义资源文件解析为Spring IOC容器封装的资源
				// 加载多个指定位置的Bean定义资源文件
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				// 委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能
				int loadCount = loadBeanDefinitions(resources);
				if (actualResources != null) {
					for (Resource resource : resources) {
						actualResources.add(resource);
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
				}
				return loadCount;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]",ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			// 将指定位置的Bean定义资源文件解析为Spring IOC容器封装的资源
			// 加载单个指定位置的Bean定义资源文件
			Resource resource = resourceLoader.getResource(location);
			// 委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能
			int loadCount = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
			}
			return loadCount;
		}
}
```

从对 AbstractBeanDefinitionReader 的 loadBeanDefinitions()方法源码分析可以看出该方法就做了两件事：

* 首先，调用资源加载器的获取资源方法resourceLoader.getResource(location)，**获取到要加载的资源**。

* 其次，真正**执行加载功能**是其子类 XmlBeanDefinitionReader 的 loadBeanDefinitions()方法。在loadBeanDefinitions()方法中调用了 AbstractApplicationContext的 getResources()方法，跟进去之后发现 getResources()方法其实定义在 ResourcePatternResolver 中，此时，我们有必要来看一下ResourcePatternResolver的全类图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201202192949428.png?)




  从上面可以看到 ResourceLoader 与 ApplicationContext 的继承关系，可以看出其实际调用的DefaultResourceLoader 中 的 getSource() 方 法 定 位 Resource ， 因为ClassPathXmlApplicationContext 本身就是 DefaultResourceLoader 的实现类，所以此时又回到了ClassPathXmlApplicationContext中来。 

## 8.解析配置文件路径 (DefaultResourceLoader)

### getResource()

XmlBeanDefinitionReader 通过调用 ClassPathXmlApplicationContext 的父类 **DefaultResourceLoader** 的 getResource()方法**获取要加载的资源**

```java
// 获取Resource的具体实现方法
@Override
public Resource getResource(String location) {
	Assert.notNull(location, "Location must not be null");

	for (ProtocolResolver protocolResolver : this.protocolResolvers) {
		Resource resource = protocolResolver.resolve(location, this);
		if (resource != null) {
			return resource;
		}
	}
	// 如果是类路径的方式，那需要使用ClassPathResource 来得到bean文件的资源对象
	if (location.startsWith("/")) {
		return getResourceByPath(location);
	}
	else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
		return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()),getClassLoader());
	}
	else {
		try {
			// Try to parse the location as a URL...
			// 如果是URL 方式，使用UrlResource 作为 bean 文件的资源对象
			URL url = new URL(location);
			return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
		}
		catch (MalformedURLException ex) {
			// No URL -> resolve as resource path.
			// 如果既不是classpath标识，又不是URL标识的Resource定位，则调用容器本身的getResourceByPath方法获取Resource
			return getResourceByPath(location);
		}
	}
}
```

### getResourceByPath()

DefaultResourceLoader 提供了 getResourceByPath()方法的实现，就是为了处理既不是 classpath标识，又不是URL标识的Resource定位这种情况

```java
protected Resource getResourceByPath(String path) {
		return new ClassPathContextResource(path, getClassLoader());
}
```

在 ClassPathResource中完成了对整个路径的解析。这样，就可以从类路径上**对 IOC 配置文件进行加载**，当然我们可以按照这个逻辑从任何地方加载，在 Spring中我们看到它提供的各种资源抽象，比如 ClassPathResource、URLResource、FileSystemResource等来供我们使用。

上面我们看到的是定位Resource 的一个过程，而这只是加载过程的一部分。例如 FileSystemXmlApplication 容器就重写了getResourceByPath()方法：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020120219311047.png?)


通过子类的覆盖，巧妙地完成了将类路径变为文件路径的转换。

## 9.读取配置内容 (XmlBeanDefinitionReader)

回到 **XmlBeanDefinitionReader** 的 loadBeanDefinitions()方法看到代表 bean 文件的资源定义以后的载入过程

### loadBeanDefinitions(Resource)

XmlBeanDefinitionReader加载资源的入口方法

```java
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
	// 将读入的XML资源进行特殊编码处理
	return loadBeanDefinitions(new EncodedResource(resource));
}
```

### loadBeanDefinitions(EncodedResource)

这里是载入XML形式Bean定义资源文件方法

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws 
       BeanDefinitionStoreException {
	//...
	try {
		// 将资源文件转为InputStream的IO流
		InputStream inputStream = encodedResource.getResource().getInputStream();
		try {
			// 从InputStream中得到XML的解析源
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
			// 这里是具体的读取过程
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
		}
		finally {
			// 关闭从Resource中得到的IO流
			inputStream.close();
		}
	}
	//...
}
```

### doLoadBeanDefinitions(Input, Res)

从特定XML文件中实际载入Bean定义资源的方法

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
		throws BeanDefinitionStoreException {
	try {
		// 将XML文件转换为DOM对象，解析过程由documentLoader实现
		Document doc = doLoadDocument(inputSource, resource);
		// 这里是启动对Bean定义解析的详细过程，该解析过程会用到Spring的Bean配置规则
		return registerBeanDefinitions(doc, resource);
	}
	//...		
}
```
## 10.解析配置文件 (DefaultDocumentLoader)

DocumentLoader的实现类**DefaultDocumentLoader** 将Bean配置资源转换成Document对象

### loadDocument()

使用标准的JAXP将载入的Bean定义资源转换成document对象

```java
@Override
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
		ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

	// 创建文件解析器工厂
	DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
	if (logger.isDebugEnabled()) {
		logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
	}
	// 创建文档解析器
	DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
	// 解析Spring的Bean定义资源
	return builder.parse(inputSource);
}
```

### createDocumentBuilderFactory()

```java
protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware)  throws ParserConfigurationException {

		// 创建文档解析工厂
		DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
		factory.setNamespaceAware(namespaceAware);

		// 设置解析XML的校验
		if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
			factory.setValidating(true);
			if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
				// Enforce namespace aware for XSD...
				factory.setNamespaceAware(true);
				try {
					factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
				}
				catch (IllegalArgumentException ex) {
					ParserConfigurationException pcex = new ParserConfigurationException(
							"Unable to validate using XSD: Your JAXP provider [" + factory 
							+ "] does not support XML Schema. Are you running on Java 1.4 with Apache Crimson? "
							+ "Upgrade to Apache Xerces (or Java 1.5) for full XSD support.");
					pcex.initCause(ex);
					throw pcex;
				}
			}
		}

		return factory;
}
```

上面的解析过程是调用 JavaEE 标准的 JAXP 标准进行处理。至此 SpringIOC 容器根据定位的 Bean 配置信息，将其**加载读入并转换成为 Document 对象过程完成**。

>下一篇我们继续分析 Spring IOC 容器将载入的 **Bean 配置信息转换为 Document 对象之后，是如何将其解析为 SpringIOC 管理的 Bean 对象 并将其注册到容器中的**。
