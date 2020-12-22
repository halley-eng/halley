# MyBatis 源码
### Mapper代理创建


org.mybatis.spring.mapper.MapperFactoryBean#getObject

```JAVA
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
```

这里的配置通常是 SqlSessionTemplate 
它是 spring-mybatis 实现的线程安全会话创建和生命周期管理类;


后期 SqlSession 的方法都会通过他的拦截器 SqlSessionTemplate.SqlSessionInterceptor
详细如下:

###  Sql Session 拦截器接收sql参数并执行

SqlSessionTemplate.SqlSessionInterceptor

该拦截器负责
1. 封装 SqlSession 的获取
2. 调用 SqlSession 的相关方法，完成sql执行的生命周期; 

```JAVA
  private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      // 1. 获取会话 
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,   // 每个工厂么每个线程都能仅能找到唯一一个 SqlSession
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        // 2. 会话中执行 sql, 并得到结果 result  因为使用当前拦截器的类和目标类sqlSession(DefaultSqlSession) 
        //    都有实现 SqlSession 所有这里可以转发过去
        Object result = method.invoke(sqlSession, args);
        // 3. 不是事务的会话 则自动提交
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        // 4. 异常情况 有异常处理并且是PersistenceException异常，则释放连接避免死锁;
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        //  5. 正常结束, 关闭会话
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
```

综上, org.mybatis.spring.SqlSessionTemplate 将sql调用 转发 给 org.mybatis.spring.SqlSessionTemplate#sqlSessionProxy 
通过后者实现 获取当前线程上下文的SqlSession和session声明周期的管理



如上涉及四个子问题

1. 如何获取会话
2. 如何执行sql
3. 如何参数映射
3. 如何结果映射


### 会话获取

SqlSessionUtils#getSqlSession(SqlSessionFactory, ExecutorType, PersistenceExceptionTranslator)
该方法可以通过 TransactionSynchronizationManager 保证每个会话工厂创建唯一一个 会话; 

```JAVA
  public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);
  
    // 1. 查询会话  每个工厂实例都会定位一个一个会话
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
    
    SqlSession session = sessionHolder(executorType, holder);
    // 2. 能查到会话就可以直接返回了
    if (session != null) {
      return session;
    }

    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug("Creating a new SqlSession");
    }
    // 3. 创建会话
    session = sessionFactory.openSession(executorType);
    // 4. 注册会话
    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
  }
```

会话创建过程如下:
DefaultSqlSessionFactory#openSessionFromDataSource

```JAVA
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      // 1. 配置信息
      final Environment environment = configuration.getEnvironment();
      // 2. 根据配置信息获取事务工厂
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      // 3. 事务工厂根据数据源 封装事务对象
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      // 4. 配置器根据事务和执行器类型创建执行器
      final Executor executor = configuration.newExecutor(tx, execType);
      // 5. 将执行器和配置信息封装为 会话
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

如上事务对象 JdbcTransaction 其实只是封装了 connection 对象的连接获取, 事务管理功能, 是后者的子集;

![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086461900202.jpg)



### 会话中执行sql - Mybatis 查询过程


如下为业务代码的访问实现;

```java
		Instant start = Instant.parse("2018-12-10T11:08:00.000Z");
		Instant end = Instant.parse("2018-12-11T11:09:00.006Z");

		List<TbeSyncRecord> records = mapper.findNextReportBatch(start, end);
		ValidateUtils.validIsTrue(records.size() > 0, "not empty");
```


如上 mapper 就是一个 mybatis 生成的一个代理对象;

该代理对象任何方法的调用都会委托给如下回调类

```JAVA
/**
 * @author Clinton Begin
 * @author Eduardo Macarron
 */
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;
  // 支持的mapper接口
  private final Class<T> mapperInterface;
  // 
  private final Map<Method, MapperMethod> methodCache;

  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // 1. 封装声明方法调用
    if (Object.class.equals(method.getDeclaringClass())) {
      try {

        return method.invoke(this, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
    // 2. 封装原始方法method 为 MapperMethod, 由其负责某个statement的调用, 其可以做一些缓存的操作, 比如缓存statement
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }

  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }

}

