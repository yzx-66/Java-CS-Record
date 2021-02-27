在 [【MyBatis】基本使用（一）：编程式使用（单用）及核心对象生命周期](https://yzx66.blog.csdn.net/article/details/114156629) 一文中我们看到了如何单独使用 mybatis，但在实际开发中我们却很少单独使用它，而是整合到 Spring 中去使用。

>经过前面几篇文章的分析，我们已经了解了 mybatis 的底层执行原理，认识了 mybatis 原生API 中三个核心对象：SqlSessionFactory、SqlSession、MapperProxy。
>* [【MyBatis】执行原理（一）：创建会话工厂(SqlSessionFactory) ](https://yzx66.blog.csdn.net/article/details/114156643)
>* [【MyBatis】执行原理（二）：创建会话(SqlSession) 源码分析](https://yzx66.blog.csdn.net/article/details/114156649)
>* [【MyBatis】执行原理（三）：获取代理对象(MapperProxy) 源码分析](https://yzx66.blog.csdn.net/article/details/114156654)
>* [【MyBatis】执行原理（四）：MapperProxy执行SQL源码分析](https://yzx66.blog.csdn.net/article/details/114156660)
>

如果现在让你自己去做整合你会怎么做？或许你会说，不就是把 SqlSessionFactory 和 SqlSession 交给 IOC 容器，在我使用的时候能拿到 sqlsession 不就行了
```xml
<!-- SqlSessionFactory 的默认实现 DefaultSqlSessionFactory-->
<bean id="sqlSessionFactory" class="org.apache.ibatis.session.defaults.DefaultSqlSessionFactory">
	<property name="configuration" value="classpath:mybatis-config.xml"></property>
</bean>
<!-- SqlSession 的默认实现 DefaultSqlSession-->
<bean id="sqlSession" class="org.apache.ibatis.session.defaults.DefaultSqlSession">
	<property name="factory" ref=“sqlSessionFactory”></property>
	<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"></property>
</bean>
```
但这么做仍然存在问题：

1. IOC 容器默认是单例模式，SqlSession 的 DefaultSqlSession 是线程安全的吗？
2. 直接拿到 SqlSession 去操作数据库的话是需要传入 StatementID，是硬编码，比如 sqlsession.selectOne(""blog.findUserById",1)，如何采到基于mapper接口的动态代理模式？
3. 如果我要采用基于 mapper 接口的动态代理模式，那这接口如何注册到 IOC 容器？


下面就我们来看看人家 mybatis 到底是怎么去跟 spring 整合的...


1）引入依赖。除了 MyBatis的依赖之外，我们还需要在 pom 文件中引入 MyBatis 和 Spring 整合的jar包（注意版本！mybatis 的版本和 mybatis-spring 的版本有兼容关系）。
```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.4.2</version>
</dependency> 
<!-- mybatis-spring -->
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-spring</artifactId>
    <version>1.3.1</version>
</dependency>
```
>Spring 对 MyBatis 的对象进行了管理，但是并不会替换 MyBatis 的核心对象。也就意味着：MyBatis jar 包中的 SqlSessionFactory、SqlSession、MapperProxy 这些都会用到，而 mybatis-spring.jar 里面的类只是做了一些包装或者桥梁的工作。
>
2）配置 SqlSessionFactory。在 Spring 的 applicationContext.xml 里面配置 SqlSessionFactoryBean，它是用来帮助我们创建会话的，其中还要指定全局配置文件，数据源，mapper映射器文件的路径
```xml
<!-- 与mybatis整合 -->
<!--  sqlSessionFactory  -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
   <!-- mybatis全局配置文件 -->
   <property name="configLocation" value="classpath:mybatis-config.xml"></property>
   <!-- 数据源 -->
   <property name="dataSource" ref="pooledDataSource"></property>
   <!-- mapper文件 -->
   <property name="mapperLocations" value="classpath:mapper/*.xml"></property>
</bean>
```
3）配置 mapper 接口扫描。我们还需要在 applicationContext.xml 里面指定 mapper 接口所在包
```xml
<!-- 配置扫描器，将mybatis接口的实现加入Ioc(代理)-->
<!-- 注：上面的component-scan无法处理@Mapper，因为@Mapper是mybatis的注解，而且标识的是接口并不是一个class -->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <!-- 扫描所有dao接口 -->
    <property name="basePackage" value="com.xupt.yzh.dao"></property>
</bean>
```
注意，这还有两种配置扫描 mapper 接口的方式：
1. `<mybatis-spring:scan base-package="com.xupt.yzh.dao"/> `
2. @MapperScan("com.xupt.yzh.dao")

4）在使用时我们直接 @Autowired 注入一个Mapper接口，调用它的方法。
```java
@Service 
public class EmployeeService { 
    
    @Autowired 
    // 直接注入mapper接口
    // 使用时直接调用其中方法就好了
    // mybatis-spring.jar 到底做了什么能让我们这么搞？
    EmployeeMapper employeeMapper;
    
	public List<Employee> getAll() { 
        return employeeMapper.selectByMap(null);
    }
}
```
> 想了解 mybatis-spring.jar 到底做了什么及前面几个问题的同学可以参考：
> * [【MyBatis】Spring集成原理（一）：分析注入 SqlSessionFactoryBean](https://yzx66.blog.csdn.net/article/details/114184715)
> * [【MyBatis】Spring集成原理（二）：分析注入 MapperScannerConfigurer](https://yzx66.blog.csdn.net/article/details/114184742)
> * [【MyBatis】Spring集成原理（三）：MapperFactoryBean 与 SqlSessionTemplate](https://yzx66.blog.csdn.net/article/details/114185205)
> * [【MyBatis】Spring集成原理（四）：分析注入 MapperProxy](https://yzx66.blog.csdn.net/article/details/114185867)

