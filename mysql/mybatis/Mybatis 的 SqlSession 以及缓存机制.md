# Mybatis 的 SqlSession 以及缓存机制


### Executor 执行器
```
Mybatis 中处理 SQL的地方

Mybatis 中将配置和SQL解析并调用，获取结果
如方法：
<E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;


```

#### Executor 初始化
```
Configuration 中默认是 SIMPLE 的执行器 SimpleExecutor 

//Mybatis 全局的配置类
public class Configuration {
	protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
	.... 省略很多代码
}

```


#### 但是之后在生成 Executor 的 newExecutor() 方法中 
```

  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    //默认缓存是开启的，等于 Executor 最后会被设置成 CachingExecutor
    // 这里的 executor 是 SimpleExecutor ，所以 CachingExecutor 中的 delegate 是 SimpleExecutor
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }

```

#### 所以 Executor 最后会初始化为 CachingExecutor
```
CachingExecutor 中的 delegate 是 SimpleExecutor
```


### MapperFactoryBean 
```
我们都知道 Mybatis 的策略就是在 Spring 启动的时候识别 @Mapper 标识的 Mapper接口
通过 MapperFactoryBean 进行动态代理，生成实现接口，代理实现SQL查询

```

#### 代码：

```

public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {

  private Class<T> mapperInterface;

  private boolean addToConfig = true;

  public MapperFactoryBean() {
    //intentionally empty 
  }
  
  public MapperFactoryBean(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  /**
   * {@inheritDoc}
   *
   * 这里是 afterPropertiesSet() 方法触发的
   * 
   */
  @Override
  protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
      	//将解析到的mapper加载到 configuration 中的mapper中
      	//后续通过 map 获取id 可以去到对应的 MapperProxyFactory 进行代理
        configuration.addMapper(this.mapperInterface);
      } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }

  /**
   * {@inheritDoc}
   * bean实例化 ， 根据 FactoryBean 的特性 ，获取到mapper接口时，会调用 getObject() 方法
   * 这里的重点是 getSqlSession() -> SqlSessionTemplate 对象
   */
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public Class<T> getObjectType() {
    return this.mapperInterface;
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public boolean isSingleton() {
    return true;
  }

  //------------- mutators --------------

  /**
   * Sets the mapper interface of the MyBatis mapper
   *
   * @param mapperInterface class of the interface
   */
  public void setMapperInterface(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  /**
   * Return the mapper interface of the MyBatis mapper
   *
   * @return class of the interface
   */
  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  /**
   * If addToConfig is false the mapper will not be added to MyBatis. This means
   * it must have been included in mybatis-config.xml.
   * <p/>
   * If it is true, the mapper will be added to MyBatis in the case it is not already
   * registered.
   * <p/>
   * By default addToCofig is true.
   *
   * @param addToConfig
   */
  public void setAddToConfig(boolean addToConfig) {
    this.addToConfig = addToConfig;
  }

  /**
   * Return the flag for addition into MyBatis config.
   *
   * @return true if the mapper will be added to MyBatis in the case it is not already
   * registered.
   */
  public boolean isAddToConfig() {
    return addToConfig;
  }
}

```

#### 结论
```
1. SqlSession 对象在 MapperFactoryBean 中的对象是 SqlSessionTemplate
2. 重点是在 getObject() 方法中进行动态代理

```

### SqlSessionTemplate 全局的dao层的执行
#### Mybatis 通过 SqlSession(SqlSessionTemplate 是实现类) 调用 mybatis 封装和配置的数据库操作
#### 在 SqlSessionTemplate 的构造函数中，通过构造函数，设置了一个动态代理如下
```


  /**
   * Constructs a Spring managed {@code SqlSession} with the given
   * {@code SqlSessionFactory} and {@code ExecutorType}.
   * A custom {@code SQLExceptionTranslator} can be provided as an
   * argument so any {@code PersistenceException} thrown by MyBatis
   * can be custom translated to a {@code RuntimeException}
   * The {@code SQLExceptionTranslator} can also be null and thus no
   * exception translation will be done and MyBatis exceptions will be
   * thrown
   *
   * @param sqlSessionFactory
   * @param executorType
   * @param exceptionTranslator
   */
  public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    //这里设置了 CacheExecutor 的执行器
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    //这里通过JDK 动态代理生成了一个代理类
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
  }


  //....省略部分代码
  //....后续的操作都是通过代理类 sqlSessionProxy 拦截的


  /**
   * {@inheritDoc}
   */
  @Override
  public <T> T selectOne(String statement, Object parameter) {
    return this.sqlSessionProxy.<T> selectOne(statement, parameter);
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public <K, V> Map<K, V> selectMap(String statement, String mapKey) {
    return this.sqlSessionProxy.<K, V> selectMap(statement, mapKey);
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey) {
    return this.sqlSessionProxy.<K, V> selectMap(statement, parameter, mapKey);
  }


```


