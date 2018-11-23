# mybatis 的 org.mybatis.spring.mapper.MapperScannerConfigurer 加载流程


学习mybatis 源码时，清楚底层是通过动态代理实现对接口代理，映射到具体的sql ,但是还是自己打断点一步一步分析出加载过程和执行过程；
看代码过程中产生的迷惑
迷惑：
1.把bean 扫描出来，如何判断 bean是不是mybatis 的dao 类型 （因为用的都是spring 的bean的注解）
2.动态代理的入口在哪
3. dao.xml 扫描的入口在哪
4. dao.java 和 dao.xml 是否有先后加载的关系 

仔细断点获得的解答：
1.通过doScan() 方法中将接口并且没有被实现过的类扫描出来，将扫描到的 BeanDefinition 设置了 ，mapperInterface属性为当前接口的类， 设置了 bean 的类为  MapperFactoryBean.class 类型，类 MapperFactoryBean 实现 FactoryBean 所以在 getObject() 方法中就是完成动态代理的入口，
而其父类 DaoSupports 实现了 InitializingBean ，则初始化前会执行 afterPropertySet() 方法，这时候就是触发判断 bean 类型是不是 mybatis要用到的dao (通过查找或者临时加载 dao.xml 能找到对应的 dao.xml )则说明符合
2.MapperFactoryBean 实现 FactoryBean 所以在 getObject() 方法中就是完成动态代理的入口;
3.MapperScannerConfigurer 的加载 MapperFactoryBean.afterPropertySet() -> checkDaoConfig() -> configuration.addMapper(this.mapperInterface); -> mapperRegistry.addMapper(type); 最后找到加载的入口 
还有 sqlSessionFactoryBean 加载 mybatis-config.xml 中配置的mappers
4. dao.xml 也可以先加载，dao.java加载完会去查找dao.xml(根据namespace) ，若查不到会进入 dao.java 的类目录下进行查找




类 MapperScannerConfigurer 实现 BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware 接口


配置上只需要配置加载加载的包、和 sqlSessionFactory 的参数

```
  <!-- 配置 扫描器-->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <!-- 扫描com.ouc.dao这个包以及它的子包下的所有映射接口类 -->
        <property name="basePackage" value="com.java" />
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
    </bean>

```

扫描类 MapperScannerConfigurer 的入口方法：
```

  /**
   * {@inheritDoc}
   * 
   * @since 1.0.2
   */
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }
	/*
		初始化扫描器参数 用来装配和代理，mybatis的接口

	*/ 
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.registerFilters();
    // 重点是 scan 方法 扫描并实现动态代理生成对应 mybatis 接口类的实现
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }

```


scanner.scan() 方法:


```

	/**
	 * Perform a scan within the specified base packages.
	 * @param basePackages the packages to check for annotated classes
	 * @return number of beans registered
	 */
	public int scan(String... basePackages) {
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
		

		/*
			这一步重要了 ， 这里调用的是子类 ClassPathMapperScanner 实现的 doScan() 方法，当然 ClassPathMapperScanner 类的doScan() 方法也调用了父类的 doScan() 将配置包目录下的所有类加载到 set 集合中

		*/ 
		doScan(basePackages);

		// Register annotation config processors, if necessary.
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}

		return this.registry.getBeanDefinitionCount() - beanCountAtScanStart;
	}

```

ClassPathMapperScanner 的 doScan() 方法
#### 这部分主要是把扫描出的类 封装成 GenericBeanDefinition 类型
设置
BeanDefinition 类的 mapperInterface 为它本身 （mybatis的dao 类型都是接口类型，扫描时会进行判断是否接口，并无实现类）
设置这个bean 的类型为 MapperFactoryBean (等下讲的重点)
addToConfig 这个参数用来控制是否加到 configure 类中
sqlSessionFactory 

``` 


 /**
   * Calls the parent search that will search and register all the candidates.
   * Then the registered objects are post processed to set them as
   * MapperFactoryBeans
   */
  @Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
      for (BeanDefinitionHolder holder : beanDefinitions) {
        GenericBeanDefinition definition = (GenericBeanDefinition) holder.getBeanDefinition();

        if (logger.isDebugEnabled()) {
          logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() 
              + "' and '" + definition.getBeanClassName() + "' mapperInterface");
        }

        // the mapper interface is the original class of the bean
        // but, the actual class of the bean is MapperFactoryBean
        definition.getPropertyValues().add("mapperInterface", definition.getBeanClassName());
        definition.setBeanClass(MapperFactoryBean.class);

        definition.getPropertyValues().add("addToConfig", this.addToConfig);

        boolean explicitFactoryUsed = false;
        if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
          definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
          explicitFactoryUsed = true;
        } else if (this.sqlSessionFactory != null) {
          definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
          explicitFactoryUsed = true;
        }

        if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
          if (explicitFactoryUsed) {
            logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
          }
          definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
          explicitFactoryUsed = true;
        } else if (this.sqlSessionTemplate != null) {
          if (explicitFactoryUsed) {
            logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
          }
          definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
          explicitFactoryUsed = true;
        }

        if (!explicitFactoryUsed) {
          if (logger.isDebugEnabled()) {
            logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
          }
          definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
        }
      }
    }

    return beanDefinitions;
  }

```



MapperFactoryBean 类关系

```
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
	....
	/*
		可以看出 MapperFactoryBean 实现了 FactoryBean 
		所以有三个方法 isSingle() , getObjectType() , getObject();

		getObjectType() 就是讲 mapperInterface 类型返回，
		getObject() 就是对 mapperInterface 接口进行动态代理，生成该接口实现类，注册到spring bean 容器中，
		使得项目可以通过调用接口类来进行Mybatis的操作


		
	*/
}
```

