# 入参为 ConfigureClass 之 ConfigurationClassPostProcessor 处理

postProcessBeanDefinitionRegistry()
```java
	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        int registryId = System.identityHashCode(registry);
        if (this.registriesPostProcessed.contains(registryId)) {
            throw new IllegalStateException("postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
        } else if (this.factoriesPostProcessed.contains(registryId)) {
            throw new IllegalStateException("postProcessBeanFactory already called on this post-processor against " + registry);
        } else {
            this.registriesPostProcessed.add(registryId);
            // 处理 configurationClass 注入问题的
            this.processConfigBeanDefinitions(registry);
        }
    }

	 public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
        List<BeanDefinitionHolder> configCandidates = new ArrayList();
        String[] candidateNames = registry.getBeanDefinitionNames();
        String[] var4 = candidateNames;
        int var5 = candidateNames.length;

        for(int var6 = 0; var6 < var5; ++var6) {
            String beanName = var4[var6];
            BeanDefinition beanDef = registry.getBeanDefinition(beanName);
            // 判断是否有 configureClass
            if (!ConfigurationClassUtils.isFullConfigurationClass(beanDef) && !ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
                    // 有的话加入集合
                    configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
                }
            } else if (this.logger.isDebugEnabled()) {
                this.logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
            }
        }

        if (!configCandidates.isEmpty()) {
            configCandidates.sort((bd1, bd2) -> {
                int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
                int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
                return Integer.compare(i1, i2);
            });
            SingletonBeanRegistry sbr = null;
            if (registry instanceof SingletonBeanRegistry) {
                sbr = (SingletonBeanRegistry)registry;
                if (!this.localBeanNameGeneratorSet) {
                    BeanNameGenerator generator = (BeanNameGenerator)sbr.getSingleton("org.springframework.context.annotation.internalConfigurationBeanNameGenerator");
                    if (generator != null) {
                        this.componentScanBeanNameGenerator = generator;
                        this.importBeanNameGenerator = generator;
                    }
                }
            }

            if (this.environment == null) {
                this.environment = new StandardEnvironment();
            }

            ConfigurationClassParser parser = new ConfigurationClassParser(this.metadataReaderFactory, this.problemReporter, this.environment, this.resourceLoader, this.componentScanBeanNameGenerator, registry);
            Set<BeanDefinitionHolder> candidates = new LinkedHashSet(configCandidates);
            HashSet alreadyParsed = new HashSet(configCandidates.size());

			// 遍历所有的配置类
            do {
            	// 关键步骤
            	// @ComponentScan 就是被 ConfigureClassParser 解析的
            	// @Import 解析、与调用 ImportSelector 接口的 selectImport 也是这里
                parser.parse(candidates);
                parser.validate();
                // 经过 ConfigureClassParser 后有扫描到的配置类
                Set<ConfigurationClass> configClasses = new LinkedHashSet(parser.getConfigurationClasses());
                configClasses.removeAll(alreadyParsed);
                if (this.reader == null) {
                    this.reader = new ConfigurationClassBeanDefinitionReader(registry, this.sourceExtractor, this.resourceLoader, this.environment, this.importBeanNameGenerator, parser.getImportRegistry());
                }

				// 关键步骤，@Bean 与 import 的 resource 都是被 ConfigureClassBeanDefitionReader 解析的
                this.reader.loadBeanDefinitions(configClasses);
                alreadyParsed.addAll(configClasses);
                candidates.clear();
                
                if (registry.getBeanDefinitionCount() > candidateNames.length) {
                    String[] newCandidateNames = registry.getBeanDefinitionNames();
                    Set<String> oldCandidateNames = new HashSet(Arrays.asList(candidateNames));
                    Set<String> alreadyParsedClasses = new HashSet();
                    Iterator var12 = alreadyParsed.iterator();

                    while(var12.hasNext()) {
                        ConfigurationClass configurationClass = (ConfigurationClass)var12.next();
                        alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
                    }

                    String[] var23 = newCandidateNames;
                    int var24 = newCandidateNames.length;

                    for(int var14 = 0; var14 < var24; ++var14) {
                        String candidateName = var23[var14];
                        if (!oldCandidateNames.contains(candidateName)) {
                            BeanDefinition bd = registry.getBeanDefinition(candidateName);
                            if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) && !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                                candidates.add(new BeanDefinitionHolder(bd, candidateName));
                            }
                        }
                    }

                    candidateNames = newCandidateNames;
                }
            } while(!candidates.isEmpty());

            if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
                sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
            }

            if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
                ((CachingMetadataReaderFactory)this.metadataReaderFactory).clearCache();
            }

        }
    }
```

