# Mybatis缓存

### 背景
```
有段业务中需要查询多个事件的详情，事件中有对应的作者信息查询
但是多个事件可能会是同一个作者信息，这时候就想到使用缓存；
但是用redis又需要自己显式地管理缓存，就把目光对准了Mybatis;

以前知道Mybatis有一级二级缓存，现在通过查看源码把Mybatis缓存扒干净；

```

### 一级缓存和二级缓存的范围、区别
```
	
配置：
	一级缓存：springboot 中 配置 mybatis.configuration.cache-enabled=true
						   mybatis.configuration.local-cache-scope=session

	二级缓存：在具体的mapper的sql 中配置 <cache> 和 userCache 和 flushCache 属性

范围：
	一级缓存：statment 级别或者 session(sqlSession) 级别
	二级缓存：应用级别
	
区别：
	一级缓存：需要具体 sql 具体配置 cache 
	二级缓存：无法配置 cache 容量和策略，可能会有oom 风险(个人觉得可能性太小，除非一个业务请求涉及查询太大量)
	
执行：
	一级缓存：BaseExecutor (SimpleExecutor)
	二级缓存：CachingExecutor

```

### SqlSession 
#### SqlSession 是提供给用户(开发者)，调用 Mybatis提供查询的接口
#### Spring 中使用 SqlSessionTemplate 通过代理 SqlSession (SqlSessionInterceptor) 实现生成 SqlSession(DefaultSqlSession) 策略
#### 原因：
```
原则上每次请求都会生成一个 SqlSession 实例，但是在Service中，用户可能会当成一个事务单位(但是对于 mysql，应用里的很多数据库操作可能是同一个连接，就会有应用层的事务管理需要实现)

所以在 SqlSessionInterceptor 中调用 SqlSessionUtils.getSqlSession() 获取 SqlSession
每次获取 SqlSession 时，都会通过Mybatis的 Configuration 创建一个 Executor 对象，
实际的数据库操作都是通过 SqlSession 中对 Executor 的调用；

```

#### SqlSessionInterceptor 代码

```

    /**
   * Proxy needed to route MyBatis method calls to the proper SqlSession got
   * from Spring's Transaction Manager
   * It also unwraps exceptions thrown by {@code Method#invoke(Object, Object...)} to
   * pass a {@code PersistenceException} to the {@code PersistenceExceptionTranslator}.
   */
  private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      /*
 		SqlSessionUtils.getSqlSession() 获取SqlSession 
 		如果是同一个事务，获取到的会是同一个 SqlSession 
 		通过ThreadLocal 就是 Spring 的事务实现
      */
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
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
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }

```


#### SqlSessionUtils.getSqlSession()

```


  /**
   * Gets an SqlSession from Spring Transaction Manager or creates a new one if needed.
   * Tries to get a SqlSession out of current transaction. If there is not any, it creates a new one.
   * Then, it synchronizes the SqlSession with the transaction if Spring TX is active and
   * <code>SpringManagedTransactionFactory</code> is configured as a transaction manager.
   *
   * @param sessionFactory a MyBatis {@code SqlSessionFactory} to create new sessions
   * @param executorType The executor type of the SqlSession to create
   * @param exceptionTranslator Optional. Translates SqlSession.commit() exceptions to Spring exceptions.
   * @throws TransientDataAccessResourceException if a transaction is active and the
   *             {@code SqlSessionFactory} is not using a {@code SpringManagedTransactionFactory}
   * @see SpringManagedTransactionFactory
   */
  public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);
    /*
    	Spring 事务管理器 TransactionSynchronizationManager
    */
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
      return session;
    }

    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug("Creating a new SqlSession");
    }
    /*
    	新建sqlSession 这里的 sessionFactory 是 DefaultSqlSessionFactory 
    */
    session = sessionFactory.openSession(executorType);

    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
  }

```


#### DefaultSqlSessionFactory.openSessionFromConnection() 方法
```

  private SqlSession openSessionFromConnection(ExecutorType execType, Connection connection) {
    try {
      boolean autoCommit;
      try {
        autoCommit = connection.getAutoCommit();
      } catch (SQLException e) {
        // Failover to true, as most poor drivers
        // or databases won't support transactions
        autoCommit = true;
      }      
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      final Transaction tx = transactionFactory.newTransaction(connection);
      /*
      	这里很清楚看到每次的数据库访问请求都会创建一个 Executor 对象
      	并且创建的 SqlSession 对象实例一般都是 DefaultSqlSession
      */
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }


```



### 一级缓存与 BaseExecutor (SimpleExecutor)
#### 当二级缓存 (CachingExecutor) 查询不到结果时，调用一级缓存的 query() 方法, 即delegate.query() 
#### BaseExecutor 源码
```

public abstract class BaseExecutor implements Executor {
	
  private static final Log log = LogFactory.getLog(BaseExecutor.class);

  protected Transaction transaction;
  protected Executor wrapper;

  protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
  //本地缓存的对象
  protected PerpetualCache localCache;
  protected PerpetualCache localOutputParameterCache;
  protected Configuration configuration;

  protected int queryStack;
  private boolean closed;

  ....省略部分代码


  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    //获取缓存 CacheKey
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
 }

  @SuppressWarnings("unchecked")
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      // resultHandler 一般是null ， 所以会先查缓存
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
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
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }



}

```

#### 总结
```
1. BaseExecutor.query() 方法对应了二级缓存
2. 二级缓存没办法设置淘汰策略和限制，有风险
3. 二级缓存主要是 sqlSession 范围内 或者是statment返回，可以实现应用内的 repeatable read 

```


### 二级缓存与 CachingExecutor 
#### Mybatis 默认创建 SimpleExecutor ，当开启一级缓存，会套一个 CachingExecutor (源码见：Configuration.newExecutor() 方法 )
#### 一般默认开启, 当 mybatis.configuration.cache-enabled=false时，Mybatis不会创建 CachingExecutor 

#### CachingExecutor 源码
```

public class CachingExecutor implements Executor {

  private final Executor delegate;
  private final TransactionalCacheManager tcm = new TransactionalCacheManager();

  public CachingExecutor(Executor delegate) {
  	/*
  		delegate 就是 SimpleExecutor，通过装饰着模式，调用 this.query() 完
  		调用 this.delegate.query() 方法
  	*/
    this.delegate = delegate;
    delegate.setExecutorWrapper(this);
  }


  ....省略很多代码

  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    /*
    	封装 CacheKey ，也就是缓存的key ，这里通过整个sql 去hash，
    	可能会有点性能影响，但是如果不设置缓存，好像这个也关不掉，
    	但是经过测试，1k长度的sql 执行50w次，也不过一秒多一点
    */
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }



  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    /*
    	MappedStatement 的cache 配置在每个mapper.xml 中，配置 <cache> 标签
    */
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      /*
      	这条sql是否配置了 useCache 参数 
      */
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, parameterObject, boundSql);
        @SuppressWarnings("unchecked")
        //从缓存中获取数据
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          //调用 delegate 也就是 BaseExecutor 的 query() 方法
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

}

```

#### 总结
```
1. Mybatis 默认使用 CachingExecutor 实现一级缓存
2. MappedStatement 中需要查看 <cache>
3. 在Sql 中配置了 useCache 时，可以缓存结果，应用级别
4. CachingExecutor 的缓存是应用级别，但是可以设置策略和大小，做到安全

```




