![在这里插入图片描述](https://img-blog.csdnimg.cn/20201208152458211.png?)


向IOC容器注册在第二步解析好的 BeanDefinition，这个过程是通过 BeanDefinitionRegistery 接口来实现的。在 IOC 容器内部其实是将第二个过程解析得到的 BeanDefinition 注入到一个 **HashMap** 容器中，IOC 容器就是通过这个 HashMap 来维护这些 BeanDefinition 的。

>在这里需要注意的一点是这个过程并没有完成依赖注入，**依赖注册是发生在应用第一次调用 getBean() 向容器索要 Bean 时**。当然我们可以通过设置预处理，即对某个 Bean 设置 lazyinit 属性，那么这个 Bean 的依赖注入就会在容器初始化的时候完成。

## 1.分配注册策略 (BeanDefinitionReaderUtils)

让我们继续跟踪程序的执行顺序，接下来我们来分析 DefaultBeanDefinitionDocumentReader 对 Bean 定义转换的 Document 对象解析的流程中，在其 parseDefaultElement()方法中完成对 Document 对象的解析后，得到封装 BeanDefinition 的 BeanDefinitionHold 对象，然后调用**BeanDefinitionReaderUtils** 的 registerBeanDefinition()方法向 IOC 容器注册解析的 Bean。BeanDefinitionReaderUtils的注册的源码如下：

### registerBeanDefinition()

将解析的BeanDefinitionHold注册到容器中

```java
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {
 
		// Register bean definition under primary name.
		// 获取解析的BeanDefinition的名称
		String beanName = definitionHolder.getBeanName();
		// 向IOC容器注册BeanDefinition
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
		// 如果解析的BeanDefinition有别名，向容器为其注册别名
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
```

当调用BeanDefinitionReaderUtils向 IOC 容器注册解析的BeanDefinition时，真正完成注册功DefaultListableBeanFactory

## 2.向容器注册 (DefaultListableBeanFactory)

**DefaultListableBeanFactory** 中使用一个 HashMap 的集合对象存放 IOC 容器中注册解析的 BeanDefinition，向IOC 容器注册的主要源码如下：
<img src="https://img-blog.csdnimg.cn/20201208155438309.png?" width="28%">

### registerBeanDefinition()

向IOC容器注册解析的BeanDefiniton

```java
	@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		// 校验解析的BeanDefiniton
		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition oldBeanDefinition;

		oldBeanDefinition = this.beanDefinitionMap.get(beanName);

		if (oldBeanDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName 
						+ "': There is already [" + oldBeanDefinition + "] bound.");
			}
			else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or 
                // ROLE_INFRASTRUCTURE
				if (this.logger.isWarnEnabled()) {
					this.logger.warn("Overriding user-defined bean definition for bean '" + beanName 
					 +	"' with a framework-generated bean definition: replacing [" +oldBeanDefinition 
					 + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(oldBeanDefinition)) {
				if (this.logger.isInfoEnabled()) {
					this.logger.info("Overriding bean definition for bean '" + beanName 
						+ "' with a different definition: replacing [" + oldBeanDefinition +"] with [" 
						+ beanDefinition + "]");
				}
			}
			else {
				if (this.logger.isDebugEnabled()) {
					this.logger.debug("Overriding bean definition for bean '" + beanName 
						+ "' with an equivalent definition: replacing [" + oldBeanDefinition + "] with ["
						+ beanDefinition + "]");
				}
			}
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				// 注册的过程中需要线程同步，以保证数据的一致性
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>
                        	(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
				// Still in startup registration phase
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		// 检查是否有同名的BeanDefinition已经在IOC容器中注册
		if (oldBeanDefinition != null || containsSingleton(beanName)) {
			// 重置所有已经注册过的BeanDefinition的缓存
			resetBeanDefinition(beanName);
		}
	}
```

至此，Bean配置信息中配置的Bean被解析过后，已经注册到IOC 容器中，被容器管理起来，真正完成了 IOC 容器初始化所做的全部工作。现在 IOC 容器中已经建立了整个 Bean 的配置信息，**这些BeanDefinition信息已经可以使用**，并且可以被检索。

>**IOC 容器的作用就是对这些注册的Bean定义信息进行处理和维护**。这些的注册的 Bean定义信息是IOC 容器**控制反转的基础**，正是有了这些注册的数据，容器才可以进行依赖注入。
