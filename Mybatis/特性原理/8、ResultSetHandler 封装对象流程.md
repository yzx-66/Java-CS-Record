# ResultSetHandler

## 前言

### 结果封装原理

```xml
<resultMap type="Person" id="resultPersonMap">
    <result property="id" column="id"/>
    <result property="username" column="username"/>
    <result property="password" column="password"/>
    <result property="fltNum" column="flt_num"/>
</resultMap>

<select id="findByUsername" resultType="Person" parameterType="Person">
    select * from person where username=#{username};
</select>
```

如上定义这样一个ResultMap后，使用ObjectFactory创建一个Person对象，

```java
person.setId(resultSet.getInt("id"))
person.setUsername(resultSet.getString("username"))
person.setPassword(resultSet.getString("password"))
person.setFltNum(resultSet.getString("flt_num"))
```

不过这个转换过程在实现上很复杂，其中就用到TypeHandler。



### ResultSetWrapper

通过ResultSet 获取 ResultSetMetaData 来获取列的属性，遍历列，获取列名称、列类型、对应的JdbcType。

```java
public ResultSetWrapper(ResultSet rs, Configuration configuration) throws SQLException {
    super();
    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.resultSet = rs;
    final ResultSetMetaData metaData = rs.getMetaData();
    final int columnCount = metaData.getColumnCount();
    for (int i = 1; i <= columnCount; i++) {
        columnNames.add(configuration.isUseColumnLabel() ? metaData.getColumnLabel(i) : metaData.getColumnName(i));
        jdbcTypes.add(JdbcType.forCode(metaData.getColumnType(i)));
        classNames.add(metaData.getColumnClassName(i));
    }
}
```

  

## ResultSetHandler 

 ResultSetHandler是个接口

```java
public interface ResultSetHandler {
  //将结果转换为List
  <E> List<E> handleResultSets(Statement stmt) throws SQLException;
  //将结果转换为游标Cursor
  <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;

  void handleOutputParameters(CallableStatement cs) throws SQLException;
}
```

 实现类只有DefaultResultSetHandler，实现有点复杂，因为要考虑的情况很多。

###  DefaultResultSetHandler

handleResultSets 方法：

```java
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());
    final List<Object> multipleResults = new ArrayList<Object>();
    int resultSetCount = 0;
    // 通过 ResultSetWrapper 获取列的属性
    ResultSetWrapper rsw = getFirstResultSet(stmt);//1

    // 获取我们定义的resultMap，即 mapper.xml 里配置的 <resultMap>
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();//2
    int resultMapCount = resultMaps.size();
    // 验证resultMap个数，如果小于1则会报错
    validateResultMapsCount(rsw, resultMapCount);//3
    while (rsw != null && resultMapCount > resultSetCount) {
        // 获取resultMap，从List中
        ResultMap resultMap = resultMaps.get(resultSetCount);//4
        // 核心：调用 handleResultSet -> handleRowValues -> handleRowValuesForSimpleResultMap
        handleResultSet(rsw, resultMap, multipleResults, null);//5
        
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
    }

    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
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



handleRowValuesForSimpleResultMap

```java
private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
    throws SQLException {
    DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
    skipRows(rsw.getResultSet(), rowBounds);
    // 不断调用resultSet.next()方法，会获取resultSet中的所有数据
    while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
        ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
        // getRowValue方法，该方法获取resultSet中的一行数据，并将数据封装位对象
        Object rowValue = getRowValue(rsw, discriminatedResultMap);//1、
        // getRowValue方法返回值，storeObject方法中将值放入到List中。
        storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
    }
}
```



getRowValue：

```java
private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap) throws SQLException {
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    // 通过 objectFacotry 创建对象，其中很重要的一步，懒加载也是在这里生成的代理对象
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, null);//1
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        final MetaObject metaObject = configuration.newMetaObject(rowValue);
        boolean foundValues = this.useConstructorMappings;
        if (shouldApplyAutomaticMappings(resultMap, false)) {
            // 进行属性的填充
        	foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, null) || foundValues;//2
        }
        foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, null) || foundValues;
        foundValues = lazyLoader.size() > 0 || foundValues;
        rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
}
```



applyAutomaticMappings：

```java
private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    // 遍历 resultSetWrapper 里的列名，然后根据当前 coloumName 在 resultMap 找到对应的 propertyName ， 
    // 并且把 colunName、propertyName、typeHandler 封装成 autoMapping 加入 autoMappings 集合
    List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    boolean foundValues = false;
    if (!autoMapping.isEmpty()) {
        // 遍历这些列，对每一列，都调用typeHandler.getResult方法获取值，
        // 之后用metaObject.setValue，内部通过反射的方式设置值。
        for (UnMappedColumnAutoMapping mapping : autoMapping) {
            final Object value = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
            if (value != null) {
                foundValues = true;
            }
            if (value != null || (configuration.isCallSettersOnNulls() && !mapping.primitive)) {
                // gcode issue #377, call setter on nulls (value is not 'found')
                metaObject.setValue(mapping.property, value);
            }
        }
    }
    return foundValues;
}
```

