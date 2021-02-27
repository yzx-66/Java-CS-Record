要分析 mybatis 的执行原理，首先要搞清楚的就是它如何解析配置文件的。

> PS：配置文件其实有两种，一个是由许多标签构成的**mybatis-config.xml**全局配置文件，另一个是可能有多个的**Mapper.xml**文件。本篇我们只看全局配置文件是如何解析的。

在[【MyBatis】编程式使用及核心对象生命周期](https://yzx66.blog.csdn.net/article/details/114156629) 我们说了如何使用 mybatis
```java
String resource = "mybatis-config.xml"; 
InputStream inputStream = Resources. getResourceAsStream(resource); 
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

// 在创建SqlSessionFactory就可以去获取SqlSession执行sql了
```
所以，解析配置文件的奥秘就在 SqlSessionFactoryBuilder#build 中，我们下面就从源码中寻找答案...
## SqlSessionFactoryBuilder
SqlSessionFactoryBuilder 采用了建造者模式，是用来构建 SqlSessionFactory 的
>SqlSessionFactory 只需要一个（单例的），所以只要构建了这一个 SqlSessionFactory，它的使命就完成了，也就没有存在的意义了。所以它的生命周期只存在于方法的局部。

我们打开 SqlSessionFactoryBuilder，可以发现里面全是 build 方法

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201216171433677.png?)


```java
// 入口的build方法
public SqlSessionFactory build(Reader reader) {
        return this.build((Reader)reader, (String)null, (Properties)null);
    }

// 1.获取XMLConfigBuilder对象
// 2.通过XMLConfigBuilder建造Configuration对象（单例）
// 3.通过Configuration构造SqlSessionFactory（单例）
public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
      
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      // XMLConfigBuilder#parse()会返回一个Configuration对象
      return build(parser.parse());
      
      //...
  }

// 最终的build方法
public SqlSessionFactory build(Configuration config) {
		// SqlSessionFactory 是一个接口
		// 所以，根据配置信息返回一个默认的实现类实例 SqlSessionFactory
		// 注意，这里再构造 DefaultSqlSessionFactory 时传入了 Configuration（后面会用到）
        return new DefaultSqlSessionFactory(config);
    }
```
<img src="https://img-blog.csdnimg.cn/20201216192613618.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNTkyNw==,size_16,color_FFFFFF,t_70#pic_center" width="60%"><br>


所以，我们现在要解决的就是下面两个问题：
1. Configuration 到底是什么？
2. XMLConfigBuilder#parse() 是如何构造出来一个 Configuration 的？