## ConfigurationClassParser.parse()
```java
	public void parse(Set<BeanDefinitionHolder> configCandidates) {
        this.deferredImportSelectors = new LinkedList();
        Iterator var2 = configCandidates.iterator();

        while(var2.hasNext()) {
            BeanDefinitionHolder holder = (BeanDefinitionHolder)var2.next();
            BeanDefinition bd = holder.getBeanDefinition();

            try {
                if (bd instanceof AnnotatedBeanDefinition) {
                	// 处理 @ConpomentScan
                    this.parse(((AnnotatedBeanDefinition)bd).getMetadata(), holder.getBeanName());
                } else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition)bd).hasBeanClass()) {
                    this.parse(((AbstractBeanDefinition)bd).getBeanClass(), holder.getBeanName());
                } else {
                    this.parse(bd.getBeanClassName(), holder.getBeanName());
                }
            } catch (BeanDefinitionStoreException var6) {
                throw var6;
            } catch (Throwable var7) {
                throw new BeanDefinitionStoreException("Failed to parse configuration class [" + bd.getBeanClassName() + "]", var7);
            }
        }
		
		// 处理 DeferredImportSelector 接口（延迟导入）
		// **** 第二个调用 processImports 的地方 *****
        this.processDeferredImportSelectors();
    }
```

### 处理 @ComponentScan
`this.parse(((AnnotatedBeanDefinition)bd).getMetadata(), holder.getBeanName())` 会走到：
```java
	protected final ConfigurationClassParser.SourceClass doProcessConfigurationClass(ConfigurationClass configClass, ConfigurationClassParser.SourceClass sourceClass) throws IOException {
        this.processMemberClasses(configClass, sourceClass);
        Iterator var3 = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(), PropertySources.class, PropertySource.class).iterator();

        AnnotationAttributes importResource;
        while(var3.hasNext()) {
            importResource = (AnnotationAttributes)var3.next();
            if (this.environment instanceof ConfigurableEnvironment) {
                this.processPropertySource(importResource);
            } else {
                this.logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() + "]. Reason: Environment must implement ConfigurableEnvironment");
            }
        }
		
		// 处理 @ConpomentScan
        Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
        if (!componentScans.isEmpty() && !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
            Iterator var13 = componentScans.iterator();

            while(var13.hasNext()) {
                AnnotationAttributes componentScan = (AnnotationAttributes)var13.next();
                // 使用 componentScanParser 扫描
                // 扫描的过程中会注册 beanDefition
                Set<BeanDefinitionHolder> scannedBeanDefinitions = this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
                Iterator var7 = scannedBeanDefinitions.iterator();

                while(var7.hasNext()) {
                    BeanDefinitionHolder holder = (BeanDefinitionHolder)var7.next();
                    BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                    if (bdCand == null) {
                        bdCand = holder.getBeanDefinition();
                    }

					// 如果扫描到 ConfigurationClass，那么继续递归调用 ConfigurationClassParser 解析
					// 也就是说每一个 ConfigurationClass 都会经过 ConfigurationClassParser 的解析
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                        this.parse(bdCand.getBeanClassName(), holder.getBeanName());
                    }
                }
            }
        }
		
		// this.getImports(sourceClass) 会拿到当前配置类所有 import 的类
		// **** 第一个调用 processImports 的地方 *****
        this.processImports(configClass, sourceClass, this.getImports(sourceClass), true);
        importResource = AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        if (importResource != null) {
            String[] resources = importResource.getStringArray("locations");
            Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
            String[] var19 = resources;
            int var21 = resources.length;

            for(int var22 = 0; var22 < var21; ++var22) {
                String resource = var19[var22];
                String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
                configClass.addImportedResource(resolvedResource, readerClass);
            }
        }

        Set<MethodMetadata> beanMethods = this.retrieveBeanMethodMetadata(sourceClass);
        Iterator var17 = beanMethods.iterator();

        while(var17.hasNext()) {
            MethodMetadata methodMetadata = (MethodMetadata)var17.next();
            configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
        }

        this.processInterfaces(configClass, sourceClass);
        if (sourceClass.getMetadata().hasSuperClass()) {
            String superclass = sourceClass.getMetadata().getSuperClassName();
            if (superclass != null && !superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
                this.knownSuperclasses.put(superclass, configClass);
                return sourceClass.getSuperClass();
            }
        }

        return null;
    }
```
#### ComponentScanParser
parse
```java
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
    ...
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
				componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);
		...
		for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
			for (TypeFilter typeFilter : typeFiltersFor(filter)) {
				scanner.addIncludeFilter(typeFilter);
			}
		}
		for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
			for (TypeFilter typeFilter : typeFiltersFor(filter)) {
				scanner.addExcludeFilter(typeFilter);
			}
		}

		boolean lazyInit = componentScan.getBoolean("lazyInit");
		if (lazyInit) {
			scanner.getBeanDefinitionDefaults().setLazyInit(true);
		}

		Set<String> basePackages = new LinkedHashSet<String>();
		String[] basePackagesArray = componentScan.getStringArray("basePackages");
		for (String pkg : basePackagesArray) {
			String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
					ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
			basePackages.addAll(Arrays.asList(tokenized));
		}
		for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
			basePackages.add(ClassUtils.getPackageName(clazz));
		}
		if (basePackages.isEmpty()) {
			basePackages.add(ClassUtils.getPackageName(declaringClass));
		}

		scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
			@Override
			protected boolean matchClassName(String className) {
				return declaringClass.equals(className);
			}
		});
		// 会扫描，然后注册 BeanDefition
		return scanner.doScan(StringUtils.toStringArray(basePackages));
	}
```
doScan
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
### 处理 ImportSelector 接口

