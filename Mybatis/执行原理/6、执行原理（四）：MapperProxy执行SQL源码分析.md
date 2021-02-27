通过前三篇的分析，我们已经了解了 SqlSessionFactory，SqlSession，MapperProxy 底层的逻辑。

```java
String resource = "mybatis-config.xml"; 
InputStream inputStream = Resources. getResourceAsStream(resource); 
// build 最终返回了 SqlSessionFactory 接口的默认实现 DefaultSqlSessionFactory
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

// openSession 最终返回SqlSession接口的一个默认实现DefaultSqlSession 
SqlSession session = sqlSessionFactory.openSession(); 

// 方式一：直接使用 DefaultSqlSession，硬编码，不推荐
Blog blog = session.selectBlogById(1); 

// 方式二：getMapper 最终返回了一个MapperProxy代理对象，推荐
BlogMapper mapper = session.getMapper(BlogMapper.class);
Blog blog = mapper.selectBlogById(1); 
```
我们其实还有最关键的一步没有分析，就是 SqlSession 中的 Executor 到底是如何找到 mapper文件中的 SQL，然后执行它并返回结果的。

既然推荐使用方式二，那我们就以它作为入口。由于所有的 Mapper 都是 MapperProxy 代理对象，所以任意的方法都是执行 MapperProxy 的 invoke() 方法。现在问题来了，这里没有实现类，进入到 invoke 方法的时候做了什么事情？它是怎么找到我们要执行的SQL的？

> 引入MapperProxy 是为了解决硬编码和编译时检查问题。它需要做的事情是：根据方法查找Statement ID。

下面从 MapperProxy#invoke 的源码看起...

### MapperProxy.invoke()

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    // 1.判断当前接口是否有默认方法（default）
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
    } else if (isDefaultMethod(method)) {
      return invokeDefaultMethod(proxy, method, args);
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
  // 2.获取缓存，保存了方法签名和接口方法关系
  final MapperMethod mapperMethod = cachedMapperMethod(method);
  // 3.调用MapperMethod的excute方法
  // 注：这里是通过DefaultSqlSession的Executor去执行方法的（Mapper接口没有实现对象）
  return mapperMethod.execute(sqlSession, args);
}

 private MapperMethod cachedMapperMethod(Method method) {
    // key不存在或value为null，则新加缓存
    return methodCache.computeIfAbsent(method, k -> new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
  }
```
### MapperMethod.excute()

```java
public Object execute(SqlSession sqlSession, Object[] args) {
  Object result;
  switch (command.getType()) {
    // insert
    case INSERT: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
      break;
    }
    // update
    case UPDATE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
      break;
    }
    // delete
    case DELETE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
      break;
    }
    // select
    case SELECT:
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        result = executeForCursor(sqlSession, args);
      } else {
        // 进行参数转换，转换成sql参数！！！
        Object param = method.convertArgsToSqlCommandParam(args);
        // 最终还是调用sqlsession的selectOne方法！！！
        // 通过内部类SqlCommand拿到statementId
        result = sqlSession.selectOne(command.getName(), param);
        if (method.returnsOptional()&& (result == null || !method.getReturnType().equals(result.getClass()))) {
          result = Optional.ofNullable(result);
        }
      }
      break;
    case FLUSH:
      result = sqlSession.flushStatements();
      break;
    default:
      throw new BindingException("Unknown execution method for: " + command.getName());
  }
  if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
    throw new BindingException("Mapper method '" + command.getName()
        + " attempted to return null from a method with a primitive return type (" 
        + method.getReturnType() + ").");
  }
  return result;
}
```
> PS：如果是采用上面的方式一（session.selectOne），就没有以上两个函数调用过程，直接到 DefaultSqlSession#selectOne。
### DefaultSqlSession.selectOne()

```java
public <T> T selectOne(String statement, Object parameter) {
  // selectOne只是selectList的特殊情况
  List<T> list = this.selectList(statement, parameter);
  if (list.size() == 1) {
    return list.get(0);
  } else if (list.size() > 1) {
    throw new TooManyResultsException("Expected one result (or null) to be returned by 
    selectOne(), but found: " + list.size());
  } else {
    return null;
  }
}                                 
```

### DefaultSqlSession.selectList()

```java
public <E> List<E> selectList(String statement, Object parameter) {
  // 传入默认分页（内存分页），size=0，page=0
  return this.selectList(statement, parameter, RowBounds.DEFAULT);
}

@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
    // 根据statementId拿到MappedStatement
    // 这个ms有我们在xml中配置的所有属性，包括id、statementType、sqlSource、useCache、入参、出参等等。
    MappedStatement ms = configuration.getMappedStatement(statement);
    // 调用Excutor的query方法
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

### BaseExcutor.query()

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  BoundSql boundSql = ms.getBoundSql(parameter);
  // 拼接cachekey,这个CacheKey就是缓存的Key
  // 从Configuration 中获取MappedStatement， 然后从BoundSql 中获取SQL信息
  CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
  return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}