## 1.Configuration
Configuration 包含了 MyBatis 所有的配置信息，源码如下：
>关于 mybatis-conf.xml  中可以配置什么，可以参考 [【MyBatis】配置文件 mybatis-conf.xml 详解](https://yzx66.blog.csdn.net/article/details/114156634)...

```java
public class Configuration {
  // 数据源
  protected Environment environment;
  
  // 很多设置都是setting标签的，有默认值！！！
  protected boolean safeRowBoundsEnabled;
  protected boolean safeResultHandlerEnabled = true;
  protected boolean mapUnderscoreToCamelCase;
  protected boolean aggressiveLazyLoading;
  protected boolean multipleResultSetsEnabled = true;
  protected boolean useGeneratedKeys;
  protected boolean useColumnLabel = true;
  // 缓存
  protected boolean cacheEnabled = true;
  protected boolean callSettersOnNulls;
  protected boolean useActualParamName = true;
  protected boolean returnInstanceForEmptyRow;

  protected String logPrefix;
  // 日志
  protected Class<? extends Log> logImpl;
  protected Class<? extends VFS> vfsImpl;
  protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
  protected JdbcType jdbcTypeForNull = JdbcType.OTHER;
  protected Integer defaultStatementTimeout;
  protected Integer defaultFetchSize;
  protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
  protected AutoMappingBehavior autoMappingBehavior = AutoMappingBehavior.PARTIAL;

  protected Properties variables = new Properties();
  protected ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
  // ObjectFactory
  protected ObjectFactory objectFactory = new DefaultObjectFactory();
  // ObjectWrapperFactory
  protected ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();
	
  // 懒加载
  protected boolean lazyLoadingEnabled = false;
  // 代理工厂
  protected ProxyFactory proxyFactory = new JavassistProxyFactory(); 

  protected String databaseId;

  protected Class<?> configurationFactory;
  
  // Registrys
  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
  protected final InterceptorChain interceptorChain = new InterceptorChain();
  protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry();
  protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
  protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();

  protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
  protected final Map<String, ResultMap> resultMaps = new StrictMap<>("Result Maps collection");
  protected final Map<String, ParameterMap> parameterMaps = new StrictMap<>("Parameter Maps collection");
  protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<>("Key Generators collection");


  protected final Collection<XMLStatementBuilder> incompleteStatements = new LinkedList<>();
  protected final Collection<CacheRefResolver> incompleteCacheRefs = new LinkedList<>();
  protected final Collection<ResultMapResolver> incompleteResultMaps = new LinkedList<>();
  protected final Collection<MethodResolver> incompleteMethods = new LinkedList<>();

  protected final Map<String, String> cacheRefMap = new HashMap<>();
    
  //...methods
}
```
方无非就两大类 ，读（get）和写（set/add）。set 主要是对于上面懒加载那些只有一个属性的，而 add 第对于那些以map为数据结构的 registry 而言的，比如添加缓存
```java
// 添加缓存到map中
public void addCache(Cache cache) {
    this.caches.put(cache.getId(), cache);
}
// 获取全部缓存
public Collection<Cache> getCaches() {
    return this.caches.values();
}
// 获取指定缓存
public Cache getCache(String id) {
    return (Cache)this.caches.get(id);
}
```

下面我们继续去看第二个问题，XMLConfigBuilder#parse 是如何构造 Configuration 对象的。

## 2.XMLConfigBuilder
我们先来看看 XMLConfigBuilder 的父类 BaseBuilder，

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201216172218605.png?)

```java
// XMLConfigBuilder的抽象父类，专门用来解析全局配置文件
// 另外还有子类XMLMapperBuilder（解析Mapper映射器），XMLStatementBuilder（解析增删改查标签）
public abstract class BaseBuilder {

  // Configuration 是其中一个成员变量
  protected final Configuration configuration;
  
  // 别名库，是构造出的 Configuration 的 typeAliasRegistry 的引用
  // 作用是 typeAlias 标签中的别名关系直接加到这里，不用再调用 configuration.add
  protected final TypeAliasRegistry typeAliasRegistry;
  // 类型处理器库，typeHandler标签中的类型处理器直接加到这里，不用再调用configuration.add
  protected final TypeHandlerRegistry typeHandlerRegistry;
    
  // 构造函数传入Configuration
  public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    // 别名库与类型处理器库实际已经在Configuration对象中的
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
  }
    
  //.......
}
```

