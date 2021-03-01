# 入参为 ConfigureClass 之注册配置类

当创建注解处理容器时，如果传入的初始参数是具体的注解Bean定义类时，注解容器读取并注册。

```java
// 注册多个注解Bean定义类
public void register(Class<?>... annotatedClasses) {
   for (Class<?> annotatedClass : annotatedClasses) {
      registerBean(annotatedClass);
   }
}

// 注册一个注解Bean定义类
public void registerBean(Class<?> annotatedClass) {
   doRegisterBean(annotatedClass, null, null, null);
}

public <T> void registerBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier) {
   doRegisterBean(annotatedClass, instanceSupplier, null, null);
}

public <T> void registerBean(Class<T> annotatedClass, String name, @Nullable Supplier<T> instanceSupplier) {
   doRegisterBean(annotatedClass, instanceSupplier, name, null);
}

// Bean定义读取器注册注解Bean定义的入口方法
@SuppressWarnings("unchecked")
public void registerBean(Class<?> annotatedClass, Class<? extends Annotation>... qualifiers) {
   doRegisterBean(annotatedClass, null, null, qualifiers);
}

// Bean定义读取器向容器注册注解Bean定义类
@SuppressWarnings("unchecked")
public void registerBean(Class<?> annotatedClass, String name, Class<? extends Annotation>... qualifiers) {
   doRegisterBean(annotatedClass, null, name, qualifiers);
}
```

doRegisterBean
```java
// Bean定义读取器向容器注册注解Bean定义类
<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
      @Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {

   // 根据指定的注解Bean定义类，创建Spring容器中对注解Bean的封装的数据结构
   AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
   if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
      return;
   }

   abd.setInstanceSupplier(instanceSupplier);
   // 解析注解Bean定义的作用域，
   // 若@Scope("prototype")，则Bean为原型类型；若@Scope("singleton")，则Bean为单态类型
   ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
   // 为注解Bean定义设置作用域
   abd.setScope(scopeMetadata.getScopeName());
   // 为注解Bean定义生成Bean名称
   String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

   // 处理注解Bean定义中的通用注解
   AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
   // 如果在向容器注册注解Bean定义时，使用了额外的限定符注解，则解析限定符注解。
   // 主要是配置的关于autowiring自动依赖注入装配的限定条件，即@Qualifier注解
   // Spring自动依赖注入装配默认是按类型装配，如果使用@Qualifier则按名称
   if (qualifiers != null) {
      for (Class<? extends Annotation> qualifier : qualifiers) {
         // 如果配置了@Primary注解，设置该Bean为autowiring自动依赖注入装//配时的首选
         if (Primary.class == qualifier) {
            abd.setPrimary(true);
         }
         // 如果配置了@Lazy注解，则设置该Bean为非延迟初始化，如果没有配置，则该Bean为预实例化
         else if (Lazy.class == qualifier) {
            abd.setLazyInit(true);
         }
         // 如果使用了除@Primary和@Lazy以外的其他注解，则为该Bean添加一个autowiring自动依赖注入装配限定符，
         // 该Bean在进autowiring自动依赖注入装配时，根据名称装配限定符指定的Bean
         else {
            abd.addQualifier(new AutowireCandidateQualifier(qualifier));
         }
      }
   }
   for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
      customizer.customize(abd);
   }

   // 创建一个指定Bean名称的Bean定义对象，封装注解Bean定义类数据
   BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
   // 根据注解Bean定义类中配置的作用域，创建相应的代理对象
   definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
   // 向IOC容器注册注解Bean类定义对象
   BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```
## 注册配置类四步

从上面的源码我们可以看出，注册注解Bean定义类的基本步骤：

1. 需要使用注解元数据解析器解析注解Bean中关于作用域的配置
2. 使用 AnnotationConfigUtils 的 processCommonDefinitionAnnotations()方法处理注解 Bean 定义类中通用的注解
3. 使用AnnotationConfigUtils的applyScopedProxyMode()方法创建对于作用域的代理对象
4. 通过BeanDefinitionReaderUtils向容器注册Bean（只是这个配置类的 bean）

下面我们继续分析这4步的具体实现过程。

### 1. AnnotationScopeMetadataResolver解析作用域元数据

AnnotationScopeMetadataResolver 通过 resolveScopeMetadata() 方法解析注解 Bean 定义类的作用域元信息，即判断注册的Bean是原生类型(prototype)还是单态(singleton)类型，其源码如下：

