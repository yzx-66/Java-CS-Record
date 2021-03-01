# 条件注解
条件注解是Spring4提供的一种bean加载特性，主要用于控制配置类和bean初始化条件。在springBoot，springCloud一系列框架底层源码中，条件注解的使用到处可见。

不少人在使用@ConditionalOnBean注解时会遇到不生效的情况，依赖的 bean 明明已经配置了，但就是不生效。到底@ConditionalOnBean和bean加载的顺序有没有关系呢？跟着源码，一探究竟。

* 问题演示：
	```java
	@Configuration
	public class Configuration1 {
	
	    @Bean
	    @ConditionalOnBean(Bean2.class)
	    public Bean1 bean1() {
	        return new Bean1();
	    }
	}
	
	@Configuration
	public class Configuration2 {
	
	    @Bean
	    public Bean2 bean2(){
	        return new Bean2();
	    }
	}
	```

结果：
* @ConditionalOnBean(Bean2.class) 返回false。
* 命名定义了bean2，bean1却未加载。

# 源码分析
首先要明确一点，条件注解的解析一定发生在spring ioc的bean definition阶段，因为 spring bean初始化的前提条件就是有对应的bean definition，条件注解正是通过判断bean definition来控制bean能否实例化。


## ConfigurationClassPostProcessor
从 bean definition解析的入口开始：ConfigurationClassPostProcessor
```java
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        int registryId = System.identityHashCode(registry);
        if (this.registriesPostProcessed.contains(registryId)) {
            throw new IllegalStateException(
                    "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
        }
        if (this.factoriesPostProcessed.contains(registryId)) {
            throw new IllegalStateException(
                    "postProcessBeanFactory already called on this post-processor against " + registry);
        }
        this.registriesPostProcessed.add(registryId);
    
        // 解析bean definition入口
        processConfigBeanDefinitions(registry);
    }
```

跟进processConfigBeanDefinitions方法：
```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {

            //省略不必要的代码...

            // 在这里 AutoConfigurationImportSelector 已经回调过了
            // 所有 @Conpoment 扫描到的类 和 @Import 导入的类，都已经注册了
            parser.parse(candidates);
            parser.validate();
    
            // 将所有扫到的配置类存入集合
            Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
            configClasses.removeAll(alreadyParsed);
    
            // Read the model and create bean definitions based on its content
            if (this.reader == null) {
                this.reader = new ConfigurationClassBeanDefinitionReader(
                        registry, this.sourceExtractor, this.resourceLoader, this.environment,
                        this.importBeanNameGenerator, parser.getImportRegistry());
            }
            // 开始解析配置类，也就是条件注解解析的入口
            this.reader.loadBeanDefinitions(configClasses);
            alreadyParsed.addAll(configClasses);
            //...

}
```
## ConfigurationClassBeanDefinitionReader
### loadBeanDefinitions
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
		// 如果要 skip，那么当前配置类即使在 ConpomentScanParser 中被保存到了 BeanDefitionRegistry 也要移除掉
		// skip 的条件是有 @Conditional 注解，且不满足 Condition 的条件
        if (trackedConditionEvaluator.shouldSkip(configClass)) {
            String beanName = configClass.getBeanName();
            if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
     
                this.registry.removeBeanDefinition(beanName);
            }
			
			// 对于 selectImport 返回的类名，并没有进行注册，而是保存在了 importRegistry
			// 不牵扯移除 @Import 导入的类中用 @Bean 注入的类，因为 shouldSkip 不成立才解析 @Bean
            this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
        } else {
            if (configClass.isImported()) {
                this.registerBeanDefinitionForImportedConfigurationClass(configClass);
            }

            Iterator var3 = configClass.getBeanMethods().iterator();

            while(var3.hasNext()) {
                BeanMethod beanMethod = (BeanMethod)var3.next();
                // 解析 @Bean 注解（即遍历 method）
                // 如果 method 上有 @Condtional 还是会调用 shouldSkip 判断）
                this.loadBeanDefinitionsForBeanMethod(beanMethod);
            }

            this.loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
            this.loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
        }
    }
```
loadBeanDefinitionsForBeanMethod
```java
	private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
        ConfigurationClass configClass = beanMethod.getConfigurationClass();
        MethodMetadata metadata = beanMethod.getMetadata();
        String methodName = metadata.getMethodName();
        
        if (this.conditionEvaluator.shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN)) {
            configClass.skippedBeanMethods.add(methodName);
        } else if (!configClass.skippedBeanMethods.contains(methodName)) {
        	....
        	this.registry.registerBeanDefinition(beanName, beanDefToRegister);
        }
    }