### XMLConfigBuilder()
```java
// XMLConfigBuilder构造函数
private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    // 这里先直接创建一个Configuration，然后传入 BaseBuilder
    super(new Configuration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
  }
```
### parse()
解析配置文件构造 Configuration 的入口方法
```java
public Configuration parse() {
    // 配置文件 只解析一次
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    // 对Configuration根据传入的具体配置信息进行解析
    parseConfiguration(parser.evalNode("/configuration"));
    // 返回Configuration
    return configuration;
  }

// 核心逻辑都在下面的方法中
private void parseConfiguration(XNode root) {
  try {
    // 解析Properties标签，读取引入的外部配置文件
    propertiesElement(root.evalNode("properties"));
    // 将 setting 标签解析成 Properties 对象，对于 settings 子标签的处理在后面
    Properties settings = settingsAsProperties(root.evalNode("settings"));
    // 获取Vitual File System的自定义实现类，比如我们要读取本地文件，或者FTP远程文件时，就要用到自定义VFS类。
    loadCustomVfs(settings);
    // 根据 logImpl 标签获取日志的实现类，我们可以用到很多的日志的方案，包括 LOG4J，LOG4J2，SLF4J 等等。
    loadCustomLogImpl(settings);
    // 解析 typeAliases 标签（别名），注册到 TypeAliasRegistry
    typeAliasesElement(root.evalNode("typeAliases"));
    // 解析plugins标签，比如Pagehelper翻页插件，或者自定义插件
    pluginElement(root.evalNode("plugins"));
    // 解析objectFactory标签，生成ObjectFactory对象，设置到Configuration里面
    objectFactoryElement(root.evalNode("objectFactory"));
    // 解析objectWrapperFactory标签，生成ObjectWrapperFactory对象，设置到Configuration里面
    objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
    reflectorFactoryElement(root.evalNode("reflectorFactory"));
    // 处理 settings 的子标签，方法入参是之前解析的 properties对象
    settingsElement(settings);
    // 解析environments标签
    environmentsElement(root.evalNode("environments"));
    databaseIdProviderElement(root.evalNode("databaseIdProvider"));
    // 解析typeHandler标签，将类型处理器注册到 typeHandlerRegistry
    typeHandlerElement(root.evalNode("typeHandlers"));
    // 解析mapper映射关系，保存到mapperRegistry
    mapperElement(root.evalNode("mappers"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
  }
}
```

### propertiesElement()
解析Properties标签，读取引入的外部配置文件
```java
private void propertiesElement(XNode context) throws Exception {
  if (context != null) {
    Properties defaults = context.getChildrenAsProperties();
    // 1.获取配置文件路径
    // 1.1 放在resource目录下，相对路径
    String resource = context.getStringAttribute("resource");
    // 1.2 绝对路径
    String url = context.getStringAttribute("url");
    if (resource != null && url != null) {
      throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
    }
    // 2.将配置信息放到名为defaults的Properties对象里面
    if (resource != null) {
      defaults.putAll(Resources.getResourceAsProperties(resource));
    } else if (url != null) {
      defaults.putAll(Resources.getUrlAsProperties(url));
    }
    Properties vars = configuration.getVariables();
    if (vars != null) {
      defaults.putAll(vars);
    }
    // 3.把XPathParser 和 Configuration 的 Properties 属性都设置成我们填充后的 Properties对象
    parser.setVariables(defaults);    
    configuration.setVariables(defaults);
  }
}
```

### settingsAsProperties()
将 setting 标签解析成 Properties 对象，对于 settings 子标签的处理在后面。
>在早期的版本里面解析和设置都是在后面一起的，这里先解析成Properties对象是因为下面的两个方法要用到。

```java
private Properties settingsAsProperties(XNode context) {
  if (context == null) {
    return new Properties();
  }
  Properties props = context.getChildrenAsProperties();
  // Check that all settings are known to the configuration class
  // 检查配置类是否知道所有设置
  MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
  for (Object key : props.keySet()) {
    if (!metaConfig.hasSetter(String.valueOf(key))) {
      throw new BuilderException("The setting " + key + " is not known.  Make sure you spelled it correctly (case sensitive).");
    }
  }
  return props;
}
```

