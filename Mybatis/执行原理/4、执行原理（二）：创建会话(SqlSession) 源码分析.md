在上一篇我们已经分析了创建 SqlSessionFactory 的源码，最终得到了一个 DefaultSqlSessionFactory。
```java
String resource = "mybatis-config.xml"; 
InputStream inputStream = Resources. getResourceAsStream(resource); 
// build 方法最终返回了 SqlSessionFactory 接口的默认实现 DefaultSqlSessionFactory
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

SqlSession session = sqlSessionFactory.openSession(); 
Blog blog = session.selectOne("blog.findUserById",1);
```


而跟数据库的每一次连接，都需要创建一个会话，我们用 openSession() 方法来创建。所以，这篇我们就接着来分析 openSession() 创建会话的原理。

### openSession()

```java
// DefaultSqlSessionFactory#openSession
// 由于 SqlSessionFactory 是一个接口，所以 SqlSessionFactoryBuilder 最终返回的是一个默认实现类 
public class DefaultSqlSessionFactory implements SqlSessionFactory {

    private final Configuration configuration;    
    // 在 SqlSessionFactoryBuilder#build 构造 DefaultSqlSessionFactory 时已经传入了 Configuration 
    public DefaultSqlSessionFactory(Configuration configuration) {
        this.configuration = configuration;
    }
    
    // 空参openSession（最常用）
    public SqlSession openSession() {
        return this.openSessionFromDataSource(this.configuration.getDefaultExecutorType(), (TransactionIsolationLevel)null, false);
    }
    
    private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
        Transaction tx = null;

        DefaultSqlSession var8;
        try {
        	// 获取 mybatis-conf 中 environment 标签定义的数据源相关内容
            Environment environment = this.configuration.getEnvironment();
           
            // 获取事务工厂
            TransactionFactory transactionFactory = this.getTransactionFactoryFromEnvironment(environment);
            // 1.创建事务
            tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
            
            // 2.创建 Executor
         	// 注：创建 Executor 需要指定事务类型和执行器类型（可以在mybatis-conf.xml配置，默认为SIMPLE）
            Executor executor = this.configuration.newExecutor(tx, execType);
            
            // SqlSession 也是一个接口
            // 所以，这里返回一个默认实现类 DefaultSqlSession 的实例对象
            // 注意，这里传入的参数是 configuration，excutor，是否自动提交事务
            var8 = new DefaultSqlSession(this.configuration, executor, autoCommit);
        } catch (Exception var12) {
            this.closeTransaction(tx);
            throw ExceptionFactory.wrapException("Error opening session.  Cause: " + var12, var12);
        } finally {
            ErrorContext.instance().reset();
        }

        return var8;
    }

	//......还有很多重载的openSession方法
}
```
这个会话里面，需要包含一个 Executor 用来执行 SQL。Executor 又要指定事务类型和执行器的类型。

**1.创建 Transaction**

我们会先从Configuration里面拿到 Enviroment，Enviroment 里面就有事务工厂。
```xml
<environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/><!-- 单独使用时配置成MANAGED没有事务 -->
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
</environments>
```

|  属性   |         产生工厂         |      产生事务      |
| :-----: | :----------------------: | :----------------: |
|  JDBC   |  JdbcTransactionFactory  |  JdbcTransaction   |
| MANAGED | ManagedTransactionFactor | ManagedTransaction |

* 如果配置的是 JDBC，则会使用 Connection 对象的 commit()、rollback()、close() 管理事务。
* 如果配置成 MANAGED，会把事务交给容器来管理，比如 JBOSS，Weblogic。因为我们跑的是本地程序，如果配置成MANAGE不会有任何事务。
>PS：如果是 Spring + MyBatis ， 则没有必要配置 transactionManager， 因为我们会直接在 applicationContext.xml 里面配置数据源和事务管理器，覆盖MyBatis的配置。

**2.创建 Executor**

我们知道， Executor 的基本类型有三种： SIMPLE、BATCH、REUSE，默认是 SIMPLE（settingsElement()读取默认值），他们都继承了抽象类 BaseExecutor（模板方法模式）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201216215041608.png?)


* SimpleExecutor：每执行一次update或select，就开启一个Statement对象，用完立刻关闭Statement对象。
* ReuseExecutor：执行 update 或 select，以 sql 作为 key查找 Statement 对象，存在就使用，不存在就创建，用完后，不关闭 Statement 对象，而是放置于 Map 内，供下一次使用。简言之，就是重复使用Statement对象。
* BatchExecutor：执行 update（没有select，JDBC 批处理不支持 select），将所有 sql 都添加到批处理中（addBatch()），等待统一执行（executeBatch()），它缓存了多个Statement 对象，每个Statement 对象都是 addBatch()完毕后，等待逐一执行executeBatch()批处理。与JDBC批处理相同。

### =>总结

创建会话的过程，我们获得了一个 DefaultSqlSession，里面包含了一个Executor，它是SQL的执行者

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210224004659826.png?)


