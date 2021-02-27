## 入参为 basePackages 
先看调用时序图：![Component](https://img-blog.csdnimg.cn/img_convert/8f1fa94757628df26ec314a317951e29.png)

```java
public AnnotationConfigApplicationContext(String... basePackages) {
   this();
   scan(basePackages);
   refresh();
}
```

> Spring启动时，会去扫描指定包下的文件。

```java
public void scan(String... basePackages) {
   Assert.notEmpty(basePackages, "At least one base package must be specified");
   this.scanner.scan(basePackages);
}
```

> 对应时序图方法1，ClassPathBeanDefinitionScanner#scan。交给ClassPathBeanDefinitionScanner处理。

ClassPathBeanDefinitionScanner 初始化时设置了注解过滤器

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,Environment environment, @Nullable ResourceLoader resourceLoader) {
	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
	this.registry = registry;
	if (useDefaultFilters) {
    // 注册注解过滤器
		registerDefaultFilters();
	}
	setEnvironment(environment);
	setResourceLoader(resourceLoader);
}
protected void registerDefaultFilters() {
   // 添加Component类型
   this.includeFilters.add(new AnnotationTypeFilter(Component.class));
   ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
   try {
      this.includeFilters.add(new AnnotationTypeFilter(
            ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
   }
   catch (ClassNotFoundException ex) {
   }
   try {
      this.includeFilters.add(new AnnotationTypeFilter(
            ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
   }
   catch (ClassNotFoundException ex) {
   }
}
```

> 在includeFilters添加了Component，ManagedBean两种注解类型。后面用来过滤加载到的class文件是否需要交给Spring容器管理。

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
   Assert.notEmpty(basePackages, "At least one base package must be specified");
   Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
   for (String basePackage : basePackages) {
      // 扫描包下有Spring Component注解，并且生成BeanDefinition
      Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
      for (BeanDefinition candidate : candidates) {
         // 设置scope，默认是singleton
         ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
         candidate.setScope(scopeMetadata.getScopeName());
         String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
         if (candidate instanceof AbstractBeanDefinition) {
            postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
         }
         if (candidate instanceof AnnotatedBeanDefinition) {
            AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
         }
         if (checkCandidate(beanName, candidate)) {
            BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
            // 生成代理类信息
            definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
            beanDefinitions.add(definitionHolder);
            // 注册到Spring容器
            registerBeanDefinition(definitionHolder, this.registry);
         }
      }
   }
   return beanDefinitions;
}
```

> 对应时序图方法2，ClassPathBeanDefinitionScanner#doScan。该方法对包下class文件解析，符合Spring容器管理的类生成BeanDefinition，并注册到容器中。

扫描包下的class文件，把有Component注解的封装BeanDefinition列表返回。

```java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
   if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
      return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
   }
   else {
      return scanCandidateComponents(basePackage);
   }
}
```

> 对应时序图方法3，ClassPathScanningCandidateComponentProvider#findCandidateComponents。

```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
   Set<BeanDefinition> candidates = new LinkedHashSet<>();
   try {
      // classpath*:basePackage/**/*.class
      String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
            resolveBasePackage(basePackage) + '/' + this.resourcePattern;
      // 获取 basePackage 包下的 .class 文件资源
      Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
      for (Resource resource : resources) {
         // 判断是否可读
         if (resource.isReadable()) {
            try {
               // 获取.class文件类信息
               MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
               if (isCandidateComponent(metadataReader)) {
                  ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                  sbd.setResource(resource);
                  sbd.setSource(resource);
                  if (isCandidateComponent(sbd)) {
                     candidates.add(sbd);
                  }
               }
            } catch (Throwable ex) {
               throw new BeanDefinitionStoreException("Failed to read candidate component class: " + resource, ex);
            }
         }
      }
   }
   catch (IOException ex) {
      throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
   }
   return candidates;
}
```

> 对应时序图方法4，ClassPathScanningCandidateComponentProvider#scanCandidateComponents。

```java
public MetadataReader getMetadataReader(Resource resource) throws IOException {
   if (this.metadataReaderCache instanceof ConcurrentMap) {
      // No synchronization necessary...
      MetadataReader metadataReader = this.metadataReaderCache.get(resource);
      if (metadataReader == null) {
         // 获取.class类元信息
         metadataReader = super.getMetadataReader(resource);
         this.metadataReaderCache.put(resource, metadataReader);
      }
      return metadataReader;
   }
   else if (this.metadataReaderCache != null) {
      synchronized (this.metadataReaderCache) {
         MetadataReader metadataReader = this.metadataReaderCache.get(resource);
         if (metadataReader == null) {
            metadataReader = super.getMetadataReader(resource);
            this.metadataReaderCache.put(resource, metadataReader);
         }
         return metadataReader;
      }
   }
   else {
      return super.getMetadataReader(resource);
   }
}
```

> 对应时序图方法5，CachingMetadataReaderFactory#getMetadataReader。  super.getMetadataReader(resource) 调用的是  SimpleMetadataReaderFactory#getMetadataReader。

```java
public MetadataReader getMetadataReader(Resource resource) throws IOException {
   // 默认是SimpleMetadataReader实例
   return new SimpleMetadataReader(resource, this.resourceLoader.getClassLoader());
}
SimpleMetadataReader(Resource resource, @Nullable ClassLoader classLoader) throws IOException {
   // 加载.class文件
   InputStream is = new BufferedInputStream(resource.getInputStream());
   ClassReader classReader;
   try {
      classReader = new ClassReader(is);
   }
   catch (IllegalArgumentException ex) {
      throw new NestedIOException("ASM ClassReader failed to parse class file - " +
            "probably due to a new Java class file version that isn't supported yet: " + resource, ex);
   }
   finally {
      is.close();
   }
   AnnotationMetadataReadingVisitor visitor = new AnnotationMetadataReadingVisitor(classLoader);
   // 解析.class元信息
   classReader.accept(visitor, ClassReader.SKIP_DEBUG);
   this.annotationMetadata = visitor;
   this.classMetadata = visitor;
   this.resource = resource;
}
```

> 对应时序图方法6，SimpleMetadataReader#SimpleMetadataReader。 组装SimpleMetadataReader。

```java
public void accept(
    final ClassVisitor classVisitor,
    final Attribute[] attributePrototypes,
    final int parsingOptions) {
  Context context = new Context();
  context.attributePrototypes = attributePrototypes;
  context.parsingOptions = parsingOptions;
  context.charBuffer = new char[maxStringLength];

  ... 省略代码
    
  // Visit the RuntimeVisibleAnnotations attribute.
  if (runtimeVisibleAnnotationsOffset != 0) {
    int numAnnotations = readUnsignedShort(runtimeVisibleAnnotationsOffset);
    int currentAnnotationOffset = runtimeVisibleAnnotationsOffset + 2;
    while (numAnnotations-- > 0) {
      // Parse the type_index field.
      String annotationDescriptor = readUTF8(currentAnnotationOffset, charBuffer);
      currentAnnotationOffset += 2;
      // 这里面封装Spring Component注解
      currentAnnotationOffset =
          readElementValues(classVisitor.visitAnnotation(annotationDescriptor,true),
              currentAnnotationOffset,true,charBuffer);
    }
  }

 ... 省略代码
}
```

> 对应时序图方法7，ClassReader#accept。该方法把二进制的.class文件解析组装到AnnotationMetadataReadingVisitor

```java
private int readElementValues(
    final AnnotationVisitor annotationVisitor,
    final int annotationOffset,
    final boolean named,
    final char[] charBuffer) {
  ... 省略代码
  if (annotationVisitor != null) {
    // 主要逻辑还在这里面
    annotationVisitor.visitEnd();
  }
  return currentOffset;
}
```

> 对应时序图方法8，ClassReader#readElementValues。

```java
public void visitEnd() {
   super.visitEnd();

   Class<? extends Annotation> annotationClass = this.attributes.annotationType();
   if (annotationClass != null) {
      ... 省略代码
      // 过滤java.lang.annotation包下的注解，及保留Spring注解
      if (!AnnotationUtils.isInJavaLangAnnotationPackage(annotationClass.getName())) {
         try {
            // 获取该类上的所有注解
            Annotation[] metaAnnotations = annotationClass.getAnnotations();
            if (!ObjectUtils.isEmpty(metaAnnotations)) {
               Set<Annotation> visited = new LinkedHashSet<>();
               for (Annotation metaAnnotation : metaAnnotations) {
                  // 过滤java.lang.annotation包下的注解，及保留Spring注解
                  recursivelyCollectMetaAnnotations(visited, metaAnnotation);
               }
               // 封装需要的注解
               if (!visited.isEmpty()) {
                  Set<String> metaAnnotationTypeNames = new LinkedHashSet<>(visited.size());
                  for (Annotation ann : visited) {
                     metaAnnotationTypeNames.add(ann.annotationType().getName());
                  }
                  this.metaAnnotationMap.put(annotationClass.getName(), metaAnnotationTypeNames);
               }
            }
         }
         catch (Throwable ex) {
         }
      }
   }
}
```

> 对应时序图方法9，AnnotationAttributesReadingVisitor#visitEnd。过滤掉 java.lang.annotation 包下的注解，然后把剩下的注解放到metaAnnotationMap。

```java
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
   for (TypeFilter tf : this.excludeFilters) {
      if (tf.match(metadataReader, getMetadataReaderFactory())) {
         return false;
      }
   }
   for (TypeFilter tf : this.includeFilters) {
      if (tf.match(metadataReader, getMetadataReaderFactory())) {
         return isConditionMatch(metadataReader);
      }
   }
   return false;
}
```

> 对应时序图方法10，ClassPathScanningCandidateComponentProvider#isCandidateComponent。使用前面提过的ClassPathBeanDefinitionScanner初始化时设置的注解类型过滤器，includeFilters 包含ManagedBean和Component类型。

```java
public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
      throws IOException {

   if (matchSelf(metadataReader)) {
      return true;
   }

  ... 省略代码

   return false;
}
```

> 对应时序图方法11，AbstractTypeHierarchyTraversingFilter#match。

```java
protected boolean matchSelf(MetadataReader metadataReader) {
   AnnotationMetadata metadata = metadataReader.getAnnotationMetadata();
   return metadata.hasAnnotation(this.annotationType.getName()) ||
         (this.considerMetaAnnotations && metadata.hasMetaAnnotation(this.annotationType.getName()));
}
```

> 对应时序图方法12，AnnotationTypeFilter#matchSelf。判断类的metadata中是否包含Component。

总结@Component到Spring  bean容器管理过程。
* 第一步，初始化时设置了Component类型过滤器；
* 第二步，根据指定扫描包扫描.class文件，生成Resource对象；
* 第三步、解析.class文件并注解归类，生成MetadataReader对象；
* 第四步、使用第一步的注解过滤器过滤出有@Component类；
* 第五步、生成BeanDefinition对象；
* 第六步、把BeanDefinition注册到Spring容器。

以上是@Component注解原理，@Service、@Controller和@Repository上都有@Component修饰，所以原理是一样的。