```java
// 解析注解Bean定义类中的作用域元信息
@Override
public ScopeMetadata resolveScopeMetadata(BeanDefinition definition) {
   ScopeMetadata metadata = new ScopeMetadata();
   if (definition instanceof AnnotatedBeanDefinition) {
      AnnotatedBeanDefinition annDef = (AnnotatedBeanDefinition) definition;
      // 从注解Bean定义类的属性中查找属性为”Scope”的值，即@Scope注解的值
      // annDef.getMetadata().getAnnotationAttributes()方法将Bean中所有的注解和注解的值存放在一个map集合中
      AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(
            annDef.getMetadata(), this.scopeAnnotationType);
      // 将获取到的@Scope注解的值设置到要返回的对象中
      if (attributes != null) {
         metadata.setScopeName(attributes.getString("value"));
         // 获取@Scope注解中的proxyMode属性值，在创建代理对象时会用到
         ScopedProxyMode proxyMode = attributes.getEnum("proxyMode");
         // 如果@Scope的proxyMode属性为DEFAULT或者NO
         if (proxyMode == ScopedProxyMode.DEFAULT) {
            //设置proxyMode为NO
            proxyMode = this.defaultProxyMode;
         }
         // 为返回的元数据设置proxyMode
         metadata.setScopedProxyMode(proxyMode);
      }
   }
   // 返回解析的作用域元信息对象
   return metadata;
}
```

上述代码中的 annDef.getMetadata().getAnnotationAttributes()方法就是获取对象中指定类型的注解的值。

### 2. AnnotationConfigUtils处理注解Bean定义类中的通用注解

AnnotationConfigUtils 类的 processCommonDefinitionAnnotations()在向容器注册 Bean 之前，首先对注解Bean定义类中的通用Spring 注解进行处理，源码如下：

```java
// 处理Bean定义中通用注解
static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
   AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
   // 如果Bean定义中有@Lazy注解，则将该Bean预实例化属性设置为@lazy注解的值
   if (lazy != null) {
      abd.setLazyInit(lazy.getBoolean("value"));
   }

   else if (abd.getMetadata() != metadata) {
      lazy = attributesFor(abd.getMetadata(), Lazy.class);
      if (lazy != null) {
         abd.setLazyInit(lazy.getBoolean("value"));
      }
   }
   // 如果Bean定义中有@Primary注解，则为该Bean设置为autowiring自动依赖注入装配的首选对象
   if (metadata.isAnnotated(Primary.class.getName())) {
      abd.setPrimary(true);
   }
   // 如果Bean定义中有@ DependsOn注解，则为该Bean设置所依赖的Bean名称，
   // 容器将确保在实例化该Bean之前首先实例化所依赖的Bean
   AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
   if (dependsOn != null) {
      abd.setDependsOn(dependsOn.getStringArray("value"));
   }

   if (abd instanceof AbstractBeanDefinition) {
      AbstractBeanDefinition absBd = (AbstractBeanDefinition) abd;
      AnnotationAttributes role = attributesFor(metadata, Role.class);
      if (role != null) {
         absBd.setRole(role.getNumber("value").intValue());
      }
      AnnotationAttributes description = attributesFor(metadata, Description.class);
      if (description != null) {
         absBd.setDescription(description.getString("value"));
      }
   }
}
```

### 3. AnnotationConfigUtils根据注解Bean定义类中配置的作用域为其应用相应的代理策略

AnnotationConfigUtils 类的 applyScopedProxyMode()方法根据注解 Bean 定义类中配置的作用域@Scope注解的值，为Bean定义应用相应的代理模式，主要是在Spring 面向切面编程(AOP)中使用。源码如下：

```java
// 根据作用域为Bean应用引用的代码模式
static BeanDefinitionHolder applyScopedProxyMode(
      ScopeMetadata metadata, BeanDefinitionHolder definition, BeanDefinitionRegistry registry) {

   // 获取注解Bean定义类中@Scope注解的proxyMode属性值
   ScopedProxyMode scopedProxyMode = metadata.getScopedProxyMode();
   // 如果配置的@Scope注解的proxyMode属性值为NO，则不应用代理模式
   if (scopedProxyMode.equals(ScopedProxyMode.NO)) {
      return definition;
   }
   // 获取配置的@Scope注解的proxyMode属性值，如果为TARGET_CLASS，则返回true，如果为INTERFACES，则返回false
   boolean proxyTargetClass = scopedProxyMode.equals(ScopedProxyMode.TARGET_CLASS);
   // 为注册的Bean创建相应模式的代理对象
   return ScopedProxyCreator.createScopedProxy(definition, registry, proxyTargetClass);
}
```
这段为Bean引用创建相应模式的代理，这里不做深入的分析。

### 4. BeanDefinitionReaderUtils向容器注册Bean

BeanDefinitionReaderUtils 主要是校验 BeanDefinition 信息，然后将 Bean 添加到容器中一个管理BeanDefinition的HashMap中



