SpringIOC 容器对于类级别的注解和类内部的注解分以下两种处理策略：

* 类级别的注解：如@Component、@Repository、@Controller、@Service 以及 JavaEE6 的 @ManagedBean和@Named 注解，都是添加在类上面的类级别注解，Spring容器根据注解的过滤规则扫描读取注解Bean定义类，并将其注册到Spring IOC 容器中。
* 类内部的注解：如@Autowire、@Value、@Resource以及EJB 和 WebService 相关的注解等，都是添加在类内部的字段或者方法上的类内部注解，SpringIOC 容器通过 Bean 后置注解处理器解析Bean内部的注解。

下面将根据这两种处理策略，分别分析Spring 处理注解相关的源码。 

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201208232158636.png#pic_center)
在 Spring 中管理注解 Bean 定义的容器有两个 ：AnnotationConfigApplicationContext 和 AnnotationConfigWebApplicationContex。这两个类是专门处理Spring 注解方式配置的容器，直接依赖于注解作为容器配置信息来源的 IOC 容器。AnnotationConfigWebApplicationContext 是 AnnotationConfigApplicationContext 的 Web版本，两者的用法以及对注解的处理方式几乎没有差别。

## 不同点：AnnotationConfigWebApplicationContext 

AnnotationConfigWebApplicationContext 是 AnnotationConfigApplicationContext 的 Web 版，它们对于注解 Bean 的注册和扫描是基本相同的，但是 AnnotationConfigWebApplicationContext 对注解Bean 定义的载入稍有不同， AnnotationConfigWebApplicationContext 注入注解Bean定义。源码如下：

```java
// 载入注解Bean定义资源
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) {
   // 为容器设置注解Bean定义读取器
   AnnotatedBeanDefinitionReader reader = getAnnotatedBeanDefinitionReader(beanFactory);
   // 为容器设置类路径Bean定义扫描器
   ClassPathBeanDefinitionScanner scanner = getClassPathBeanDefinitionScanner(beanFactory);

   // 获取容器的Bean名称生成器
   BeanNameGenerator beanNameGenerator = getBeanNameGenerator();
   // 为注解Bean定义读取器和类路径扫描器设置Bean名称生成器
   if (beanNameGenerator != null) {
      reader.setBeanNameGenerator(beanNameGenerator);
      scanner.setBeanNameGenerator(beanNameGenerator);
      beanFactory.registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR, beanNameGenerator);
   }

   // 获取容器的作用域元信息解析器
   ScopeMetadataResolver scopeMetadataResolver = getScopeMetadataResolver();
   // 为注解Bean定义读取器和类路径扫描器设置作用域元信息解析器
   if (scopeMetadataResolver != null) {
      reader.setScopeMetadataResolver(scopeMetadataResolver);
      scanner.setScopeMetadataResolver(scopeMetadataResolver);
   }

   if (!this.annotatedClasses.isEmpty()) {
      if (logger.isInfoEnabled()) {
         logger.info("Registering annotated classes: [" +
               StringUtils.collectionToCommaDelimitedString(this.annotatedClasses) + "]");
      }
      reader.register(this.annotatedClasses.toArray(new Class<?>[this.annotatedClasses.size()]));
   }

   if (!this.basePackages.isEmpty()) {
      if (logger.isInfoEnabled()) {
         logger.info("Scanning base packages: [" +
               StringUtils.collectionToCommaDelimitedString(this.basePackages) + "]");
      }
      scanner.scan(this.basePackages.toArray(new String[this.basePackages.size()]));
   }

   // 获取容器定义的Bean定义资源路径
   String[] configLocations = getConfigLocations();
   // 如果定位的Bean定义资源路径不为空
   if (configLocations != null) {
      for (String configLocation : configLocations) {
         try {
            // 使用当前容器的类加载器加载定位路径的字节码类文件
            Class<?> clazz = ClassUtils.forName(configLocation, getClassLoader());
            if (logger.isInfoEnabled()) {
               logger.info("Successfully resolved class for [" + configLocation + "]");
            }
            reader.register(clazz);
         }
         catch (ClassNotFoundException ex) {
            if (logger.isDebugEnabled()) {
               logger.debug("Could not load class for config location [" + configLocation +
                     "] - trying package scan. " + ex);
            }
            // 如果容器类加载器加载定义路径的Bean定义资源失败，则启用容器类路径扫描器扫描给定路径包及其子包中的类
            int count = scanner.scan(configLocation);
            if (logger.isInfoEnabled()) {
               if (count == 0) {
                  logger.info("No annotated classes found for specified class/package [" + configLocation + "]");
               }
               else {
                  logger.info("Found " + count + " annotated classes in package [" + configLocation + "]");
               }
            }
         }
      }
   }
}
```