### settingsElement()
处理 settings 的子标签，方法入参是之前解析的 properties对象。
> PS：如果没有手动设置的属性，这里会设置默认值，然后放入Configuration对象
```java
private void settingsElement(Properties props) {
  configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(
      props.getProperty("autoMappingBehavior", "PARTIAL")));
  configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(
      props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
  configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
  configuration.setProxyFactory((ProxyFactory) createInstance(props.getProperty("proxyFactory")));
  configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
  configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), false));
  configuration.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
  configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
  configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
  configuration.setDefaultExecutorType(ExecutorType.valueOf(props.getProperty("defaultExecutorType", "SIMPLE")));
  configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
  configuration.setDefaultFetchSize(integerValueOf( props.getProperty("defaultFetchSize"), null));
  configuration.setMapUnderscoreToCamelCase(booleanValueOf(props.getProperty("mapUnderscoreToCamelCase"), false));
  configuration.setSafeRowBoundsEnabled(booleanValueOf(props.getProperty("safeRowBoundsEnabled"), false));
  configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));
  configuration.setJdbcTypeForNull(JdbcType.valueOf(props.getProperty("jdbcTypeForNull", "OTHER")));
  configuration.setLazyLoadTriggerMethods(stringSetValueOf(
  	props.getProperty("lazyLoadTriggerMethods"), "equals,clone,hashCode,toString"));
  configuration.setSafeResultHandlerEnabled(booleanValueOf(props.getProperty("safeResultHandlerEnabled"), true));
  configuration.setDefaultScriptingLanguage(resolveClass(props.getProperty("defaultScriptingLanguage")));
  configuration.setDefaultEnumTypeHandler(resolveClass(props.getProperty("defaultEnumTypeHandler")));
  configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
  configuration.setUseActualParamName(booleanValueOf(props.getProperty("useActualParamName"), true));
  configuration.setReturnInstanceForEmptyRow(booleanValueOf(props.getProperty("returnInstanceForEmptyRow"), false));
  configuration.setLogPrefix(props.getProperty("logPrefix"));
  configuration.setConfigurationFactory(resolveClass(props.getProperty("configurationFactory")));
```

### loadCustomVfs()

获取Vitual File System的自定义实现类，比如我们要读取本地文件，或者FTP远程文件时，就要用到自定义VFS类。

```java
// 根据<settings>标签里面的<vfsImpl>标签，生成了一个抽象类 VFS的子类，并且赋值到Configuration中。
private void loadCustomVfs(Properties props) throws ClassNotFoundException {
  String value = props.getProperty("vfsImpl");
  if (value != null) {
    String[] clazzes = value.split(",");
    for (String clazz : clazzes) {
      if (!clazz.isEmpty()) {
        @SuppressWarnings("unchecked")
        Class<? extends VFS> vfsImpl = (Class<? extends VFS>)Resources.classForName(clazz);
        configuration.setVfsImpl(vfsImpl);
      }
    }
  }
}
```

### loadCustomLogImpl()

根据 logImpl 标签获取日志的实现类，我们可以用到很多的日志的方案，包括 LOG4J，LOG4J2，SLF4J 等等。

```java
// 这里生成了一个Log 接口的实现类，并且赋值到Configuration中
private void loadCustomLogImpl(Properties props) {
    Class<? extends Log> logImpl = resolveClass(props.getProperty("logImpl"));
    configuration.setLogImpl(logImpl);
  }
```

### typeAliasesElement()

解析 typeAliases标签（别名），注册到 TypeAliasRegistry

```java
private void typeAliasesElement(XNode parent) {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 1.配置的是一个package，那么就将该包下的所有类都放入TypeAliasRegistry
      if ("package".equals(child.getName())) {
        String typeAliasPackage = child.getStringAttribute("name");
        configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
       // 2.配置的是一个Bean，将其放入TypeAliasRegistry
      } else {
        String alias = child.getStringAttribute("alias");
        String type = child.getStringAttribute("type");
        try {
          Class<?> clazz = Resources.classForName(type);
          if (alias == null) {
            typeAliasRegistry.registerAlias(clazz);
          } else {
            typeAliasRegistry.registerAlias(alias, clazz);
          }
        } catch (ClassNotFoundException e) {
          throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
        }
      }
    }
  }
}
```
<img src="https://img-blog.csdnimg.cn/20201216182302542.png?" width="75%">