```
### shouldSkip
```java
public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, @Nullable ConfigurationPhase phase) {


        //判断是否有条件注解，否则直接返回
        if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
            return false;
        }
    
        if (phase == null) {
            if (metadata instanceof AnnotationMetadata &&
                    ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
                return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
            }
            return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
        }
    
        //获取当前定义bean的方法上，所有的条件注解
        List<Condition> conditions = new ArrayList<>();
        for (String[] conditionClasses : getConditionClasses(metadata)) {
            for (String conditionClass : conditionClasses) {
                Condition condition = getCondition(conditionClass, this.context.getClassLoader());
                conditions.add(condition);
            }
        }
    
        //根据Order来进行排序
        AnnotationAwareOrderComparator.sort(conditions);
    
        //遍历条件注解，开始执行条件注解的流程
        for (Condition condition : conditions) {
            ConfigurationPhase requiredPhase = null;
            if (condition instanceof ConfigurationCondition) {
                requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
            }
            //这里执行条件注解的 condition.matches 方法来进行匹配，返回布尔值
            if ((requiredPhase == null || requiredPhase == phase) && !condition.matches(this.context, metadata)) {
                return true;
            }
        }
    
        return false;
    }
```
## Condition
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210301011207142.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)

### SpringBootCondition
```java
@Override
	public final boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		String classOrMethodName = getClassOrMethodName(metadata);
		try {
			// 这里进行查询
			ConditionOutcome outcome = getMatchOutcome(context, metadata);
			logOutcome(classOrMethodName, outcome);
			recordEvaluation(context, classOrMethodName, outcome);
			return outcome.isMatch();
		}
		catch (NoClassDefFoundError ex) {
			throw new IllegalStateException("Could not evaluate condition on " + classOrMethodName + " due to "
					+ ex.getMessage() + " not " + "found. Make sure your own configuration does not rely on "
					+ "that class. This can also happen if you are "
					+ "@ComponentScanning a springframework package (e.g. if you "
					+ "put a @ComponentScan in the default package by mistake)", ex);
		}
		catch (RuntimeException ex) {
			throw new IllegalStateException("Error processing condition on " + getName(metadata), ex);
		}
	}
```
### OnBeanCondition
```java
	@Override
	public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
		ConditionMessage matchMessage = ConditionMessage.empty();
		
		// @ConditionalOnBean
		if (metadata.isAnnotated(ConditionalOnBean.class.getName())) {
			BeanSearchSpec spec = new BeanSearchSpec(context, metadata, ConditionalOnBean.class);
			MatchResult matchResult = getMatchingBeans(context, spec);
			if (!matchResult.isAllMatched()) {
				String reason = createOnBeanNoMatchReason(matchResult);
				return ConditionOutcome
						.noMatch(ConditionMessage.forCondition(ConditionalOnBean.class, spec).because(reason));
			}
			matchMessage = matchMessage.andCondition(ConditionalOnBean.class, spec).found("bean", "beans")
					.items(Style.QUOTE, matchResult.getNamesOfAllMatches());
		}
		
		// @ConditionalOnSingleCandidate
		if (metadata.isAnnotated(ConditionalOnSingleCandidate.class.getName())) {
			BeanSearchSpec spec = new SingleCandidateBeanSearchSpec(context, metadata,
					ConditionalOnSingleCandidate.class);
			MatchResult matchResult = getMatchingBeans(context, spec);
			if (!matchResult.isAllMatched()) {
				return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnSingleCandidate.class, spec)
						.didNotFind("any beans").atAll());
			}
			else if (!hasSingleAutowireCandidate(context.getBeanFactory(), matchResult.getNamesOfAllMatches(),
					spec.getStrategy() == SearchStrategy.ALL)) {
				return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnSingleCandidate.class, spec)
						.didNotFind("a primary bean from beans")
						.items(Style.QUOTE, matchResult.getNamesOfAllMatches()));
			}
			matchMessage = matchMessage.andCondition(ConditionalOnSingleCandidate.class, spec)
					.found("a primary bean from beans").items(Style.QUOTE, matchResult.getNamesOfAllMatches());
		}
		
		// @ConditionalOnMissingBean
		if (metadata.isAnnotated(ConditionalOnMissingBean.class.getName())) {
			BeanSearchSpec spec = new BeanSearchSpec(context, metadata, ConditionalOnMissingBean.class);
			MatchResult matchResult = getMatchingBeans(context, spec);
			if (matchResult.isAnyMatched()) {
				String reason = createOnMissingBeanNoMatchReason(matchResult);
				return ConditionOutcome
						.noMatch(ConditionMessage.forCondition(ConditionalOnMissingBean.class, spec).because(reason));
			}
			matchMessage = matchMessage.andCondition(ConditionalOnMissingBean.class, spec).didNotFind("any beans")
					.atAll();
		}
		return ConditionOutcome.match(matchMessage);
	}
