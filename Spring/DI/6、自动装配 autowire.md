Spring IOC 容器提供了两种管理Bean依赖关系的方式：

* 显式管理：通过BeanDefinition的属性值和构造方法实现Bean依赖关系管理。
* autowiring： Spring IOC 容器的依赖自动装配功能，不需要对Bean属性的依赖关系做显式的声明，只需要在配置好 autowiring 属性，IOC 容器会自动使用反射查找属性的类型和名称，然后基于属性的类型或者名称来自动匹配容器中管理的Bean，从而自动地完成依赖注入。

通过对 autowiring 自动装配特性的理解，我们知道容器对Bean的自动装配发生在容器对Bean依赖注入的过程中。

在前面几篇对 Spring IOC 容器的依赖注入过程源码分析中，我们已经知道了容器对Bean实例对象的属性注入的处理发生在 **AbstractAutoWireCapableBeanFactory** 类中的 populateBean()方法中，我们通过程序流程分析autowiring的实现原理：

### 1.AbstractAutoWireCapableBeanFactory 对Bean实例进行属性依赖注入

应用第一次通过getBean()方法(配置了 lazy-init预实例化属性的除外)向IOC 容器索取 Bean时，容器创建 Bean 实例对象，并且对 Bean 实例对象进行属性依赖注入 ，**AbstractAutoWireCapableBeanFactory** 的 populateBean()方法就是实现 Bean 属性依赖注入的功能，其主要源码如下：

```java
	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw)
    {
		if (bw == null) {
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}

		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the state of the bean before properties are set. 
		// This can be used, for example,to support styles of field injection.
		boolean continueWithPropertyPopulation = true;

		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}

		if (!continueWithPropertyPopulation) {
			return;
		}
		// 获取容器在解析Bean定义资源时为BeanDefiniton中设置的属性值
		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

		// 对依赖注入处理，首先处理autowiring自动装配的依赖注入
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME 
				|| mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// Add property values based on autowire by name if applicable.
			// 根据Bean名称进行autowiring自动装配处理
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// Add property values based on autowire by type if applicable.
			// 根据Bean类型进行autowiring自动装配处理
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}

		// 对非autowiring的属性进行依赖注入处理
		...
```
### 2.Spring IOC容器根据Bean名称或者类型进行autowiring自动依赖注入
```java
	// 根据类型对属性进行自动依赖注入
	protected void autowireByType(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

		// 获取用户定义的类型转换器
		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}

		// 存放解析的要注入的属性
		Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
		// 对Bean对象中非简单属性(不是简单继承的对象，如8中原始类型，字符URL等都是简单属性)进行处理
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		for (String propertyName : propertyNames) {
			try {
				// 获取指定属性名称的属性描述器
				PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
				// Don't try autowiring by type for type Object: never makes sense,
				// even if it technically is a unsatisfied, non-simple property.
				// 不对Object类型的属性进行autowiring自动依赖注入
				if (Object.class != pd.getPropertyType()) {
					// 获取属性的setter方法
					MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
					// Do not allow eager init for type matching in case of a prioritized post-processor.
					// 检查指定类型是否可以被转换为目标对象的类型
					boolean eager = !PriorityOrdered.class.isInstance(bw.getWrappedInstance());
					// 创建一个要被注入的依赖描述
					DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
					// 根据容器的Bean定义解析依赖关系，返回所有要被注入的Bean对象
					Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
					if (autowiredArgument != null) {
						// 为属性赋值所引用的对象
						pvs.add(propertyName, autowiredArgument);
					}
					for (String autowiredBeanName : autowiredBeanNames) {
						// 指定名称属性注册依赖Bean名称，进行属性依赖注入
						registerDependentBean(autowiredBeanName, beanName);
						if (logger.isDebugEnabled()) {
							logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" 
								+ propertyName + "' to bean named '" + autowiredBeanName + "'");
						}
					}
					// 释放已自动注入的属性
					autowiredBeanNames.clear();
				}
			}
			catch (BeansException ex) {
				throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
			}
		}
	}
```
通过上面的源码分析，我们可以看出来通过属性名进行自动依赖注入的相对比通过属性类型进行自动依赖注入要稍微简单一些，但是真正实现属性注入的是DefaultSingletonBeanRegistry 类的 registerDependentBean() 方法。

### 3.DefaultSingletonBeanRegistry 的registerDependentBean()方法对属性注入
```java
	// 为指定的Bean注入依赖的Bean
	public void registerDependentBean(String beanName, String dependentBeanName) {
		// A quick check for an existing entry upfront, avoiding synchronization...
		// 处理Bean名称，将别名转换为规范的Bean名称
		String canonicalName = canonicalName(beanName);
		Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
		if (dependentBeans != null && dependentBeans.contains(dependentBeanName)) {
			return;
		}

		// No entry yet -> fully synchronized manipulation of the dependentBeans Set
		// 多线程同步，保证容器内数据的一致性
		// 先从容器中：bean名称-->全部依赖Bean名称集合找查找给定名称Bean的依赖Bean
		synchronized (this.dependentBeanMap) {
			// 获取给定名称Bean的所有依赖Bean名称
			dependentBeans = this.dependentBeanMap.get(canonicalName);
			if (dependentBeans == null) {
				// 为Bean设置依赖Bean信息
				dependentBeans = new LinkedHashSet<>(8);
				this.dependentBeanMap.put(canonicalName, dependentBeans);
			}
			// 向容器中：bean名称-->全部依赖Bean名称集合添加Bean的依赖信息
			// 即，将Bean所依赖的Bean添加到容器的集合中
			dependentBeans.add(dependentBeanName);
		}
		// 从容器中：bean名称-->指定名称Bean的依赖Bean集合找查找给定名称Bean的依赖Bean
		synchronized (this.dependenciesForBeanMap) {
			Set<String> dependenciesForBean = this.dependenciesForBeanMap.get(dependentBeanName);
			if (dependenciesForBean == null) {
				dependenciesForBean = new LinkedHashSet<>(8);
				this.dependenciesForBeanMap.put(dependentBeanName, dependenciesForBean);
			}
			// 向容器中：bean名称-->指定Bean的依赖Bean名称集合添加Bean的依赖信息
			// 即，将Bean所依赖的Bean添加到容器的集合中
			dependenciesForBean.add(canonicalName);
		}
	}
```
通过对autowiring的源码分析，我们可以看出，autowiring的实现过程：
1. 对Bean的属性代调用getBean()方法，完成依赖Bean的初始化和依赖注入。
2. 将依赖Bean的属性引用设置到被依赖的Bean属性上。
3. 将依赖Bean的名称和被依赖Bean的名称存储在IOC 容器的集合中。

SpringIOC 容器的 autowiring 属性自动依赖注入是一个很方便的特性，可以简化开发时的配置，但是凡是都有两面性，自动属性依赖注入也有不足。

>首先，Bean的依赖关系在 配置文件中无法很清楚地看出来，对于维护造成一定困难。其次，由于自动依赖注入是 Spring容器自动执行的，容器是不会智能判断的，如果配置不当，将会带来无法预料的后果，所以自动依赖注入特性在使用时还是综合考虑。
