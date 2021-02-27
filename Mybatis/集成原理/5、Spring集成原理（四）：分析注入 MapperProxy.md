我们已经分析了 MyBatis 是如何实现自定义注册 Mapper 接口到 IOC 容器中的，最后我们看到 BeanDefinition 中实际保存的是 MapperFactoryBean.class。



![在这里插入图片描述](https://img-blog.csdnimg.cn/20201219032105859.png?)

可以看到 MapperFactoryBean 实现了 FactoryBean 接口（表明是一个工厂bean），所以 getBean() 最终返回的并不是它自己，而是 getObject 中创建的对象。



### MapperFactoryBean
```java
// MapperFactoryBean#getObject()
public T getObject() throws Exception { 
	// 因为 MapperFactoryBean 继承了 SqlSessionDaoSupport 
	// 所以这个 getSqlSession() 就是调用父类的方法，返回 SqlSessionTemplate。
	return getSqlSession().getMapper(this.mapperInterface);
}

// SqlSessionTemplate#getMapper()
public <T> T getMapper(Class<T> type) {  
	// 1.获取配置类 Configuration
	// 2.创建代理对象 MapperProxy
	// PS:这里采用的是链式编程
	return getConfiguration().getMapper(type, this);
}
```
我想第一个问题的答案已经很明确了。

**1.获取配置类 Configuration**
```java
// 1.1 SqlSessionTemplate#getConfiguration
public Configuration getConfiguration() {
	// 调用 DefaultSqlSessionFactory#getConfiguration()
    return this.sqlSessionFactory.getConfiguration(); 
}

// 1.2 DefaultSqlSessionFactory#getConfiguration
public Configuration getConfiguration() { 
	// 返回全部配置Configuration
	return configuration; 
}
```
**2.创建代理对象**

回到了 mybatis

```java
// 2.1 Configuration.getMapper()
public <T> T getMapper(Class<T> type, SqlSession sqlSession) { 
	// 调用 MapperRegister#getMapper()
	return mapperRegistry.getMapper(type, sqlSession); 
}

// 2.2 MapperRegister#getMapper()
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
	// 获取MapperProxy工厂
    // 在解析mapper标签和Mapper.xml的时候已经把接口类型和类型对应的MapperProxyFactory放到了一个Map中。
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      // 通过 MapperProxyFactory 获取具体对象
      // 注：这里使用的是JDK动态代理，不过代理对象不用组合接口实现对象
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```
可以看到，跟编程式使用里面的 getMapper 一样，通过工厂类 MapperProxyFactory 获得一个MapperProxy代理对象。也就是说，我们注入到Service层的接口，实际上还是一个MapperProxy代理对象。所以最后调用 Mapper接口方法，就是执行 MapperProxy的invoke()方法，后面流程就跟编程式的工程里面一模一样了。



### 小结

下面对 Spring 集成原理这四篇文章做个小结：

1）几个关键对象

| 对象                            | 生命周期                                                     |
| :------------------------------- | :------------------------------------------------------------ |
| SqlSessionTemplate              | Spring 中 SqlSession 的替代品，是线程安全的，通过代理的方式调用 DefaultSqlSession 的方法 |
| SqlSessionInterceptor（内部类） | 代理对象，用来代理 DefaultSqlSession，在 SqlSessionTemplate 中使用 |
| SqlSessionDaoSupport            | 用于获取 SqlSessionTemplate，只要继承它即可 MapperFactoryBean 注册到 IOC 容器中替换接口类，继承了 SqlSessionDaoSupport 用来获取 SqlSessionTemplate，因为注入接口的时候，就会调用它的 getObject()方法 |
| SqlSessionHolder                | 控制 SqlSession 和事务                                       |

2）用到 Spring 的扩展点
| 接口                                | 方法                                | 作用                                |
| :----------------------------------- | :----------------------------------- | :----------------------------------- |
| FactoryBean                         | getObject()                         | 返回由 FactoryBean 创建的 Bean 实例 |
| InitializingBean                    | afterPropertiesSet()                | bean 属性初始化完成后添加操作       |
| BeanDefinitionRegistryPostProcessor | postProcessBeanDefinitionRegistry() | 注入 BeanDefination 时添加操作      |