```
#### BeanSearchSpec
条件注解依赖的bean被封装成了BeanSearchSpec，从名字可以看出是要寻找的对象，这是一个静态内部类，构造方法如下：
```java
		BeanSearchSpec(ConditionContext context, AnnotatedTypeMetadata metadata,
				Class<?> annotationType) {
			this.annotationType = annotationType;
			//读取 metadata中的设置的value
			MultiValueMap<String, Object> attributes = metadata
					.getAllAnnotationAttributes(annotationType.getName(), true);
			//设置各参数，根据这些参数进行寻找目标类
			collect(attributes, "name", this.names);
			collect(attributes, "value", this.types);
			collect(attributes, "type", this.types);
			collect(attributes, "annotation", this.annotations);
			collect(attributes, "ignored", this.ignoredTypes);
			collect(attributes, "ignoredType", this.ignoredTypes);
			this.strategy = (SearchStrategy) metadata
					.getAnnotationAttributes(annotationType.getName()).get("search");
			BeanTypeDeductionException deductionException = null;
			try {
				if (this.types.isEmpty() && this.names.isEmpty()) {
					addDeducedBeanType(context, metadata, this.types);
				}
			}
			catch (BeanTypeDeductionException ex) {
				deductionException = ex;
			}
			validate(deductionException);
		}
```

#### getMatchingBeans()
MatchResult matchResult = getMatchingBeans(context, spec);
```java
	private MatchResult getMatchingBeans(ConditionContext context, BeanSearchSpec beans) {
		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
		if (beans.getStrategy() == SearchStrategy.ANCESTORS) {
			BeanFactory parent = beanFactory.getParentBeanFactory();
			Assert.isInstanceOf(ConfigurableListableBeanFactory.class, parent,
					"Unable to use SearchStrategy.PARENTS");
			beanFactory = (ConfigurableListableBeanFactory) parent;
		}
		MatchResult matchResult = new MatchResult();
		boolean considerHierarchy = beans.getStrategy() != SearchStrategy.CURRENT;
		List<String> beansIgnoredByType = getNamesOfBeansIgnoredByType(
				beans.getIgnoredTypes(), beanFactory, context, considerHierarchy);
				
		//因为实例代码中设置的是类型，所以这里会遍历类型，根据type获取目标bean是否存在
		for (String type : beans.getTypes()) {
			Collection<String> typeMatches = getBeanNamesForType(beanFactory, type,
					context.getClassLoader(), considerHierarchy);
			typeMatches.removeAll(beansIgnoredByType);
			if (typeMatches.isEmpty()) {
				matchResult.recordUnmatchedType(type);
			}
			else {
				matchResult.recordMatchedType(type, typeMatches);
			}
		}
		
		//根据注解寻找
		for (String annotation : beans.getAnnotations()) {
			List<String> annotationMatches = Arrays
					.asList(getBeanNamesForAnnotation(beanFactory, annotation,
							context.getClassLoader(), considerHierarchy));
			annotationMatches.removeAll(beansIgnoredByType);
			if (annotationMatches.isEmpty()) {
				matchResult.recordUnmatchedAnnotation(annotation);
			}
			else {
				matchResult.recordMatchedAnnotation(annotation, annotationMatches);
			}
		}
		
		//根据设置的name进行寻找
		for (String beanName : beans.getNames()) {
			if (!beansIgnoredByType.contains(beanName)
					&& containsBean(beanFactory, beanName, considerHierarchy)) {
				matchResult.recordMatchedName(beanName);
			}
			else {
				matchResult.recordUnmatchedName(beanName);
			}
		}
		return matchResult;
	}
```

getBeanNamesForType()方法
* 最终会委托给BeanTypeRegistry类的getNamesForType方法来获取对应的指定类型的bean name

```java
	Set<String> getNamesForType(Class<?> type) {
		//同步spring容器中的bean
		updateTypesIfNecessary();
		//返回指定类型的bean
		return this.beanTypes.entrySet().stream()
				.filter((entry) -> entry.getValue() != null
						&& type.isAssignableFrom(entry.getValue()))
				.map(Map.Entry::getKey)
				.collect(Collectors.toCollection(LinkedHashSet::new));
	}
```

## 小结
我们来分析一下上面示例bean1为何没有注册？
* 因为在 ConfigurationClassBeanDefinitionReader 的loadBeanDefinitionsForBeanMethod 时，也会判断是否有 @Conditional，有的话也会调用 shouldSkip，而此时注册部分 @Bean 可能还并没有注册。


解决（有两种方式）
* 项目中条件注解依赖的类，大多会交给spring容器管理，所以如果要在配置中Bean通过@ConditionalOnBean依赖配置中的Bean时，完全可以用@ConditionalOnClass(Bean2.class)来代替。
* 如果一定要区分两个**配置类的先后顺序**，可以将这两个类交与EnableAutoConfiguration管理和触发。也就是定义在META-INF\spring.factories中声明是配置类，然后通过@AutoConfigureBefore、AutoConfigureAfter、AutoConfigureOrder控制先后顺序。因为这三个注解只对自动配置类生效。