```java
	private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
            Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

        if (importCandidates.isEmpty()) {
            return;
        }

        if (checkForCircularImports && isChainedImportOnStack(configClass)) {
            this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
        }
        else {
            this.importStack.push(configClass);
            try {
                for (SourceClass candidate : importCandidates) {
　　　　　　　　　　　　//对ImportSelector的处理
                    if (candidate.isAssignable(ImportSelector.class)) {
                        // Candidate class is an ImportSelector -> delegate to it to determine imports
                        Class<?> candidateClass = candidate.loadClass();
                        ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                        // 先调用 aware
                        ParserStrategyUtils.invokeAwareMethods(
                                selector, this.environment, this.resourceLoader, this.registry);
                        if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
　　　　　　　　　　　　　　　　//如果为延迟导入处理则加入集合当中
                            this.deferredImportSelectors.add(
                                    new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
                        }
                        else {
　　　　　　　　　　　　　　　　//根据ImportSelector方法的返回值来进行递归操作
                            String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                            Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                            processImports(configClass, currentSourceClass, importSourceClasses, false);
                        }
                    }
                    else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                        // Candidate class is an ImportBeanDefinitionRegistrar ->
                        // delegate to it to register additional bean definitions
                        Class<?> candidateClass = candidate.loadClass();
                        ImportBeanDefinitionRegistrar registrar =
                                BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                        ParserStrategyUtils.invokeAwareMethods(
                                registrar, this.environment, this.resourceLoader, this.registry);
                        configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                    }
                    else {
　　　　　　　　　　　　　　// 如果当前的类既不是ImportSelector也不是ImportBeanDefinitionRegistar就进行@Configuration的解析处理
                        // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                        // process it as an @Configuration class
                        this.importStack.registerImport(
                                currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                        processConfigurationClass(candidate.asConfigClass(configClass));
                    }
                }
            }
            catch (BeanDefinitionStoreException ex) {
                throw ex;
            }
            catch (Throwable ex) {
                throw new BeanDefinitionStoreException(
                        "Failed to process import candidates for configuration class [" +
                        configClass.getMetadata().getClassName() + "]", ex);
            }
            finally {
                this.importStack.pop();
            }
        }
    }
```
## ConfigurationClassBeanDefinitionReader.loadBeanDefinitions()
reader.loadBeanDefinitions(configClasses) 会走到这
```java
	public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
        ConfigurationClassBeanDefinitionReader.TrackedConditionEvaluator trackedConditionEvaluator = new ConfigurationClassBeanDefinitionReader.TrackedConditionEvaluator();
        Iterator var3 = configurationModel.iterator();

        while(var3.hasNext()) {
            ConfigurationClass configClass = (ConfigurationClass)var3.next();
            this.loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
        }

    }

    private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass, ConfigurationClassBeanDefinitionReader.TrackedConditionEvaluator trackedConditionEvaluator) {
    	// 如果要 skip，那么之前在 ConpomentScanParser 中即使扫到保存到了 BeanDefitionRegistry 也会被移除
    	// skip 的条件是有 @Conditional 注解，且不满足 Condition 的条件
        if (trackedConditionEvaluator.shouldSkip(configClass)) {
            String beanName = configClass.getBeanName();
            if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
                this.registry.removeBeanDefinition(beanName);
            }

            this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
        } else {
            if (configClass.isImported()) {
                this.registerBeanDefinitionForImportedConfigurationClass(configClass);
            }

            Iterator var3 = configClass.getBeanMethods().iterator();

            while(var3.hasNext()) {
                BeanMethod beanMethod = (BeanMethod)var3.next();
                // 解析 @Bean，会遍历所有的 method
                this.loadBeanDefinitionsForBeanMethod(beanMethod);
            }

			// 解析 import 的配置文件
            this.loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());                
			this.loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
        }
    }
```