### SqlSessionInterceptor 用途

#### SqlSessionTemplate 的一个内部类，用来实现创建 SqlSession ，并且能够与事务关联
#### 为了解决定义的一个 Transaction 中，使用的是 同一个 SqlSession

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
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
      	//调用的是 DefaultSqlSession 的原来的方法
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

#### 结论
```
1. 如果声明了 @Transactional , 在同一个 @Transactional 范围内，获取到的 sqlSession 将会是同一个；
2. SqlSessionTemplate 其实是个工厂模式，每次请求是通过 proxy 去获取 SqlSession；
3. 原本以为全局单例的 SqlSession ，通过 SqlSessionInterceptor 代码看，并不是；

```


### SqlSessionTemplate 的初始化
#### 在 SqlSessionFactoryBean 的实例化时

#### SqlSessionFactoryBean 的代码：
```

  /**
   * {@inheritDoc}
   */
  @Override
  public void afterPropertiesSet() throws Exception {
    notNull(dataSource, "Property 'dataSource' is required");
    notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
    state((configuration == null && configLocation == null) || !(configuration != null && configLocation != null),
              "Property 'configuration' and 'configLocation' can not specified with together");

    this.sqlSessionFactory = buildSqlSessionFactory();
  }



  /**
   * Build a {@code SqlSessionFactory} instance.
   *
   * The default implementation uses the standard MyBatis {@code XMLConfigBuilder} API to build a
   * {@code SqlSessionFactory} instance based on an Reader.
   * Since 1.3.0, it can be specified a {@link Configuration} instance directly(without config file).
   *
   * @return SqlSessionFactory
   * @throws IOException if loading the config file failed
   */
  protected SqlSessionFactory buildSqlSessionFactory() throws IOException {

    Configuration configuration;

    XMLConfigBuilder xmlConfigBuilder = null;
    if (this.configuration != null) {
      configuration = this.configuration;
      if (configuration.getVariables() == null) {
        configuration.setVariables(this.configurationProperties);
      } else if (this.configurationProperties != null) {
        configuration.getVariables().putAll(this.configurationProperties);
      }
    } else if (this.configLocation != null) {
      xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
      configuration = xmlConfigBuilder.getConfiguration();
    } else {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Property 'configuration' or 'configLocation' not specified, using default MyBatis Configuration");
      }
      configuration = new Configuration();
      if (this.configurationProperties != null) {
        configuration.setVariables(this.configurationProperties);
      }
    }

    if (this.objectFactory != null) {
      configuration.setObjectFactory(this.objectFactory);
    }

    if (this.objectWrapperFactory != null) {
      configuration.setObjectWrapperFactory(this.objectWrapperFactory);
    }

    if (this.vfs != null) {
      configuration.setVfsImpl(this.vfs);
    }

    if (hasLength(this.typeAliasesPackage)) {
      String[] typeAliasPackageArray = tokenizeToStringArray(this.typeAliasesPackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      for (String packageToScan : typeAliasPackageArray) {
        configuration.getTypeAliasRegistry().registerAliases(packageToScan,
                typeAliasesSuperType == null ? Object.class : typeAliasesSuperType);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Scanned package: '" + packageToScan + "' for aliases");
        }
      }
    }

    if (!isEmpty(this.typeAliases)) {
      for (Class<?> typeAlias : this.typeAliases) {
        configuration.getTypeAliasRegistry().registerAlias(typeAlias);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registered type alias: '" + typeAlias + "'");
        }
      }
    }

    if (!isEmpty(this.plugins)) {
      for (Interceptor plugin : this.plugins) {
        configuration.addInterceptor(plugin);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registered plugin: '" + plugin + "'");
        }
      }
    }

    if (hasLength(this.typeHandlersPackage)) {
      String[] typeHandlersPackageArray = tokenizeToStringArray(this.typeHandlersPackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      for (String packageToScan : typeHandlersPackageArray) {
        configuration.getTypeHandlerRegistry().register(packageToScan);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Scanned package: '" + packageToScan + "' for type handlers");
        }
      }
    }

    if (!isEmpty(this.typeHandlers)) {
      for (TypeHandler<?> typeHandler : this.typeHandlers) {
        configuration.getTypeHandlerRegistry().register(typeHandler);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registered type handler: '" + typeHandler + "'");
        }
      }
    }

    if (this.databaseIdProvider != null) {//fix #64 set databaseId before parse mapper xmls
      try {
        configuration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
      } catch (SQLException e) {
        throw new NestedIOException("Failed getting a databaseId", e);
      }
    }

    if (this.cache != null) {
      configuration.addCache(this.cache);
    }

    if (xmlConfigBuilder != null) {
      try {
        xmlConfigBuilder.parse();

        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Parsed configuration file: '" + this.configLocation + "'");
        }
      } catch (Exception ex) {
        throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
      } finally {
        ErrorContext.instance().reset();
      }
    }

    if (this.transactionFactory == null) {
      this.transactionFactory = new SpringManagedTransactionFactory();
    }

    configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));

    if (!isEmpty(this.mapperLocations)) {
      for (Resource mapperLocation : this.mapperLocations) {
        if (mapperLocation == null) {
          continue;
        }

        try {
          XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
              configuration, mapperLocation.toString(), configuration.getSqlFragments());
          xmlMapperBuilder.parse();
        } catch (Exception e) {
          throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
        } finally {
          ErrorContext.instance().reset();
        }

        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Parsed mapper file: '" + mapperLocation + "'");
        }
      }
    } else {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Property 'mapperLocations' was not specified or no matching resources found");
      }
    }

    return this.sqlSessionFactoryBuilder.build(configuration);
  }


```
#### 最后的 this.sqlSessionFactoryBuilder.build(configuration) 创建了 SqlSessionFactory

