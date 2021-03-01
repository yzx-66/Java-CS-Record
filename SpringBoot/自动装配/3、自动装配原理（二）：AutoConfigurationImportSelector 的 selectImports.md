# AutoConfigurationImportSelector
该类实现`ImportSelector`接口，最重要的是实现`selectImports`方法，该方法的起到的作用是，根据配置文件（`spring.factories`），将需要注入到容器的bean注入到容器。

## selectImports

```java
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata,
				annotationMetadata);
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
	}
```

首先我们看下，怎样判断自动装配开关的：

```java
protected boolean isEnabled(AnnotationMetadata metadata) {   
    // 判断当前实例的class   
    if (getClass() == AutoConfigurationImportSelector.class) {      
        // 返回 spring.boot.enableautoconfiguration 的值，如果为null，返回true      
        // spring.boot.enableautoconfiguration 可在配置文件中配置，不配则为null      
        return getEnvironment().getProperty(EnableAutoConfiguration.ENABLED_OVERRIDE_PROPERTY, Boolean.class, true);   
    }   
    return true; 
} 
```

接下来，我们看如何获取需要装配的bean：

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
			AnnotationMetadata annotationMetadata) {   
    // 检查自动装配开关   
    if (!isEnabled(annotationMetadata)) {      
        return EMPTY_ENTRY;   
    }   
    // 获取EnableAutoConfiguration中的参数，exclude()/excludeName()   
    AnnotationAttributes attributes = getAttributes(annotationMetadata);   
    // 获取需要自动装配的所有配置类，读取META-INF/spring.factories   
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);   
    // 去重,List转Set再转List   
    configurations = removeDuplicates(configurations);   
    // 从EnableAutoConfiguration的exclude/excludeName属性中获取排除项   
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);   
    // 检查需要排除的类是否在configurations中，不在报错   
    checkExcludedClasses(configurations, exclusions);   
    // 从configurations去除exclusions   
    configurations.removeAll(exclusions);   
    // 对configurations进行过滤，剔除掉不满足 spring-autoconfigure-metadata.properties 所写条件的配置类   
    configurations = filter(configurations, autoConfigurationMetadata);  
    // 监听器 import 事件回调 
    fireAutoConfigurationImportEvents(configurations, exclusions);   
    // 返回(configurations, exclusions)组   
    return new AutoConfigurationEntry(configurations, exclusions); } 
```

可见`selectImports()`是`AutoConfigurationImportSelector`的**核心方法**

该方法的功能主要是以下三点：

- 获取`META-INF/spring.factories`中`EnableAutoConfiguration`所对应的`Configuration`类列表
- 由`@EnableAutoConfiguration`注解中的`exclude/excludeName`参数筛选一遍
- 再由私有内部类`ConfigurationClassFilter`筛选一遍，即不满足`@Conditional`的配置类


# 源码流程
## AutoConfigurationMetadataLoader
AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader)
* 读取所有 classPath 下的 spring-autoconfigure-metadata.properties


![img](https://img-blog.csdnimg.cn/img_convert/c624f8f19af21c5abc2261d1abe69186.png)
结果如下：
![img](https://img-blog.csdnimg.cn/img_convert/5e464c2afe8c6af630a57308d2f2507c.png)

## getAutoConfigurationEntry()
* SprinBoot框架层帮忙做的自动装配元数据

![img](https://img-blog.csdnimg.cn/img_convert/e882f716aba6dfa775e3a35968f02ace.png)

### getAttributes()
AnnotationAttributes attributes = getAttributes(annotationMetadata)
* 获取@EnableAutoConfiguration标注类的元信息。
```java
	protected AnnotationAttributes getAttributes(AnnotationMetadata metadata) {
		String name = getAnnotationClass().getName();
		AnnotationAttributes attributes = AnnotationAttributes.fromMap(metadata.getAnnotationAttributes(name, true));
		Assert.notNull(attributes, () -> "No auto-configuration attributes found. Is " + metadata.getClassName()
				+ " annotated with " + ClassUtils.getShortName(name) + "?");
		return attributes;
	}
```

### getCandidateConfigurations()
List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes)
* 读取所有 classPath 下的 spring.factories
```java
	protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
		// 加载 spring.factories 文件
		// getSpringFactoriesLoaderFactoryClass() = EnableAutoConfiguration.class
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
				getBeanClassLoader());
		Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
				+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```
SpringFactoriesLoader.loadFactoryNames
```java
	
	public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
	}

	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		try {
			// FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					String factoryClassName = ((String) entry.getKey()).trim();
					for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
						result.add(factoryClassName, factoryName.trim());
					}
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210228225418904.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)


  返回的是 ：key = org.springframework.boot.autoconfigure.EnableAutoConfiguration，对应的values值
  * 这些values即是SpringBoot默认的自动装配类，所以有时候读者阅读源码时，发现某些类莫名其妙的被装载到Spring容器中了，一部分原因可能是这个地方搞的鬼。

### removeDuplicates()
configurations = removeDuplicates(configurations)
* 移除重复定义的配置类（ 利用set集合的不可重复性 ）
```java
	protected final <T> List<T> removeDuplicates(List<T> list) {
		return new ArrayList<>(new LinkedHashSet<>(list));
	}
```