public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, 
							CacheKey key, BoundSql boundSql) throws SQLException {
							
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    // 1.查询前处理
    // queryStack用于记录查询栈，防止递归查询重复处理缓存。
    // flushCache=true的时候，会先清理本地缓存（一级缓存）：clearLocalCache();
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      // 2.先查询缓存
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      // 3.没有缓存就从数据库里查
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      deferredLoads.clear();
      // 4.如果LocalCacheScope == STATEMENT，会清理本地缓存
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        clearLocalCache();
      }
    }
    return list;
  }
```

### BaseExcutor.queryFromDatabase()

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, 
ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  List<E> list;
  // 1.缓存：先在缓存用占位符占位。执行查询后，移除占位符，放入数据
  localCache.putObject(key, EXECUTION_PLACEHOLDER);
  try {
    // 2.查询：执行Executor的doQuery()；默认是SimpleExecutor。
    list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
  } finally {
    localCache.removeObject(key);
  }
  localCache.putObject(key, list);
  if (ms.getStatementType() == StatementType.CALLABLE) {
    localOutputParameterCache.putObject(key, parameter);
  }
  return list;
}
```

### SimpleExcutor.doQuery()
```java
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, 
                           ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
  Statement stmt = null;
  try {
    Configuration configuration = ms.getConfiguration();
    // 1.创建StatementHandle 
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, 
                                                                  rowBounds, resultHandler, boundSql);
    // 2.创建具体的statement
    stmt = prepareStatement(handler, ms.getStatementLog());
    // 3.执行sql，最后委托给PreparedStatement
    return handler.query(stmt, resultHandler);
  } finally {
    closeStatement(stmt);
  }
}
```

### SimpleExcutor.newStatementHandler()

在configuration.newStatementHandler()中，new 一个StatementHandler，先得到 RoutingStatementHandler。

RoutingStatementHandler 里面没有任何的实现 ， 是用来创建基本的 StatementHandler 的。这里会根据 MappedStatement 里面的 statementType 决定 StatementHandler 的类型 。

```java
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds 
rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

  switch (ms.getStatementType()) {
    case STATEMENT:
      delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, 
      boundSql);
      break;
    // 默认是 PREPARED （ STATEMENT 、 PREPARED 、CALLABLE）。
    case PREPARED:
      delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, 
      boundSql);
      break;
    case CALLABLE:
      delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, 
      boundSql);
      break;
    default:
      throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
  }

}
```

### SimpleExcutor.prepareStatement()

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Statement stmt;
  // 在第一次执行sql时，创建连接
  Connection connection = getConnection(statementLog);
  stmt = handler.prepare(connection, transaction.getTimeout());
  // 对语句进行预编译，处理参数
  handler.parameterize(stmt);
  return stmt;
}
```

### StatementHandler.query()

RoutingStatementHandler的query()方法。delegate 委派，最终执行PreparedStatementHandler的query()方法。

```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
  PreparedStatement ps = (PreparedStatement) statement;
  // JDBC包PreparedStatement执行sql
  ps.execute();
  // 结果集处理
  return resultSetHandler.handleResultSets(ps);
}
```

### ResultSetHandler.handlerResultSets()

ResultSetHandler 只有一个实现类：DefaultResultSetHandler。

```java
@Override
public List<Object> handleResultSets(Statement stmt) throws SQLException {
  ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

  final List<Object> multipleResults = new ArrayList<>();

  int resultSetCount = 0;
  ResultSetWrapper rsw = getFirstResultSet(stmt);
 
  List<ResultMap> resultMaps = mappedStatement.getResultMaps();
  int resultMapCount = resultMaps.size();
  validateResultMapsCount(rsw, resultMapCount);
  while (rsw != null && resultMapCount > resultSetCount) {
    ResultMap resultMap = resultMaps.get(resultSetCount);
    handleResultSet(rsw, resultMap, multipleResults, null);
    rsw = getNextResultSet(stmt);
    cleanUpAfterHandlingResultSet();
    resultSetCount++;
  }
  // 拿到结果集，如果没有配置一个查询返回多个结果集的情况，一般只有一个结果集
  String[] resultSets = mappedStatement.getResultSets();
  if (resultSets != null) {
    // 如果下面的这个while循环我们也不用，就是执行一次。然后会调用handleResultSet()方法。
    while (rsw != null && resultSetCount < resultSets.length) {
      ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
      if (parentMapping != null) {
        String nestedResultMapId = parentMapping.getNestedResultMapId();
        ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
        handleResultSet(rsw, resultMap, null, parentMapping);
      }
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }
  }

  return collapseSingleResultList(multipleResults);
}
```

### =>总结
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210224004548557.png?)



