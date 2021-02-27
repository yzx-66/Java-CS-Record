通过前两篇的分析，我们已经了解了 SqlSessionFactory，SqlSession 底层的逻辑。

```java
String resource = "mybatis-config.xml"; 
InputStream inputStream = Resources. getResourceAsStream(resource); 
// build 最终返回了 SqlSessionFactory 接口的默认实现 DefaultSqlSessionFactory
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

// openSession 最终返回SqlSession接口的一个默认实现DefaultSqlSession 
SqlSession session = sqlSessionFactory.openSession(); 
```
现在我们已经有一个 DefaultSqlSession 了，下一步工作就是找到 Mapper.xml 里面定义的 Statement ID，才能执行对应的SQL语句。

找到StatementID有两种方式：

1. 硬编码：直接调用session的方法，在参数里面传入Statement ID

 	 

	```java
	Blog blog = (Blog) session.selectOne("com.my.mapper.BlogMapper.selectBlogById ", 1);
	```
	这种方式属于硬编码，我们没办法知道有多少处调用，修改起来也很麻烦。另一个问题是如果参数传入错误，在编译阶段也是不会报错的，不利于预先发现问题。所以在实际开发中不推荐使用。
2. 约定：在 MyBatis 后期的版本提供了第二种方式，就是定义一个接口，然后再调用Mapper接口的方法。

  	由于我们的接口名称跟 Mapper.xml 的 namespace 是对应的，接口的方法跟statement ID也都是对应的，所以根据方法就能找到对应的要执行的SQL。

	```java
	BlogMapper mapper = session.getMapper(BlogMapper.class);
	Blog blog = mapper.selectBlogById(1); 
	```

我们在实际工作中般都是采用的第二种方式。由于 BlogMapper 只是一个接口，所以很容易想到 SqlSession#getMapper() 会给我们生成一个具体的代理对象，并且实现了 BlogMapper 接口。

下面，我们就来分析 Mapper 对象相关源码...

### getMapper()
```java
// 1.DefaultSqlSession#getMapper()
// 调用了 Configuration的getMapper()方法。
public <T> T getMapper(Class<T> type) {
  return configuration.getMapper(type, this);
}

// 2.Configuration#getMapper()
// 调用了MapperRegistry 的getMapper()方法。
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
 }

// 3.MapperRegister#getMapper()
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
	// 获取MapperProxy工厂
    // 在解析mapper标签和Mapper.xml的时候已经把接口类型和类型对应的MapperProxyFactory放到了一个Map中。
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      // 从相应的 mapperProxyFactory 工厂获取具体对象
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

### newInstance()

```java
// MapperProxyFactory.newInstance()
public T newInstance(SqlSession sqlSession) {
	// MapperProxy 是代理类，它实现了 InvocationHandler 接口
	// 构造代理对象时的入参是：SqlSession，mapper接口，方法缓存
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
  
// 通过JDK动态代理生成代理对象
protected T newInstance(MapperProxy<T> mapperProxy) { 
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] {mapperInterface }, mapperProxy);
  }
```
到这里我们可以看到已经生成了代理对象。

**问题**：我们常常使用的JDK动态代理和MyBatis用到的JDK动态代理有什么区别呢？

我们平常使用JDK动态代理时，需要在实现了InvocationHandler的代理类里面传入（组合）一个被代理对象的实现类，让这个被代理对象去执行接口中的方法。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201217164420365.png?)
```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	method.invoke(ProxyImpl, args);
}
```
而对于 Mybatis 动态代理，它代理的 Mapper 接口并没有实现类，因为所有 Mapper 接口中的方法是都是交给 SqlSession 的 Excutor 去执行，并不需要一个实现类。

因为我们唯一要做的就是根据接口类型+方法的名称找到StatementID。执行 Mapper 接口中方法的是 SqlSession 的执行器 Executor。
<img src="https://img-blog.csdnimg.cn/20201222170817727.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNTkyNw==,size_16,color_FFFFFF,t_70#pic_center" width="70%">
MapperProxy 唯一要做的事情就是根据接口类型（namespace）+ 方法名（sqlId）得到 statementID，然后再调用 SqlSession 的 insert()、delete()、update()、selectOne()、selectList() 等方法去执行。下面 MapperProxy 部分源码：

<img src="https://img-blog.csdnimg.cn/20201219194014700.png?" width="70%"><br>


>关于invoke的逻辑，及如何执行sql的放在下一篇去说...


### =>总结

获得Mapper对象的过程，实质上是获取了一个 MapperProxy 的代理对象。MapperProxy 中有 sqlSession、mapperInterface、methodCache。




![在这里插入图片描述](https://img-blog.csdnimg.cn/20210224004419476.png?)