### getExclusions()
Set<String> exclusions = getExclusions(annotationMetadata, attributes)
* 获取排除类名单，排除类可通过 exclude = {A.class.B.class}属性来排除指定的配置类。
```java
	// attributes 就是第一步拿到的 AnnotationMetadata 的属性
	protected Set<String> getExclusions(AnnotationMetadata metadata, AnnotationAttributes attributes) {
		Set<String> excluded = new LinkedHashSet<>();
		excluded.addAll(asList(attributes, "exclude"));
		excluded.addAll(Arrays.asList(attributes.getStringArray("excludeName")));
		excluded.addAll(getExcludeAutoConfigurationsProperty());
		return excluded;
	}
```
### checkExcludedClasses
checkExcludedClasses(configurations, exclusions)
* 检查被 ExcludedClasses 的类是否存在现在的 beanFacotry 中
```java
private void checkExcludedClasses(List<String> configurations, Set<String> exclusions) {
		List<String> invalidExcludes = new ArrayList<>(exclusions.size());
		for (String exclusion : exclusions) {
			if (ClassUtils.isPresent(exclusion, getClass().getClassLoader()) && !configurations.contains(exclusion)) {
				invalidExcludes.add(exclusion);
			}
		}
		if (!invalidExcludes.isEmpty()) {
			handleInvalidExcludes(invalidExcludes);
		}
	}
```

### filter()
configurations = filter(configurations, autoConfigurationMetadata)
*  对configurations进行过滤，剔除掉条件不成立的配置类   

  ![img](https://img-blog.csdnimg.cn/img_convert/6e93ba026b82227c82e2036c1d37d153.png)

①：
* 调用的是 **SpringFactoriesLoader.loadFactories(AutoConfigurationImportFilter.class, this.beanClassLoader)**，也是在spring.factories中获取 AutoConfigurationImportFilter类型的过滤器，此处默认有

  ![img](https://img-blog.csdnimg.cn/img_convert/dbc239a38551c7e5e01bb52987e58693.png)

②：
* 分别执行配置类的match方法，由于 **OnBeanCondition、OnClassCondition、OnWebApplicationCondition** 均继承自 FilteringSpringBootCondition，match方法如下：

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210228230742784.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)
  
* 三个子类 ![img](https://img-blog.csdnimg.cn/img_convert/60b842a5cb6b1319ee55cb3ec8991d72.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210301011212420.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)



  
**实现 getOutcomes**
* 通过上面三个子类的方法实现 `ConditionOutcome[] outcomes = getOutcomes(autoConfigurationClasses, autoConfigurationMetadata)`
* 此处拿OnBeanCondition类来分析：

  ![img](https://img-blog.csdnimg.cn/img_convert/3104d33f930ff2901cf26dda59734efb.png)

自动装配类集合迭代调用 `autoConfigurationMetadata.getSet(autoConfigurationClass, "ConditionalOnBean")` 方法获取 `配置类.ConditionalOnBean` 的元信息
* 即在元数据配置文件中的 values。
![img](https://img-blog.csdnimg.cn/img_convert/f29f5227725e3c7b65302ec058570e59.png)
* 以 RedisCacheConfiguration为例，其 "conditionOnBean" 如下：
![img](https://img-blog.csdnimg.cn/img_convert/608b24d0ff6257b62227e70a00faee91.png)

 获取返回的values值后，再调用 getOutcome()方法计算匹配结果
 * ```java
 	protected final ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
			AutoConfigurationMetadata autoConfigurationMetadata) {
		ConditionOutcome[] outcomes = new ConditionOutcome[autoConfigurationClasses.length];
		for (int i = 0; i < outcomes.length; i++) {
			String autoConfigurationClass = autoConfigurationClasses[i];
			if (autoConfigurationClass != null) {
				// 获得配置中 autoConfigurationClasses[i] 的 ConditionalOnBean
				Set<String> onBeanTypes = autoConfigurationMetadata.getSet(autoConfigurationClass, "ConditionalOnBean");
				// 获得 ConditionalOnBean 没有匹配上的 Bean
				outcomes[i] = getOutcome(onBeanTypes, ConditionalOnBean.class);
				// 如果 ConditionalOnBean  都匹配上了
				if (outcomes[i] == null) {
					Set<String> onSingleCandidateTypes = autoConfigurationMetadata.getSet(autoConfigurationClass,
							"ConditionalOnSingleCandidate");
					outcomes[i] = getOutcome(onSingleCandidateTypes, ConditionalOnSingleCandidate.class);
				}
			}
		}
		return outcomes;
	}

	private ConditionOutcome getOutcome(Set<String> requiredBeanTypes, Class<? extends Annotation> annotation) {
		List<String> missing = filter(requiredBeanTypes, ClassNameFilter.MISSING, getBeanClassLoader());
		if (!missing.isEmpty()) {
			ConditionMessage message = ConditionMessage.forCondition(annotation)
					.didNotFind("required type", "required types").items(Style.QUOTE, missing);
			return ConditionOutcome.noMatch(message);
		}
		return null;
	}
 	 ```
 * 最终判断是由 `ClassNameFilter.MISSING#matches` 决定的。
	<img src="https://img-blog.csdnimg.cn/img_convert/366613887a031dd2851da1f87cd49571.png"  width="75%"/>

 ### fireAutoConfigurationImportEvents()
 fireAutoConfigurationImportEvents(configurations, exclusions)
* 监听器 import 事件回调 
 ```java
 private void fireAutoConfigurationImportEvents(List<String> configurations, Set<String> exclusions) {
		List<AutoConfigurationImportListener> listeners = getAutoConfigurationImportListeners();
		if (!listeners.isEmpty()) {
			AutoConfigurationImportEvent event = new AutoConfigurationImportEvent(this, configurations, exclusions);
			for (AutoConfigurationImportListener listener : listeners) {
				invokeAwareMethods(listener);
				listener.onAutoConfigurationImportEvent(event);
			}
		}
}
```