```


如上 MapperMethod 会负责实际SQL的执行, 使得方法调用转化为sql执行
MapperMethod 负责
1. 封装java方法和接口为 SqlCommand 提供, 使得该方法能够关联到 statement, 并得到 statementname 和type信息
2. 封装java方法为 MethodSignature 提供参数映射，方法元数据查询;

![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086462393356.jpg)


在本demo中就会执行 MapperMethod#execute 中的多行查询方法
根据命令类型和返回类型路由到sqlSession中的指定方法;

```JAVA
  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    if (SqlCommandType.INSERT == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
    } else if (SqlCommandType.UPDATE == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
    } else if (SqlCommandType.DELETE == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
    } else if (SqlCommandType.SELECT == command.getType()) {
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
      // 执行多行查询请求;
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
      }
    } else if (SqlCommandType.FLUSH == command.getType()) {
        result = sqlSession.flushStatements();
    } else {
      throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

多行查询细节如下
1. 拿到参数映射
2. 调用 sqlSession 

```JAVA
  private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    // 1. 参数映射 MethodSignature#convertArgsToSqlCommandParam
    //    方法调用参数, 转化为sqlCommond调用参数
    Object param = method.convertArgsToSqlCommandParam(args);
    // 2. 交给sqlSession查询
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      result = sqlSession.<E>selectList(command.getName(), param, rowBounds);
    } else {

      result = sqlSession.<E>selectList(command.getName(), param);
    }
    // 3. 兼容方法返回类型和SqlCommand返回类型不一致的情况
    // issue #510 Collections & arrays support
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
      if (method.getReturnType().isArray()) {
        return convertToArray(result);
      } else {
        return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
      }
    }
    return result;
  }
```

这里的参数映射结果如下:

![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086462582689.jpg)


解析来会通过 sqlSessionProxy 的代理方法获取会话 SqlSession, 并将上面映射好的参数传递给它;


经过JDBC转化到sql协议
com.mysql.jdbc.JDBC42PreparedStatement#setObject(int, java.lang.Object)



### resultmapping


解析结果集


1. 封装结果处理到 DefaultResultHandler
2. 支持 parentMapping
3. 关闭结果集

```JAVA
  private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
      if (parentMapping != null) {
        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      } else {
        if (resultHandler == null) {
          // 1. 定义结果处理对象
          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
          // 2. 解析每一行并存储到 defaultResultHandler
          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
          // 3. 将处理结果存储到 multipleResults
          multipleResults.add(defaultResultHandler.getResultList());
        } else {
          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
      // issue #228 (close resultsets)
      closeResultSet(rsw.getResultSet());
    }
  }

```

1. 封装级联结果集和单个结果集

```JAVA
  private void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    if (resultMap.hasNestedResultMaps()) {
      ensureNoRowBounds();
      checkResultHandler();
      handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {
      handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
  }

```

封装单个结果集的处理


1. resultContext
1. 迭代结果集
    1. 使用discriminator鉴别器映射
    2. 解析行数据
    3. 输出行数据到resultHandler

```java
  private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
    DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
    skipRows(rsw.getResultSet(), rowBounds);
    while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
      // 1. 使用discriminator鉴别器映射
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
      // 2. 解析行数据
      Object rowValue = getRowValue(rsw, discriminatedResultMap);
      // 3. 输出行数据到resultHandler
      storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
    }
  }

```



解析行数据: 

```JAVA
  private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap) throws SQLException {
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    // 1. 创建结果实例
    Object resultObject = createResultObject(rsw, resultMap, lazyLoader, null);
    // 2. typeHandlerRegistry 如果不能处理该类型，则本地处理;
    if (resultObject != null && !typeHandlerRegistry.hasTypeHandler(resultMap.getType())) {
      // 2.1 封装实例
      final MetaObject metaObject = configuration.newMetaObject(resultObject);
      boolean foundValues = !resultMap.getConstructorResultMappings().isEmpty();
      // 2.2 
      if (shouldApplyAutomaticMappings(resultMap, false)) {
        foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, null) || foundValues;
      }
      // 2.3 将结果集输出到对象;
      foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, null) || foundValues;
      foundValues = lazyLoader.size() > 0 || foundValues;
      resultObject = foundValues ? resultObject : null;
      return resultObject;
    }
    return resultObject;
  }

```




DefaultResultSetHandler#applyPropertyMappings

```java

  private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    // 1. 根据 (resultMap.id + columnPrefix) 去rsw获取对应的列名;
    final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
    boolean foundValues = false;
    // 2. 迭代属性映射列表
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    for (ResultMapping propertyMapping : propertyMappings) {
      // 2.1 根据属性获取列名
      String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      if (propertyMapping.getNestedResultMapId() != null) {
        // the user added a column attribute to a nested result map, ignore it
        column = null;
      }
      // 2.2 结果集中存在该列名
      if (propertyMapping.isCompositeResult()
          || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH)))
          || propertyMapping.getResultSet() != null) {
        // 2.2.1 获取值  
        Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
        // 2.2.2 设置属性
        // issue #541 make property optional
        final String property = propertyMapping.getProperty();
        // issue #377, call setter on nulls
        if (value != DEFERED
            && property != null
            && (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive()))) {
          // metaObject 为 封装后的结果对象;
          metaObject.setValue(property, value);
        }
        if (value != null || value == DEFERED) {
          foundValues = true;
        }
      }
    }
    return foundValues;
  }

```


属性的设置流程如下:


MetaObject#setValue

```JAVA
  public void setValue(String name, Object value) {
    // 1. 对属性按.进行分割
    PropertyTokenizer prop = new PropertyTokenizer(name);
    // 2. 级联迭代属性;
    if (prop.hasNext()) {
      // 2.1 获取子属性
      MetaObject metaValue = metaObjectForProperty(prop.getIndexedName());
      // 2.2 遇到null值
      if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
        // 2.2.1 取消赋值
        if (value == null && prop.getChildren() != null) {
          // don't instantiate child path if value is null
          return;
        // 2.2.2 创建新对象并继续赋值;
        } else {
          metaValue = objectWrapper.instantiatePropertyValue(name, prop, objectFactory);
        }
      }
      // 2.3 设置子属性, 并迭代prop
      metaValue.setValue(prop.getChildren(), value);
      
    // 3. 仅仅设置当前属性的值即可;  
    } else {
      objectWrapper.set(prop, value);
    }
  }
```

PropertyTokenizer 起到属性迭代器的作用 
MetaObject 可以封装原始对象， 对其类型元数据的赋值类;

![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086462885642.jpg)


当前看 <3> 

赋值操作委托给了 BeanWrapper 类实现

![](https://halley-image.oss-cn-beijing.aliyuncs.com/2020/12/22/16086462991639.jpg)

org.apache.ibatis.reflection.wrapper.BeanWrapper#set

```JAVA
  @Override
  public void set(PropertyTokenizer prop, Object value) {
    // 1. 设置到集合
    if (prop.getIndex() != null) {
      Object collection = resolveCollection(prop, object);
      setCollectionValue(prop, collection, value);
    // 2. 注解修改属性值   
    } else {
      setBeanProperty(prop, object, value);
    }
  }

```
实际是从 metaClass 查询到 Invoker 并对其进行调用;

```JAVA

  private void setBeanProperty(PropertyTokenizer prop, Object object, Object value) {
    try {
      // 1. 查询调用方法;
      Invoker method = metaClass.getSetInvoker(prop.getName());
      Object[] params = {value};
      try {
      // 2. 实际执行调用
        method.invoke(object, params);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    } catch (Throwable t) {
      throw new ReflectionException("Could not set property '" + prop.getName() + "' of '" + object.getClass() + "' with value '" + value + "' Cause: " + t.toString(), t);
    }
  }
```