getObejct() 方法也是个重点等下说
先说
MapperFactoryBean 类继承 SqlSessionDaoSupport 继承了 DaoSupport 
类 DaoSupport 实现了 InitializingBean 接口， 所以只要是这个 MapperFactoryBean 注册到spring 中就会先执行 afterPropertiesSet() 方法

```
	DaoSupport 的 afterPropertiesSet()方法：

    public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
    	/*
    		这个方法用来查找dao接口类对应的xml配置，或者注解配置 对应实现在 MapperFactoryBean的 checkDaoConfig() 方法中

    	*/ 
        this.checkDaoConfig();

        try {
            this.initDao();
        } catch (Exception var2) {
            throw new BeanInitializationException("Initialization of DAO failed", var2);
        }
    }

```

MapperFactoryBean的 checkDaoConfig() 代码
重点在 configuration.addMapper(this.mapperInterface); 这个步骤

```


  /**
   * {@inheritDoc}
   */
  @Override
  protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        configuration.addMapper(this.mapperInterface);
      } catch (Throwable t) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", t);
        throw new IllegalArgumentException(t);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }


```

一步步点进去源码就会到类 MapperRegistry 的 addMapper() 方法:


```


  public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }


```

接着就是 MapperAnnotationBuilder 类的 parse() 方法，在这个类中，就是进行 mybatis 的 dao 接口配置对应的 xml 或者注解上配置的sql , resultMap , 还有 select insert update delete 标签下的 sql ， paramaterType , resultMap 等参数，加载映射到 Configure 类下，等于最后是汇总到一个类中，所以namespace 不能重复 (因为接口类不能重复)， 初始化成功后
configure 加载出 prepareStatement ， 项目运行时调用数据库执行时，就是根据 namespace+id(每个查询配置的id ) 作为key 去查找到相应的 prepareStatement 然后执行 ， 具体可以看  org.apache.ibatis.session.Configuration 类



```


  public void parse() {
    String resource = type.toString();
    if (!configuration.isResourceLoaded(resource)) {
    	/*
    		加载对应的namespace的 xml dao配置
    	*/
      loadXmlResource();
      configuration.addLoadedResource(resource);
      assistant.setCurrentNamespace(type.getName());
      parseCache();
      parseCacheRef();
      Method[] methods = type.getMethods();
      for (Method method : methods) {
        try {
          if (!method.isBridge()) { // issue #237
            parseStatement(method);
          }
        } catch (IncompleteElementException e) {
          configuration.addIncompleteMethod(new MethodResolver(this, method));
        }
      }
    }
    parsePendingMethods();
  }



  ....
	
	/*
		这个类方法下的加载dao 接口类下的目录的同名 xml 文件


	*/
  private void loadXmlResource() {
    // Spring may not know the real resource name so we check a flag
    // to prevent loading again a resource twice
    // this flag is set at XMLMapperBuilder#bindMapperForNamespace
    if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
      String xmlResource = type.getName().replace('.', '/') + ".xml";
      InputStream inputStream = null;
      try {
        inputStream = Resources.getResourceAsStream(type.getClassLoader(), xmlResource);
      } catch (IOException e) {
        // ignore, resource is not required
      }
      if (inputStream != null) {
        XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource, configuration.getSqlFragments(), type.getName());
        xmlParser.parse();
      }
    }
  }

```


所以dao.xml 有两种配置方法
1.mybatis-config.xml 配置：

```
<configuration>
        ....省略 
        <mappers>
                <mapper resource="ResourceCrossingInfoDao.xml"></mapper>
                <mapper resource="ResourceCrossingLaneInfoDao.xml"></mapper>
                <mapper resource="VehicleIllegalRecordDao.xml"></mapper>
                <mapper resource="VehicleRecordDao.xml"></mapper>

        </mappers>
</configuration>
```
这种配置 configuration.isResourceLoaded() 方法就会返回 true 

2.将 dao.xml 文件放在和 dao.java 接口类同个包下


到这里 afterPropertiesSet() 方法执行完，dao.xml 配置已经加载完毕；
接着就是进行动态代理的类 MapperFactoryBean 的 getObject() 方法 
```


  /**
   * {@inheritDoc}
   */
  public T getObject() throws Exception {

  	/*
		这里的sqlSession 的实现是 DefaultSqlSession 类


  	*/
    return getSqlSession().getMapper(this.mapperInterface);
  }


```


类 DefaultSqlSession 的 getMapper() 方法

```

  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }

```
configuration 类的 getMapper() 方法:

```
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }

```
mapperRegistry 的 getMapper() 方法：
终于看到重点了，通过 mapperProxyFactory 类进行 jdk 动态代理

```
  @SuppressWarnings("unchecked")
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null)
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }

```

接着就是在 MapperProxyFactory 类中进行动态代理，MapperProxy 是代理类：
接下去就明朗了
```

	/**
	 * @author Clinton Begin
	 * @author Eduardo Macarron
	 */
	public class MapperProxy<T> implements InvocationHandler, Serializable {

	  private static final long serialVersionUID = -6424540398559729838L;
	  private final SqlSession sqlSession;
	  private final Class<T> mapperInterface;
	  private final Map<Method, MapperMethod> methodCache;

	  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
	    this.sqlSession = sqlSession;
	    this.mapperInterface = mapperInterface;
	    this.methodCache = methodCache;
	  }

	  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	    if (Object.class.equals(method.getDeclaringClass())) {
	      try {
	        return method.invoke(this, args);
	      } catch (Throwable t) {
	        throw ExceptionUtil.unwrapThrowable(t);
	      }
	    }
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