### typeHandlerElement()
解析typeHandler标签，将类型处理器注册到 typeHandlerRegistry

跟 TypeAlias 一样，TypeHandler 有两种配置方式，一种是单独配置一个类，一种是指定一个package。最后我们得到的是 JavaType和 JdbcType，以及用来做相互映射的 TypeHandler 之间的映射关系。最后存放在TypeHandlerRegistry 对象里面。

>问题：这种三个对象（Java类型，JDBC类型，Handler）的关系怎么映射？（Map里面再放一个Map）

```java
private void typeHandlerElement(XNode parent) {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 配置方式一：指定一个packeage
      if ("package".equals(child.getName())) {
        String typeHandlerPackage = child.getStringAttribute("name");
        typeHandlerRegistry.register(typeHandlerPackage);
      // 配置方式二：单独指定某个类
      } else {
        String javaTypeName = child.getStringAttribute("javaType");
        String jdbcTypeName = child.getStringAttribute("jdbcType");
        String handlerTypeName = child.getStringAttribute("handler");
        Class<?> javaTypeClass = resolveClass(javaTypeName);
        JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
        Class<?> typeHandlerClass = resolveClass(handlerTypeName);
        if (javaTypeClass != null) {
          if (jdbcType == null) {
            typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
          } else {
            typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
          }
        } else {
          typeHandlerRegistry.register(typeHandlerClass);
        }
      }
    }
  }
}
```
<img src="https://img-blog.csdnimg.cn/20201216183556182.png?" width="75%">


### pluginElement()

解析plugins标签，比如Pagehelper翻页插件，或者自定义插件
```java
// 生成Interceptor对象添加到Configuration的InterceptorChain属性里面
private void pluginElement(XNode parent) throws Exception {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      String interceptor = child.getStringAttribute("interceptor");
      Properties properties = child.getChildrenAsProperties();
      Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
      interceptorInstance.setProperties(properties);
      configuration.addInterceptor(interceptorInstance);
    }
  }
}
```

### objectFactoryElement()
解析objectFactory标签，生成ObjectFactory对象，设置到Configuration里面

```java
private void objectFactoryElement(XNode context) throws Exception {
  if (context != null) {
    String type = context.getStringAttribute("type");
    Properties properties = context.getChildrenAsProperties();
    ObjectFactory factory = (ObjectFactory) resolveClass(type).newInstance();
    factory.setProperties(properties);
    configuration.setObjectFactory(factory);
  }
}
```
### objectWrapperFactoryElement()
解析objectWrapperFactory标签，生成ObjectWrapperFactory对象，设置到Configuration里面
```java
private void objectWrapperFactoryElement(XNode context) throws Exception {
    if (context != null) {
      String type = context.getStringAttribute("type");
      ObjectWrapperFactory factory = (ObjectWrapperFactory) resolveClass(type).newInstance();
      configuration.setObjectWrapperFactory(factory);
    }
  }
```

### environmentsElement()
解析`<environments>`标签

```java
private void environmentsElement(XNode context) throws Exception {
  if (context != null) {
    if (environment == null) {
      environment = context.getStringAttribute("default");
    }
    // 一个environment对应一个数据源
    for (XNode child : context.getChildren()) {
      String id = child.getStringAttribute("id");
      if (isSpecifiedEnvironment(id)) {
        // 根据<transactionManager>创建一个事务工厂!!!
        TransactionFactory txFactory = 
            transactionManagerElement(child.evalNode("transactionManager"));
        // 根据<dataSource>创建一个数据源
        DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
        DataSource dataSource = dsFactory.getDataSource();
        Environment.Builder environmentBuilder = new Environment.Builder(id)
            .transactionFactory(txFactory)
            .dataSource(dataSource);
        configuration.setEnvironment(environmentBuilder.build());
      }
    }
  }
}
```

