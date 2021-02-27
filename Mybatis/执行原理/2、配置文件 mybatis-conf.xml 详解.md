在上一篇 [【Mybatis】编程式使用（单用）及核心对象声明周期](https://yzx66.blog.csdn.net/article/details/114156629)  我们说了使用 mybatis，但是还有一个问题解决，就是 MyBatis 的配置文件到底能配什么？怎么配？本篇我们就一起来看看。

大部分时候我们只需要很少的配置就可以让 MyBatis 运行起来。其实 MyBatis 里面提供的配置项非常多，我们没有配置的时候使用的是系统的默认值。

>[mybatis源码地址](https://github.com/mybatis/mybatis-3/releases)...

下面是一个 mybatis-conf.xml 示例，包括了我们下来要说的的标签
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" 
"http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
	<!-- 数据源设置，dp.propeties配置了数据源信息，可通过${}获取 -->
    <properties resource="db.properties"></properties>
    
    <settings>
        <!-- 打印查询语句 -->
        <setting name="logImpl" value="STDOUT_LOGGING" />
        <!-- 控制全局缓存（二级缓存）-->
        <setting name="cacheEnabled" value="true"/>
        <!-- 延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。默认 false  -->
        <setting name="lazyLoadingEnabled" value="true"/>
        <!-- 开启时，任何方法调用都会加载该对象所有属性(默认 false)，可通过select标签fetchType来覆盖 -->
        <setting name="aggressiveLazyLoading" value="false"/>
        <!--  Mybatis 创建具有延迟加载能力的对象所用到的代理工具，默认JAVASSIST -->
       <setting name="proxyFactory" value="CGLIB" />
        <!-- STATEMENT级别的缓存，使一级缓存，只针对当前执行的这一statement有效 -->
        <!--
                <setting name="localCacheScope" value="STATEMENT"/>
        -->
        <setting name="localCacheScope" value="SESSION"/>
    </settings>
	
    <!-- 类别名，用来mapper.xml中简化参数长度-->
    <typeAliases>
        <typeAlias alias="blog" type="com.my.domain.Blog" />
    </typeAliases>
	
    <!-- 类型处理器，用来对数据库类型与java类型转换 -->
	<typeHandlers>
        <typeHandler handler="com.my.type.MyTypeHandler"></typeHandler>
    </typeHandlers>

    <!-- 对象工厂，用来对查询结果创建java对象 -->
    <!-- 注：调用时都走这个工厂，不过只有指定类型才会特殊处理-->
    <objectFactory type="com.my.objectfactory.MyObjectFactory">
        <property name="gupao" value="666"/>
    </objectFactory>

	<plugins>
        <plugin interceptor="com.my.interceptor.SQLInterceptor">
            <property name="gupao" value="betterme" />
        </plugin>
        <plugin interceptor="com.my.interceptor.MyPageInterceptor">
        </plugin>
    </plugins>

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

    <mappers>
        <mapper resource="BlogMapper.xml"/>
        <mapper resource="BlogMapperExt.xml"/>
    </mappers>
</configuration>
```

### 根标签configuration

configuration 是整个配置文件的根标签，实际上也对应着 MyBatis 里面最重要的配置类 Configuration。它贯穿 MyBatis执行流程的每一个环节。
>注意：MyBatis全局配置文件顺序是固定的，否则启动的时候会报错。

### 1.properties

properties 标签用来配置参数信息，既可以是内部也可以死是外部的，比如最常见的数据库连接信息。

为了避免直接把参数写死在 xml 配置文件中，我们可以把这些参数单独放在 properties 文件中，用 properties 标签引入进来，然后在xml 配置文件中用 ${} 引用就可以了。引入配置文件时可以用 resource 引用应用里面的相对路径，也可以用 url 指定本地服务器或者网络的绝对路径。

```xml
<!-- 数据源设置，dp.propeties配置了数据源信息，可通过${}获取 -->
<properties resource="db.properties"></properties>
```
我们为什么要把这些配置独立出来？有什么好处？或者说，项目在打包的时候，有没有把 properties 文件打包进去？

* 利于多处引用，维护简单；
* 把配置文件放在外部，避免修改后重新编译打包，只需要重启应用；
* 程序和配置分离，提升数据的安全性，比如生产环境的密码只有运维人员掌握。

### 2.environments
 environments 标签用来管理数据库的环境，比如我们可以有开发环境、测试环境、生产环境的数据库。可以在不同的环境中使用不同的数据库地址或者类型。

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

一个environment标签就是一个数据源，代表一个数据库。这里面有两个关键的标签，一个是事务管理器，一个是数据源

* transactionManager：事务配置，这里有两个选项
 	* JDBC，则会使用 Connection 对象的 commit()、rollback()、close() 管理事务。
  	*  MANAGED，会把事务交给容器来管理，比如 JBOSS，Weblogic。因为我们跑的是本地程序，如果配置成 MANAGE不会有任何事务。
* dataSource：在跟Spring集成的时候，事务和数据源都会交给Spring来管理。

>PS：如果是 Spring + MyBatis ， 则没有必要配置transactionManager， 因为我们会直接在 applicationContext.xml 里面配置数据源，覆盖MyBatis的配置。


### 3.settings【重点】

setttings 里面是MyBatis的一些核心配置，下面只列出一些关键配置

| 属性名                  | 含义                | 简介                                         | 有效值                           | 默认值                |
| :------------------------- | :------------------------------------------------------------ | :------------------------------------------------------------ | :---------------------------- | :---------------------------- |
| cacheEnabled              | 是否使用缓存        | 该配置影响的所有映射器中配置的缓存的全局开关               | true \| false                                                | true                                                         |
| lazyLoadingEnabled        | 是否开启延迟加载 | 延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。 特定关联关系中可通过设置`fetchType`属性来覆盖该项的开关状态 | true \| false                                                | false                                                        |
| aggressiveLazyLoading     | 是否需要侵入式延迟加载 | 当启用时，对任意延迟属性的调用会使带有延迟加载属性的对象完整加载；反之，每种属性将会按需加载。 | true \| false                                                | true                                                         |
| multipleResultSetsEnabled | 是否允许单一语句返回多结果集 | 即 Mapper 配置中一个单一的 SQL 配置是否能够返回多个结果集 | true \| false                                                | true                                                         |
| useColumnLabel            | 使用列标签代替列名 | 不同的驱动在这方面会有不同的表现， 具体可参考相关驱动文档或通过测试这两种不同的模式来观察所用驱动的结果。 | true \| false                                                | true                                                         |
| useGeneratedKeys          | 是否支持JDBC自动生成主键 | 允许 JDBC 支持自动生成主键，需要驱动兼容。 如果设置为 true 则这个设置强制使用自动生成主键，尽管一些驱动不能兼容但仍可正常工作（比如 Derby）。 | true \| false                                                | False                                                        |
| autoMappingBehavior       | 指定 MyBatis 应如何自动映射列到字段或属性。 | NONE 表示取消自动映射；PARTIAL 只会自动映射没有定义嵌套结果集映射的结果集。 FULL 会自动映射任意复杂的结果集（无论是否嵌套）。 | NONE, PARTIAL, FULL                                          | PARTIAL                                                      |
| defaultExecutorType       | 配置默认的执行器。 | SIMPLE 就是普通的执行器；REUSE 执行器会重用预处理语句（prepared statements）； BATCH 执行器将重用语句并执行批量更新。 | SIMPLE REUSE BATCH                                           | SIMPLE                                                       |
| defaultStatementTimeout   | 设置超时时间。 | 它决定驱动等待数据库响应的秒数。                             | Any positive integer                                         | Not Set (null)                                               |
| defaultFetchSize          | 设置驱动的结果集获取数量（fetchSize）的提示值。 | 为了防止从数据库查出来的结果过多，而导致内存溢出，可以通过设置fetchSize 参数后来控制结果集的数量 | Any positive integer                                         | Not Set (null)                                               |
| safeRowBoundsEnabled      | 允许在嵌套语句中使用分页（RowBounds） | 如果允许在 SQL 的结果集使用分页，就设置该值为 false | true \| false                                                | False                                                        |
| mapUnderscoreToCamelCase  | 是否开启驼峰命名规则（camel case）映射 | 即从经典数据库列名 A_COLUMN 到经典 Java 属性名 aColumn 的类似映射。 | true \| false                                                | False                                                        |
| localCacheScope           | MyBatis 利用本地缓存机制（Local Cache）防止循环引用（circular references）和加速重复嵌套查询。 | 默认值为 SESSION，这种情况下会缓存一个会话中执行的所有查询。 若设置值为 STATEMENT，本地会话仅用在语句执行上，对相同 SqlSession 的不同调用将不会共享数据。 | SESSION \| STATEMENT                                         | SESSION                                                      |
| jdbcTypeForNull           | JDBC类型的默认设置 | 当没有为参数提供特定的 JDBC 类型时，为空值指定 JDBC 类型。 某些驱动需要指定列的 JDBC 类型，多数情况直接用一般类型即可，比如 NULL、VARCHAR 或 OTHER。 | JdbcType enumeration. Most common are: NULL, VARCHAR and OTHER | OTHER                                                        |
| lazyLoadTriggerMethods    | 指定哪个对象的方法触发一次延迟加载。                         | 配置需要触发延迟加载的方法的名字，该方法就会触发一次延迟加载 | A method name list separated by commas                       | equals,clone,<br>hashCode,toString                               |
| defaultScriptingLanguage  | 动态 SQL 默认语言 | 指定动态 SQL 生成的默认语言。                                | A type alias or fully qualified class name.                  | org.apache.ibatis.<br>scripting.xmltags.<br>XMLDynamicLanguageDriver |
| callSettersOnNulls        | 是否在控制情况下调用 set 方法 | 指定当结果集中值为 null 的时候是否调用映射对象的 setter（map 对象时为 put）方法，这对于有 Map.keySet() 依赖或 null 值初始化的时候是有用的。注意基本类型（int、boolean等）是不能设置成 null 的。 | true \| false                                                | false                                                        |
| logPrefix                 | 日志前缀 | 指定 MyBatis 增加到日志名称的前缀。                          | Any String                                                   | Not set                                                      |
| logImpl                   | 日志实现 | 指定 MyBatis 所用日志的具体实现，未指定时将自动查找。        | SLF4J \| LOG4J \| LOG4J2 \| JDK_LOGGING \| COMMONS_LOGGING \| STDOUT_LOGGING \| NO_LOGGING | Not set                                                      |
| proxyFactory              | 代理工厂 | 指定 Mybatis 创建具有延迟加载能力的对象所用到的代理工具。    | CGLIB \| JAVASSIST                                           | JAVASSIST (MyBatis 3.3 or above)                             |



### 4.typeAliases
TypeAlias是类型的别名，跟Linux系统里面的alias一样，主要用来简化全路径类名的拼写。

比如我们的参数类型和返回值类型都可能会用到我们的Bean，如果每个地方都配置全路径的话，那么内容就比较多，还可能会写错。我们可以单独指定某个Bean的别名，也可以指定一个package，自动转换。配置了别名以后，只需要写别名就可以了，比如com.my.domain.Blog都只要写blog就可以了。

>MyBatis 里面有系统预先定义好的类型别名，在TypeAliasRegistry中。

### 5.typeHandlers【重点】

由于Java类型和数据库的 JDBC  类型不是一一对应的（比如String与varchar），所以我们把Java对象转换为数据库的值，和把数据库的值转换成 Java 对象，需要经过一定的转换，这两个方向的转换就要用到 TypeHandler。

**1.默认转换器**

我们可能会有疑问，我没有做任何的配置，为什么实体类对象里面的一个String属性，可以保存成数据库里面的 varchar 字段，或者保存成 char 字段？

这是因为 MyBatis 已经内置了很多 TypeHandler（在 type 包下），

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201215211206662.png?)
它们全部全部注册在 TypeHandlerRegistry 中，并且继承了抽象类 BaseTypeHandler，泛型就是要处理的Java数据类型。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201215211525317.png?)
当我们做数据类型转换的时候，就会自动调用对应的TypeHandler的方法。

**2.自定义转换器**

如果我们需要自定义一些类型转换规则，或者要在处理类型的时候做一些特殊的动作，就可以编写自己的TypeHandler。

跟系统自定义的TypeHandler一样，继承抽象类BaseTypeHandler<T>。有4个抽象方法必须实现，我们把它分成两类：

  * set方法：Java类型 --> JDBC类型的
  * get方法：JDBC类型 --> Java类型的。 

  比如我们想要在获取或者设置String类型的时候做一些特殊处理，我们可以写一个String类型的TypeHandler。步骤如下：

  第一步，创建 MyTypeHandler继承BaseHandler

  ```java
  public class MyTypeHandler extends BaseTypeHandler<String> {
      @Override
      // Java类型 -->JDBC类型，设置 String 类型的参数的时候调用
      // 注意：只有在字段上添加typeHandler属性才会生效，sql字段上
      public void setNonNullParameter(PreparedStatement ps, int i, String parameter,
                                      JdbcType jdbcType) throws SQLException{

          // insertBlog name字段
          System.out.println("---------------setNonNullParameter1："+parameter);
          ps.setString(i, parameter);
      }
      
  	@Override
      // JDBC类型 --> java类型，根据列名获取 String 类型的参数的时候调用
      // 注意：只有在字段上添加typeHandler属性才会生效，返回结果字段上
      public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
          
          System.out.println("---------------getNullableResult1："+columnName);
          return rs.getString(columnName);
      }
      
  	@Override
      // 根据下标获取 String 类型的参数的时候调用
      public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
          System.out.println("---------------getNullableResult2："+columnIndex);
          return rs.getString(columnIndex);
      }
  	@Override
      public String getNullableResult(CallableStatement cs, int colIdx) throws SQLException {
          System.out.println("---------------getNullableResult3：");
          return cs.getString(colIdx);
      }
  }
  ```

  第二步，注册到mybatis-config.xml中

  ```xml
  <typeHandlers> 
      <typeHandler handler="com.my.type.MyTypeHandler"></typeHandler>
  </typeHandlers>
  ```

  第三步，在使用的字段上指定自定义TypeHandler

  ```xml
  <insert id="insertBlog" parameterType="com.my.domain.Blog">
      insert into blog (bid, name, author_id) values (#{bid,jdbcType=INTEGER}, 
      <!-- 插入值的时候，从Java类型到JDBC类型，在字段属性中指定typehandler -->
      #{name,jdbcType=VARCHAR,typeHandler=com.my.type.MyTypeHandler},
      #{authorId,jdbcType=INTEGER})
  </insert>
  ```

  ​	返回值的时候，从JDBC类型到Java类型，在resultMap的列上指定typehandler

  ```xml
  <result column="name" property="name" jdbcType="VARCHAR" typeHandler="com.my.type.MyTypeHandler"/>
  ```
>**应用场景举例**：如果我们的对象里面有复杂对象，比如Blog里面包括了一个Comment对象，这个时候Comment对象的全部属性不能直接映射到数据库的一个字段。
==> 创建一个TypeHandler，可以将任意的对象转换为json字符串，保存到数据库的VARCHAR类型中。在从数据库查询的时候，再转换为原来的Java对象
### 6.objectFactory【重点】

当我们把数据库返回的结果集转换为实体类的时候，需要创建对象的实例，由于我们不知道需要处理的类型是什么，有哪些属性，所以不能用 new 的方式去创建。

在MyBatis里面，它提供了一个工厂类接口，叫做ObjectFactory，专门用来创建对象实例，里面定义了4个方法。

  | 方法                                                         | 作用                           |
  | :----------------------------------------------------------- | :----------------------------- |
  | void setProperties(Properties properties);                   | 设置参数时调用                 |
  | <T> T create(Class<T> type);                                 | 创建对象（调用无参构造函数）   |
  | <T> T create(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) | 创建对象（调用带参数构造函数） |
  | <T> boolean isCollection(Class<T> type)                      | 判断是否集合                   |

**1.默认对象工厂**

ObjectFactory 有一个默认的实现类 DefaultObjectFactory，创建对象的方法最终都调用了instantiateClass()，是通过反射来实现的。

**2.自定义对象工厂**

如果想要修改对象工厂在初始化实体类的时候的行为，就可以通过创建自己的对象工厂，继承 DefaultObjectFactory 来实现（不需要再实现ObjectFactory 接口）

  ```java
  /**
   * 自定义ObjectFactory，通过反射的方式实例化对象
   * 一种是有参构造函数，一种是无参构造函数第（实际调用了无参构造）
   */
  public class MYObjectFactory extends DefaultObjectFactory {
      @Override
      public Object create(Class type) {
          System.out.println("创建对象方法：" + type);
          Object result = super.create(type);
          
          // 如果是Blog，按如下规则创建
          // 1.调用父类的create方法，按照数据库数据创建对象（已经实现了反射实例化）
          // 2.对创建好的对象的属性进行自定义设置
          if (type.equals(Blog.class)) {
              Blog blog = (Blog) result ;
              blog.setName("object factory");
              blog.setBid(1111);
              blog.setAuthorId(2222);
              return blog;
          }
          
          // 不是指定类型，则依旧按照默认规则创建
          return result;
      }
  }
  ```

  我们可以直接用自定义的工厂类来创建对象

  ```java
  public class ObjectFactoryTest {
      public static void main(String[] args) {
          MYObjectFactory factory = new MyObjectFactory();
          // 通过MYObjectFactory获取对象
          Blog myBlog = (Blog) factory.create(Blog.class);
          System.out.println(myBlog);
      }
  }
  ```

和类型处理器一样，写好我们的对象工厂后，要把它配置到全局配置文件中去，在创建对象的时候会被自动调用。另外，property 标签里我们可以设置一些参数，它会对这个参数 userName 做修改。

  ```xml
  <objectFactory type="org.mybatis.example.MyObjectFactory"> 
      	<!-- 对象工厂注入的参数化 -->
      	<!-- 注：调用时都走这个工厂，不过只有指定类型才会特殊处理-->
          <property name="userName " value="666"/>
  </objectFactory> 
  ```
  几个问题：

  1. 什么时候调用了objectFactory.create()？

     创建DefaultResultSetHandler的时候，和创建对象的时候。

  2. 创建对象后，已有的属性为什么被覆盖了？

     在DefaultResultSetHandler类的395行getRowValue()方法里面里面调用了applyPropertyMappings()。

  3. 返回结果的时候，ObjectFactory和TypeHandler哪个先工作？

     先是ObjectFactory，再是TypeHandler。肯定是先创建对象。

>应用场景举例：比如有一个手机品牌在一个电商平台上面卖货，为了让预约数量好看一点，只要有人预约，预约数量就自动乘以3。这个时候就可以创建一个ObjectFactory，只要是查询销量，就把它的预约数乘以3返回这个实体类。




### 7.plugins

插件是MyBatis的一个很强大的机制，跟很多其他的框架一样，MyBatis预留了插件的接口，让MyBatis更容易扩展。
根据官方的定义，插件可以拦截这四个对象的这些方法，我们把这四个对象称作MyBatis的四大对象。

| 类（或接口）     | 方法                                                         |
| ---------------- | ------------------------------------------------------------ |
| Executor         | update, query, flushStatements, commit, rollback, getTransaction, close, isClosed |
| ParameterHandler | getParameterObject, setParameters                            |
| ResultSetHandler | handleResultSets, handleOutputParameters                     |
| StatementHandler | prepare, parameterize, batch, update, query                  |

### 8.mappers

mappers标签配置的是我们的映射器，也就是Mapper.xml的路径。这里配置的目的是让MyBatis在启动的时候去扫描这些映射器，创建映射关系。

我们有四种指定Mapper文件的方式：

* 使用相对于类路径的资源引用（resource）
* 使用完全限定资源定位符（绝对路径）（url）
* 使用映射器接口实现类的完全限定类名（class）
* 将包内的映射器接口实现全部注册为映射器（最常用）

### 总结

|   配置名称    |  配置含义   | 配置简介                                                     |
| :-----------: | :----------: | :----------------------------------------------------------- |
| configuration | 包裹所有标签 | 配置文件的顶级标签                                           |
|  properties   |     属性     | 该标签可以引入外部配置的属性，也可以自己配置。<br>该配置标签所 在的同一个配置文件的其他配置均可以引用此配置中的属性 |
|  enviroments  |   环境配置   | 数据库环境信息的集合。<br>在一个配置文件中，可以有多种数据库环境集合这样可以使 MyBatis 将 SQL 同时映射至多个数据 |
|   settings    | 全局配置参数 | 用来配置一些改变运行时行为的信息<BR>例如是否使用缓存机制，是否使用延迟加载，是否使用错误处理机制等 |
|  typeAliases  |     别名     | 用来设置一些别名来代替 Java 的长类型声明（如 java.lang.int 变为 int），减少配置编码的冗余 |
| typeHandlers  |  类型处理器  | 将数据库获取的值以合适的方式转换为 Java 类型，或者将 Java 类型的参数转换为数据库对应的类型 |
| objectFactory |   对象工厂   | 实例化目标类的工厂类配置                                     |
|    plugins    |     插件     | 可以通过插件修改 MyBatis 的核心行为，例如对语句执行的某一点 进行拦截调用 |
|    mappers    |    映射器    | 配置 SQL 映射文件的位置，告知 MyBatis 去哪里加载 SQL 映射文件 |
