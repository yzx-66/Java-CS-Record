```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
   <!-- mybatis全局配置文件 -->
   <property name="configLocation" value="classpath:mybatis-config.xml"></property>
   <!-- 数据源 -->
   <property name="dataSource" ref="pooledDataSource"></property>
   <!-- mapper文件 -->
   <property name="mapperLocations" value="classpath:mapper/*.xml"></property>
</bean>

```

**SqlSessionFactoryBean**

SqlSessionFactoryBean 从名字就能看出来它是用来创建工厂类的，继承关系如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201218124413486.png?)
其中，InitializingBean 接口为 bean 提供了初始化方法的方式，它只包括 afterPropertiesSet() 方法，凡是继承该接口的类，在 bean 的属性值设置完的时候会自动执行该方法。

> PS：关于 InitializingBean 可以参考[这篇文章](https://blog.csdn.net/maclaren001/article/details/37039749)...

```java
@Override
public void afterPropertiesSet() throws Exception {
  // 检查标签属性dataSource，SqlSessionFactoryBuilder
  notNull(dataSource, "Property 'dataSource' is required");
  notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
  state((configuration == null && configLocation == null) || !(configuration != null && configLocation != null),
            "Property 'configuration' and 'configLocation' can not specified with together");
    
  // 创建会话工厂 SqlSessionFactory
  this.sqlSessionFactory = buildSqlSessionFactory();
}
```
另外，它还实现了 FactoryBean 接口，所以 getBean() 获取 SqlSessionFactory 实例的时候，实际上是调用 FactoryBean#getObject() 方法。可以看到，它里面调用的也是 afterPropertiesSet() 方法。



```java
@Override
public SqlSessionFactory getObject() throws Exception {
  if (this.sqlSessionFactory == null) {
    afterPropertiesSet();
  }

  return this.sqlSessionFactory;
}
```
**buildSqlSessionFactory()**

```java
protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
  // 定义了一个Configuration，叫做targetConfiguration。
  final Configuration targetConfiguration;

  XMLConfigBuilder xmlConfigBuilder = null;
  // 判断 Configuration 对象是否已经存在，也就是是否已经解析过。如果已经有对象，就覆盖一下属性
  if (this.configuration != null) {
    targetConfiguration = this.configuration;
    if (targetConfiguration.getVariables() == null) {
      targetConfiguration.setVariables(this.configurationProperties);
    } else if (this.configurationProperties != null) {
      targetConfiguration.getVariables().putAll(this.configurationProperties);
    }
    // 如果 Configuration 不存在，但是配置了 configLocation 属性，
    // 就根据mybatis-config.xml的文件路径，构建一个xmlConfigBuilder对象。
  } else if (this.configLocation != null) {
    xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
    targetConfiguration = xmlConfigBuilder.getConfiguration();
    // 否则，Configuration 对象不存在，configLocation 路径也没有，
    // 只能使用默认属性去构建去给configurationProperties赋值。
  } else {
    LOGGER.debug(() -> "Property 'configuration' or 'configLocation' not specified,using default MyBatis Configuration");
    targetConfiguration = new Configuration();
    Optional.ofNullable(this.configurationProperties).ifPresent(targetConfiguration::setVariables);
  }
  
  // 基于当前factory 对象里面已有的属性，对targetConfiguration对象里面属性的赋值。
  Optional.ofNullable(this.objectFactory).ifPresent(targetConfiguration::setObjectFactory);
  Optional.ofNullable(this.objectWrapperFactory).
                 ifPresent(targetConfiguration::setObjectWrapperFactory);
  Optional.ofNullable(this.vfs).ifPresent(targetConfiguration::setVfsImpl);

  if (hasLength(this.typeAliasesPackage)) {
    String[] typeAliasPackageArray = tokenizeToStringArray(this.typeAliasesPackage,
        ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
    Stream.of(typeAliasPackageArray).forEach(packageToScan -> {
      targetConfiguration.getTypeAliasRegistry().registerAliases(packageToScan,
          typeAliasesSuperType == null ? Object.class : typeAliasesSuperType);
      LOGGER.debug(() -> "Scanned package: '" + packageToScan + "' for aliases");
    });
  }

  if (!isEmpty(this.typeAliases)) {
    Stream.of(this.typeAliases).forEach(typeAlias -> {
      targetConfiguration.getTypeAliasRegistry().registerAlias(typeAlias);
      LOGGER.debug(() -> "Registered type alias: '" + typeAlias + "'");
    });
  }

  if (!isEmpty(this.plugins)) {
    Stream.of(this.plugins).forEach(plugin -> {
      targetConfiguration.addInterceptor(plugin);
      LOGGER.debug(() -> "Registered plugin: '" + plugin + "'");
    });
  }

  if (hasLength(this.typeHandlersPackage)) {
    String[] typeHandlersPackageArray = tokenizeToStringArray(this.typeHandlersPackage,
        ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
    Stream.of(typeHandlersPackageArray).forEach(packageToScan -> {
      targetConfiguration.getTypeHandlerRegistry().register(packageToScan);
      LOGGER.debug(() -> "Scanned package: '" + packageToScan + "' for type handlers");
    });
  }

  if (!isEmpty(this.typeHandlers)) {
    Stream.of(this.typeHandlers).forEach(typeHandler -> {
      targetConfiguration.getTypeHandlerRegistry().register(typeHandler);
      LOGGER.debug(() -> "Registered type handler: '" + typeHandler + "'");
    });
  }

  if (this.databaseIdProvider != null) {//fix #64 set databaseId before parse mapper xmls
    try {
      targetConfiguration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
    } catch (SQLException e) {
      throw new NestedIOException("Failed getting a databaseId", e);
    }
  }

  Optional.ofNullable(this.cache).ifPresent(targetConfiguration::addCache);
  // 如果xmlConfigBuilder 不为空，也就是上面的第二种情况，
  if (xmlConfigBuilder != null) {
    try {
      // 调用了xmlConfigBuilder.parse()去解析配置文件，最终会返回解析好的Configuration对象
      xmlConfigBuilder.parse();
      LOGGER.debug(() -> "Parsed configuration file: '" + this.configLocation + "'");
    } catch (Exception ex) {
      throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
    } finally {
      ErrorContext.instance().reset();
    }
  }

  // 如果没有明确指定事务工厂 ，默认使用pringManagedTransactionFactory。
  // 它创建的 SpringManagedTransaction 也有getConnection()和close()方法
  // <property name="transactionFactory" value="" />
  targetConfiguration.setEnvironment(new Environment(this.environment,
      this.transactionFactory == null ? new SpringManagedTransactionFactory() : this.transactionFactory,this.dataSource));

  if (!isEmpty(this.mapperLocations)) {
    for (Resource mapperLocation : this.mapperLocations) {
      if (mapperLocation == null) {
        continue;
      }

      try {
        XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
         targetConfiguration, mapperLocation.toString(), targetConfiguration.getSqlFragments());
        // 调用xmlMapperBuilder.parse()，
        // 它的作用是把接口和对应的MapperProxyFactory 注册到MapperRegistry 中。
        xmlMapperBuilder.parse();
      } catch (Exception e) {
        throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + e);
      } finally {
        ErrorContext.instance().reset();
      }
      LOGGER.debug(() -> "Parsed mapper file: '" + mapperLocation + "'");
    }
  } else {
    LOGGER.debug(() -> "Property 'mapperLocations' was not specified or no matching resources found");
  }

  // 最后调用 sqlSessionFactoryBuilder.build() 返回一个 DefaultSqlSessionFactory。
  return this.sqlSessionFactoryBuilder.build(targetConfiguration);
}
```
可以看到最终返回一个 SqlSessionFactory 的默认实现 DefaultSqlSessionFactory。

>最后再梳理一下 IOC 容器 getBean() 获取 SqlSessionFactory 的调用链路：
getBean() --> SqlSessionFactoryBean#getObject() --> afterPropertiesSet() --> buildSqlSessionFactory() ==> DefaultSqlSessionFactory