### mapperElement()
无论是按 package 扫描，还是按接口扫描，最后都会调用到 MapperRegistry 的 addMapper() 方法。**MapperRegistry 里面维护的其实是一个Map容器，存储接口和代理工厂的映射关系**。

<img src="https://img-blog.csdnimg.cn/20201216184113561.png?" width="85%">



```java
private void mapperElement(XNode parent) throws Exception {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      // 1.指定package扫描
      if ("package".equals(child.getName())) {
        String mapperPackage = child.getStringAttribute("name");
        // 解析注解Mapper接口的注解，注册到MapperRegistry
        configuration.addMappers(mapperPackage);
      // 2.单独类扫描
      } else {
        String resource = child.getStringAttribute("resource");
        String url = child.getStringAttribute("url");
        String mapperClass = child.getStringAttribute("class");
        // 2.1 相对路径
        if (resource != null && url == null && mapperClass == null) {
          ErrorContext.instance().resource(resource);
          InputStream inputStream = Resources.getResourceAsStream(resource);
          // 通过XMLMapperBuilder解析，然后注册到MapperRegistry中
          XMLMapperBuilder mapperParser = 
          			new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
          mapperParser.parse();
         // 2.2 绝对路径
        } else if (resource == null && url != null && mapperClass == null) {
          ErrorContext.instance().resource(url);
          InputStream inputStream = Resources.getUrlAsStream(url);
          // 通过XMLMapperBuilder解析，然后注册到MapperRegistry中
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, 
                                                               configuration.getSqlFragments());
          mapperParser.parse();
         // 2.3 指定class对象
        } else if (resource == null && url == null && mapperClass != null) {
          // 只有接口才解析
          Class<?> mapperInterface = Resources.classForName(mapperClass);
          // 解析注解Mapper接口的注解，注册到MapperRegistry
          configuration.addMapper(mapperInterface);
         // 2.4 注册过了，抛异常
        } else {
          throw new BuilderException("A mapper element may only specify a url,
                                     resource or class, but not more than one.");
        }
      }
    }
  }
}
```

**parse()**

```java
// 是对Mapper映射器的解析。
public void parse() {
  if (!configuration.isResourceLoaded(resource)) {
    // 解析所有的子标签 ， 其 中buildStatementFromContext()最终获得MappedStatement对象。
    configurationElement(parser.evalNode("/mapper"));
    configuration.addLoadedResource(resource);
    // 把namespace（接口类型）和工厂类绑定起来。
    bindMapperForNamespace();
  }
}
```

**addMapper()**

```java
// 解析Mapper接口的注解
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // 通过MapperAnnotationBuilder解析注解
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }

public void parse() {
    String resource = type.toString();
    if (!configuration.isResourceLoaded(resource)) {
      loadXmlResource();
      configuration.addLoadedResource(resource);
      assistant.setCurrentNamespace(type.getName());
      // 处理@CacheNamespace 
      parseCache();
      // 处理@CacheNamespaceRef
      parseCacheRef();
      Method[] methods = type.getMethods();
      for (Method method : methods) {
        try {
          if (!method.isBridge()) {
            // parseStatement()方法里面的各种 getAnnotation()，都是对注解的解析，
            // 比如@Options，@SelectKey，@ResultMap等等。
            // 最后同样会解析成 MappedStatement 对象
            // 也就是说在 XML 中配置，和使用注解配置，最后起到一样的效果。
            parseStatement(method);
          }
        } catch (IncompleteElementException e) {
          configuration.addIncompleteMethod(new MethodResolver(this, method));
        }
      }
    }
    parsePendingMethods();
  }
```



## =>总结

1. 在这一步，我们主要完成了config配置文件、Mapper文件、Mapper接口上的注解的解析。
2. 我们得到了一个最重要的对象Configuration，这里面存放了全部的配置信息，它在属性里面还有各种各样的容器。
3. 最后，返回了一个DefaultSqlSessionFactory，里面持有了Configuration的实例。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210224004738540.png?)