## 定位Bean扫描路径

现在我们以AnnotationConfigApplicationContext为例看看它的源码：

```java
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {

   // 保存一个读取注解的Bean定义读取器，并将其设置到容器中
   private final AnnotatedBeanDefinitionReader reader;

   // 保存一个扫描指定类路径中注解Bean定义的扫描器，并将其设置到容器中
   private final ClassPathBeanDefinitionScanner scanner;

   // 默认构造函数，初始化一个空容器，容器不包含任何 Bean 信息，需要在稍后通过调用其register()方法注册配置类
   // 并调用refresh()方法刷新容器，触发容器对注解Bean的载入、解析和注册过程
   public AnnotationConfigApplicationContext() {
      this.reader = new AnnotatedBeanDefinitionReader(this);
      this.scanner = new ClassPathBeanDefinitionScanner(this);
   }

   public AnnotationConfigApplicationContext(DefaultListableBeanFactory beanFactory) {
      super(beanFactory);
      this.reader = new AnnotatedBeanDefinitionReader(this);
      this.scanner = new ClassPathBeanDefinitionScanner(this);
   }

   // 最常用的构造函数，通过将涉及到的配置类传递给该构造函数，以实现将相应配置类中的Bean自动注册到容器中
   public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
      this();
      register(annotatedClasses);
      refresh();
   }

   // 该构造函数会自动扫描以给定的包及其子包下的所有类，并自动识别所有的Spring Bean，将其注册到容器中
   public AnnotationConfigApplicationContext(String... basePackages) {
      this();
      scan(basePackages);
      refresh();
   }

   @Override
   public void setEnvironment(ConfigurableEnvironment environment) {
      super.setEnvironment(environment);
      this.reader.setEnvironment(environment);
      this.scanner.setEnvironment(environment);
   }

   // 为容器的注解Bean读取器和注解Bean扫描器设置Bean名称产生器
   public void setBeanNameGenerator(BeanNameGenerator beanNameGenerator) {
      this.reader.setBeanNameGenerator(beanNameGenerator);
      this.scanner.setBeanNameGenerator(beanNameGenerator);
      getBeanFactory().registerSingleton(
            AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR, beanNameGenerator);
   }

   // 为容器的注解Bean读取器和注解Bean扫描器设置作用范围元信息解析器
   public void setScopeMetadataResolver(ScopeMetadataResolver scopeMetadataResolver) {
      this.reader.setScopeMetadataResolver(scopeMetadataResolver);
      this.scanner.setScopeMetadataResolver(scopeMetadataResolver);
   }
    
    //.......
}
```

通过上面的源码分析，我们可以看啊到Spring对注解的处理分为两种方式：

1. 直接将注解Bean注册到容器中

   可以在初始化容器时注册；也可以在容器创建之后手动调用注册方法向容器注册，然后通过手动刷新容器，使得容器对注册的注解Bean进行处理。

2. 通过扫描指定的包及其子包下的所有类
   
   在初始化注解容器时指定要自动扫描的路径，如果容器创建以后向给定路径动态添加了注解Bean，则需要手动调用容器扫描的方法，然后手动刷新容器，使得容器对所注册的Bean进行处理。

接下来，两篇将会对两种处理方式详细分析其实现过程。 

