我们在平时使用 Spring+MyBatis 时都是先配置 mapper 接口扫描

```xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <!-- 扫描所有dao接口 -->
    <property name="basePackage" value="com.xupt.yzh.dao"></property>
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"></property>
</bean>
```

然后在使用时直接 @Autowired 注入Mapper接口

```java
@Service 
public class EmployeeService { 
    
    @Autowired 
    // 直接注入mapper接口
    // 既没有手动编写实现类，也没有在使用时传入statemetID
    EmployeeMapper employeeMapper;
    
	public List<Employee> getAll() { 
        return employeeMapper.selectByMap(null);
    }
}
```
既然能够注入，那么它肯定需要先注册到 IOC 容器中（比如XmlWebApplicationContext)。

>注册到 IOC 容器的意思是，为 bean 创建 BeanDefinition 并添加到 beanDefinitionMap 中。只有注册过的 bean，才能被 IOC 容器管理，才能被实例化，才能被依赖注入。而 Spring 只能处理具体的类并不能够处理接口，所以 MyBatis 需要在 Spring 启动时扫描这些 Mapper 接口，然后自己实现 bean 注册。

那么问题来了，MyBatis 中到底如何实现自定义注册？注册的是什么？这还是得从 MapperScannerConfigurer 的源码来看起啊...

### MapperScannerConfigurer
MapperScannerConfigurer 实现了 BeanDefinitionRegistryPostProcessor 接口，继承关系如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201218194215759.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNTkyNw==,size_16,color_FFFFFF,t_70#pic_center)
它这里实现 BeanDefinitionRegistryPostProcessor 有啥用呢？

BeanDefinitionRegistryPostProcessor 是 BeanFactoryPostProcessor 的子类，是**Spring 的扩展点**之一，可以通过编码的方式修改、新增或者删除某些 Bean 的定义（BeanDefinition）。所以 MyBatis 就可以自己实现 bean 的注册了啊！

我们需要做的就是重写 postProcessBeanDefinitionRegistry() 方法，在这里面操作 bean 

```java
public class MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor, 
	InitializingBean, ApplicationContextAware, BeanNameAware {
	
     @Override
     // BeanDefinitionRegistry 提供了丰富的方法去操作 BeanDefiniton，包括注册，移除，判断...
     public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        if (this.processPropertyPlaceHolders) {
          processPropertyPlaceHolders();
        }

        ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
        scanner.setAddToConfig(this.addToConfig);
        scanner.setAnnotationClass(this.annotationClass);
        scanner.setMarkerInterface(this.markerInterface);
        scanner.setSqlSessionFactory(this.sqlSessionFactory);
        scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
        scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
        scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
        scanner.setResourceLoader(this.applicationContext);
        scanner.setBeanNameGenerator(this.nameGenerator);
        scanner.registerFilters();
        
        // scanner.scan()方法是 ClassPathBeanDefinitionScanner 中的，
        // 而它的子类 ClassPathMapperScanner 覆盖了 doScan() 方法 ，
        // ClassPathBeanDefinitionScanner#scan --> ClassPathMapperScanner#doscan --> ClassPathBeanDefinitionScanner#doscan 
        scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
      }
        
      //.......
}
```
### ClassPathBeanDefinitionScanner.scan()

```java
// 在指定的基本包下扫描
// 返回注册的 bean 数
public int scan(String... basePackages) {
   int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
	
   // 实际是调用子类 ClassPathMapperScanner#doScan
   doScan(basePackages);

   // Register annotation config processors, if necessary.
   if (this.includeAnnotationConfig) {
      AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
   }

   return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}
```
### ClassPathMapperScanner.doScan()

```java
@Override
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
  // 它先调用父类 ClassPathBeanDefinitionScanner#doScan() 扫描所有的接口。
  Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);
	
  if (beanDefinitions.isEmpty()) {
    LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)  + "' package. Please check your configuration.");
  } else {
    processBeanDefinitions(beanDefinitions);
  }

  return beanDefinitions;
}
```
### ClassPathBeanDefinitionScanner.doScan()
```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
   Assert.notEmpty(basePackages, "At least one base package must be specified");
   Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
   for (String basePackage : basePackages) {
   	  // 通过文件找到符合规则的 BeanDefinition
      Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
      // 处理BeanDefinition
      for (BeanDefinition candidate : candidates) {
      	 // 从definition的Annotated获取Scope注解，根据scope注解获取ScopeMetadata
         ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
         candidate.setScope(scopeMetadata.getScopeName());
         // 生成beanName,从Commponent，或者javax.annotation.ManagedBean、javax.inject.Named的value取，或者系统默认生成
         String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
         // 设置beanDefinition的默认属性，设置是否参与自动注入
         if (candidate instanceof AbstractBeanDefinition) {
            postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
         }
         // 通过注解的MetaData设置属性，用来覆盖默认属性如 lazyInit,Primary,DependsOn,Role,Description属性
         if (candidate instanceof AnnotatedBeanDefinition) {
            AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
         }
         // 注册BeanDefinition
      	 // 1.!this.registry.containsBeanDefinition(beanName)
         // 2.如果beanName存在，beanDefinition和existingDef兼容，说明不用再次注册
         // （不是ScannedGenericBeanDefinition or source相同 or beanDefinition==existingDefinition）
         // 3.如果beanName存在，还不兼容抛异常ConflictingBeanDefinitionException
         if (checkCandidate(beanName, candidate)) {
         	// 创建BeanDefinitionHolder
            BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
            definitionHolder =
                  AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
            // 放到beanDefinitions结果集
            beanDefinitions.add(definitionHolder);
            // 注册到registry中
            registerBeanDefinition(definitionHolder, this.registry);
         }
      }
   }
   return beanDefinitions;
}
```

### ClassPathMapperScanner.processBeanDefinitions()

```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();
      String beanClassName = definition.getBeanClassName();
      LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName()+ "' and '" + beanClassName + "' mapperInterface");
	  
      // 在注册 beanDefinitions 的时候，BeanClass被改为 MapperFactoryBean。
      // 原因：MapperFactoryBean 继承了 SqlSessionDaoSupport ，可以拿 SqlSessionTemplate。
      definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); 
      definition.setBeanClass(this.mapperFactoryBean.getClass());

      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      boolean explicitFactoryUsed = false;
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }

      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          LOGGER.warn(() -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together.sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          LOGGER.warn(() -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }

      if (!explicitFactoryUsed) {
        LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
    }
  }

```
到此为止 Mapper 接口的注册工作算是完成了，BeanDefinition 里面保存的 BeanClass 是 MapperFactoryBean.class，因为 MapperFactoryBean 继承了 SqlSessionDaoSupport，所以它可以拿到最关键的 SqlSessionTemplate（在上一篇分析过）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201219003634851.png?)

有了 BeanClass 就可以通过反射去创建实例对象；有了 BeanDefinition，就有了创建 bean 的依据，在 Spring 启动时 IOC 容器就可以通过 getBean 创建 bean 实例；有了 bean 实例，IOC 容器就可以将它依赖注入给上层的 ~Service 等 bean。

> 其实还有两个问题没解决：
> 1. 我们只是注入了一个接口，在对象实例化的时候，是怎么拿到 SqlSessionTemplate的？
> 2. 当我们调用方法的时候，还是不是用的 MapperProxy？
>
><BR>分析请看后两篇文章...