#### SqlSessionFactoryBuilder 的代码：

```
  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }

```

### 之后会调用 SqlSessionDaoSupport 的 setSqlSessionFactory()
#### 使得 MapperFactoryBean 的 sqlSession 是 SqlSessionTemplate
```

  public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
    if (!this.externalSqlSession) {
      this.sqlSession = new SqlSessionTemplate(sqlSessionFactory);
    }
  }


```

## Mybatis 的缓存
### 都说 Mybatis 有2层缓存，源码中具体在哪？
```
二级缓存：
全局开启配置Spring-boot中 mybatis.configuration.cache-enabled=true (默认true)
这时候系统中的 Executor 就是 CachingExecutor (其中 delegate 是 SimpleExecutor) 的 query() 方法
一级缓存：
在mapper.xml 中配置 flushCache与useCache 相对应的信息就可以开启
对应源码中的 BaseExecutor(SimpleExecutor 实现) 的 query() 方法

```

### 结论
```

1. 全局开启 mybatis.configuration.cache-enabled 的时候，系统的 Executor 是 CachingExecutor (其中 delegate 是 SimpleExecutor)；
2. MapperFactoryBean 中的 SqlSession 的实现 SqlSessionTemplate；
3. SqlSessionTemplate 中通过JDK 动态代理 SqlSessionInterceptor 拦截 SqlSession 的方法 ；
4. SqlSessionInterceptor 用来管理声明事务后，需要使用同一个 SqlSession (动态代理)；
5. 二级缓存在 CachingExecutor.query() 方法中，配置 mybatis.configuration.cache-enabled=true (默认true)；
6. 一级缓存在 BaseExecutor(SimpleExecutor 实现) 的 query() 方法中, xml文件中配置 flushCache 与 useCache 属性；

```

### Mybatis 中缓存范围(一级、二级缓存的作用域)




