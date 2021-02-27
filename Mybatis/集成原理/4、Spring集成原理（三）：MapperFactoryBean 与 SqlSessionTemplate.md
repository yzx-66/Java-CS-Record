在 Spring 里面，我们不是直接使用 DefaultSqlSession 的，而是对它进行了一个封装，这个 SqlSession 的实现类就是**SqlSessionTemplate**。这个跟 Spring 封装其他的组件是一样的，比如 JdbcTemplate，RedisTemplate 等等，也是 Spring 跟 MyBatis 整合的最关键的一个类。

>为什么不用直接使用 DefaultSqlSession？
>DefaultSqlSession 是线程不安全的。因为 SqlSession 的生命周期是请求和操作（Request/Method），所以我们会在每次请求到来（一个请求一般会执行多条sql）的时候都创建一个 DefaultSqlSession（多例，线程安全）

而从 SqlSessionTemplate 的类注释中，我们可以看到它是线程安全的,，Spring IOC 容器中只有一个 SqlSessionTemplate（默认单例）。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020121813435611.png#pic_center)



### SqlSessionDaoSupport
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210227184000586.png)

MyBatis里面提供了一个SqlSessionDaoSupport，里面持有一个SqlSessionTemplate 对象，并且提供了一个 getSqlSession()方法，让我们获得一个SqlSessionTemplate。

```java
public abstract class SqlSessionDaoSupport extends DaoSupport {

  private SqlSession sqlSession;

  private boolean externalSqlSession;
  
  // 因为在 xml 配置文件里 MapperScanConfiger 配置了 SqlSessionFactoryName 属性，
  // 所以 MapperScanConfiger 会在给 mapperClass 注册 beanDefition 时，还会在其 properyValues 里增加一个 setSqlSessionFactory
  // 即会调用到这个 set 方法，然后会创建一个 SqlSessionTemplate 实例
  public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
    if (!this.externalSqlSession) {
      this.sqlSession = new SqlSessionTemplate(sqlSessionFactory);
    }
  }
  
  public void setSqlSessionTemplate(SqlSessionTemplate sqlSessionTemplate) {
    this.sqlSession = sqlSessionTemplate;
    this.externalSqlSession = true;
  }

  // 用户通过此方法去获取 SqlSession
  // 注：实际上返回的是上面 setSqlSessionFactory 创建的 SqlSessionTemplate
  public SqlSession getSqlSession() {
    return this.sqlSession;
  }

  @Override
  protected void checkDaoConfig() {
    notNull(this.sqlSession, "Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required");
  }

}
```


### SqlSessionTemplate

SqlSessionTemplate 实现了 SqlSession 接口，所以跟 DefaultSqlSession 有一样的方法：selectOne()、selectList()、insert()、update()、delete()... 不过所有方法的实现都是通过一个代理对象：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201218135703704.png?)
这个代理对象在 SqlSessionTemplate 构造方法里面通过一个代理类创建：

```java
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {
	
    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    // 基于JDK动态代理创建代理对象
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
  }
```
所以，当调用 SqlSessionTemplate 中的方法时，它们都会走到内部代理类 SqlSessionInterceptor 的 invoke() 方法

```java
// SqlSessionTemplate的内部类
private class SqlSessionInterceptor implements InvocationHandler {

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 事务整合的起点，里面会调用 SqlSessionFactory.openSqlSession，
        // 然后就会用到 SpringMangedTransationFactory 创建 SpringMangedTransation
        // springMangedTransation.getConnection 就会从 TransationSynchrnizManger 中先取
        SqlSession sqlSession = SqlSessionUtils.getSqlSession(SqlSessionTemplate.this.sqlSessionFactory, SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);

        Object unwrapped;
        try {
            Object result = method.invoke(sqlSession, args);
            if (!SqlSessionUtils.isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
                sqlSession.commit(true);
            }

            unwrapped = result;
        } catch (Throwable var11) {
            unwrapped = ExceptionUtil.unwrapThrowable(var11);
            if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
                SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                sqlSession = null;
                Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException)unwrapped);
                if (translated != null) {
                    unwrapped = translated;
                }
            }

            throw (Throwable)unwrapped;
        } finally {
            if (sqlSession != null) {
                SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
            }

        }

        return unwrapped;
    }
```
按照编程式使用的套路，拿到 SqlSession 后其实就可以通过 `SqlSession#selectOne()` 去执行SQL操作数据库了。

### 补充说明
我们让 DAO 层的实现类继承 SqlSessionDaoSupport，并注入 sqlSessionFactory，然后调用 SqlSessionDaoSupport 的 setSqlSessionFactory 也可以获得SqlSessionTemplate，然后就可以操作数据库。


1）在BaseDao里面封装对数据库的操作，包括selectOne()、selectList()、 insert()、delete()这些方法，子类就可以直接调用。

```java
public class BaseDao extends SqlSessionDaoSupport { 

    // 在IOC容器获取 SqlSessionFactory
    // 注：这里实际是通过xml中配置的 SqlSessionFactoryBean 创建的 SqlSessionFactory 实例
    @Autowired 
    private SqlSessionFactory sqlSessionFactory;
	@Autowired
    public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) { 
        super.setSqlSessionFactory(sqlSessionFactory);
    }
	public Object selectOne(String statement, Object parameter) { 
        // 调用SqlSessionDaoSupport的getSqlSession方法获取到SqlSessionTemplate
        // statementID = namespace.sqlId
        return getSqlSession().selectOne(statementID, parameter); 
    } 
    // .......
}
```

2）让我们的实现类继承BaseDao并且实现我们的DAO层接口，这里就是我们的Mapper接口。实现类需要加上@Repository的注解。在实现类的方法里面，我们可以直接调用父类（BaseDao）封装的selectOne()方法，那么它最终会调用sqlSessionTemplate的selectOne()方法。

```java
@Repository 
public class EmployeeDaoImpl extends BaseDao implements EmployeeMapper {
    @Override 
    public Employee selectByPrimaryKey(Integer empId) {
    	// 最后会执行 sqlSessionTemplate.selectOne("com.xupt.yzh.dao.EmployeeMapper.selectById",empId); 
        Employee emp = (Employee) this.selectOne("com.xupt.yzh.dao.EmployeeMapper.selectById",empId); 
        return emp; 
    } 
    // ......
}
```

3）在需要使用的地方，比如Service层，注入我们的实现类，调用实现类的方法就行了。我们这里直接在单元测试类里面注入：

```java
@Autowired 
EmployeeDaoImpl employeeDao;
@Test 
public void EmployeeDaoSupportTest() { 
    // 最终会调用到DefaultSqlSession的方法。
    System.out.println(employeeDao.selectById(1)); 
} 
```

虽然这样也能完数据库操作，但是仍然存在问题：

 1.  代码多：我们的每一个DAO层的接口（Mapper接口也属于），如果要拿到一个 SqlSessionTemplate 去操作数据库，都要创建实现一个实现类，加上@Repository的注解，继承BaseDao，这个工作量也不小。
2. 硬编码：我们去直接调用 selectOne() 方法，还是出现了 StatementID 的硬编码，并且 MyBatis 内部基于接口的动态代理 MapperProxy 在这里根本没用上。



