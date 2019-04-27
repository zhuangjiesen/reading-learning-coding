# SpringCache框架加载/拦截原理

#### 官网文档
https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#cache

### 背景
```
项目A中需要多数据源的实现，比如UserDao.getAllUserList() 需要从readonly库中读取，但是UserDao.insert() 需要插入主(写)库 
就需要在dao层的方法调用上面添加注解！

```

#### 了解后知道-接口通过jdk代理(mybatis的mapper接口就是通过jdk代理动态生成的-> MapperFactoryBean.class )的，没办法被aop的拦截(注解配置的拦截)
```
    //dao
    @Pointcut("@annotation(com.kaola.cs.data.common.aspect.DataSourceSelect)")
    public void dao() {
    }

```

#### 然后碰巧接触了项目B，使用了SpringCache模块，但是Spring的Cache模块居然能够拦截(spring-cache也是通过注解拦截！！！)
### 引起了我的兴趣，就把源码翻了一遍


### SpringCache的用途
```
与 mybatis 对比
	1.  spring-cache 是基于spring的方法级别的，也就是说你方法做了啥不关心，它只负责缓存方法结果
		mybatis 的缓存(CachingExecutor / BaseExecutor) 是基于数据库查询结果的缓存
	2.  spring-cache 可以配置各种类型的缓存介质(redis , ehcache , hashmap, 甚至db等等) -> 它仅仅是提供接口和默认实现，可以自己拓展
		mybatis 的缓存是hashmap，单一！！lowb

```

### SpringCache 的配置
1.注解(spring-boot)
2.xml配置


### 这里只讲注解，但是初始化的类都是一样的！！！
#### 定义 CacheConfigure.java 就能直接使用
```
@EnableCaching
@Configuration
public class CacheConfigure extends CachingConfigurerSupport {
    @Override
    @Bean
    public CacheManager cacheManager() {
        SimpleCacheManager result = new SimpleCacheManager();
        List<Cache> caches = new ArrayList<>();
        caches.add(new ConcurrentMapCache("testCache"));
        result.setCaches(caches);
        return result;
    }

    @Override
    @Bean
    public CacheErrorHandler errorHandler() {
        return new SimpleCacheErrorHandler();
    }

}


```

### 通过 @EnableCaching 注解可以找到 Spring-Cache 初始化的核心类
#### ProxyCachingConfiguration.java
```

@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyCachingConfiguration extends AbstractCachingConfiguration {

	@Bean(name = CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryCacheOperationSourceAdvisor cacheAdvisor() {
		BeanFactoryCacheOperationSourceAdvisor advisor = new BeanFactoryCacheOperationSourceAdvisor();
		advisor.setCacheOperationSource(cacheOperationSource());
		advisor.setAdvice(cacheInterceptor());
		if (this.enableCaching != null) {
			advisor.setOrder(this.enableCaching.<Integer>getNumber("order"));
		}
		return advisor;
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheOperationSource cacheOperationSource() {
		return new AnnotationCacheOperationSource();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheInterceptor cacheInterceptor() {
		CacheInterceptor interceptor = new CacheInterceptor();
		interceptor.configure(this.errorHandler, this.keyGenerator, this.cacheResolver, this.cacheManager);
		interceptor.setCacheOperationSource(cacheOperationSource());
		return interceptor;
	}

}

```

### 通过注解，把3个类的bean 实例化: BeanFactoryCacheOperationSourceAdvisor 、CacheOperationSource 、 CacheInterceptor

#### 说一下这3个类的作用
#### BeanFactoryCacheOperationSourceAdvisor.java 
```
/*
  BeanFactoryCacheOperationSourceAdvisor 继承了 	AbstractBeanFactoryPointcutAdvisor
  在spring 中的效果就是，在每个bean的初始化时 (每个bean都会被加载成 advised 对象 -> 有 targetSource 和 Advisor[] 数组)
  每个bean被调用方法的时候都是先遍历advisor的方法，然后在调用原生bean(也就是targetSource)的方法，实现了aop的效果

  bean 加载的时候 BeanFactoryCacheOperationSourceAdvisor 的 getPointcut()-> 也就是 CacheOperationSourcePointcut 就会被获取，然后调用 
  CacheOperationSourcePointcut.matches()方法, 用来匹配对应的bean 
  假设bean 在 BeanFactoryCacheOperationSourceAdvisor 的扫描中 matchs() 方法返回了true
  结果就是 
  	在每个bean的方法被调用的时候 CacheInterceptor 中的 invoke() 方法就会被调用 

  总结：
  	spring-cache 也完成了aop一样的实现(spring-aop也是这样做的)

  重点就是在 CacheOperationSourcePointcut.matchs() 方法中，怎么匹配接口的了 这里先不说后面具体介绍！！！！

*/
public class BeanFactoryCacheOperationSourceAdvisor extends AbstractBeanFactoryPointcutAdvisor {

	@Nullable
	private CacheOperationSource cacheOperationSource;

	private final CacheOperationSourcePointcut pointcut = new CacheOperationSourcePointcut() {
		@Override
		@Nullable
		protected CacheOperationSource getCacheOperationSource() {
			return cacheOperationSource;
		}
	};


	/**
	 * Set the cache operation attribute source which is used to find cache
	 * attributes. This should usually be identical to the source reference
	 * set on the cache interceptor itself.
	 */
	public void setCacheOperationSource(CacheOperationSource cacheOperationSource) {
		this.cacheOperationSource = cacheOperationSource;
	}

	/**
	 * Set the {@link ClassFilter} to use for this pointcut.
	 * Default is {@link ClassFilter#TRUE}.
	 */
	public void setClassFilter(ClassFilter classFilter) {
		this.pointcut.setClassFilter(classFilter);
	}

	@Override
	public Pointcut getPointcut() {
		return this.pointcut;
	}

}

```

#### CacheOperationSource.java 是个接口
#### 实现类是 -> AnnotationCacheOperationSource.java 重点是父类 -> AbstractFallbackCacheOperationSource.java
#### 讲解一下：
```
代码量很少，主要是 attributeCache 的封装使用，通过把 method - CacheOperation 
然后在 CacheInterceptor.invoke() 的时候通过invocation 获取到 method-class 然后调用 CacheOperationSource.getCacheOperations() 获取到 CacheOperation
CacheOperation 其实就是触发对应spring-cache 注解的操作-获取缓存的实现了

```

```

public abstract class AbstractFallbackCacheOperationSource implements CacheOperationSource {

	/**
	 * Canonical value held in cache to indicate no caching attribute was
	 * found for this method and we don't need to look again.
	 */
	private static final Collection<CacheOperation> NULL_CACHING_ATTRIBUTE = Collections.emptyList();


	/**
	 * Logger available to subclasses.
	 * <p>As this base class is not marked Serializable, the logger will be recreated
	 * after serialization - provided that the concrete subclass is Serializable.
	 */
	protected final Log logger = LogFactory.getLog(getClass());

	/**
	 * Cache of CacheOperations, keyed by method on a specific target class.
	 * <p>As this base class is not marked Serializable, the cache will be recreated
	 * after serialization - provided that the concrete subclass is Serializable.
	 */
	private final Map<Object, Collection<CacheOperation>> attributeCache = new ConcurrentHashMap<>(1024);


	/**
	 * Determine the caching attribute for this method invocation.
	 * <p>Defaults to the class's caching attribute if no method attribute is found.
	 * @param method the method for the current invocation (never {@code null})
	 * @param targetClass the target class for this invocation (may be {@code null})
	 * @return {@link CacheOperation} for this method, or {@code null} if the method
	 * is not cacheable
	 */
	@Override
	@Nullable
	public Collection<CacheOperation> getCacheOperations(Method method, @Nullable Class<?> targetClass) {
		if (method.getDeclaringClass() == Object.class) {
			return null;
		}

		Object cacheKey = getCacheKey(method, targetClass);
		Collection<CacheOperation> cached = this.attributeCache.get(cacheKey);

		if (cached != null) {
			return (cached != NULL_CACHING_ATTRIBUTE ? cached : null);
		}
		else {
			Collection<CacheOperation> cacheOps = computeCacheOperations(method, targetClass);
			if (cacheOps != null) {
				if (logger.isTraceEnabled()) {
					logger.trace("Adding cacheable method '" + method.getName() + "' with attribute: " + cacheOps);
				}
				this.attributeCache.put(cacheKey, cacheOps);
			}
			else {
				this.attributeCache.put(cacheKey, NULL_CACHING_ATTRIBUTE);
			}
			return cacheOps;
		}
	}

	/**
	 * Determine a cache key for the given method and target class.
	 * <p>Must not produce same key for overloaded methods.
	 * Must produce same key for different instances of the same method.
	 * @param method the method (never {@code null})
	 * @param targetClass the target class (may be {@code null})
	 * @return the cache key (never {@code null})
	 */
	protected Object getCacheKey(Method method, @Nullable Class<?> targetClass) {
		return new MethodClassKey(method, targetClass);
	}

	@Nullable
	private Collection<CacheOperation> computeCacheOperations(Method method, @Nullable Class<?> targetClass) {
		// Don't allow no-public methods as required.
		if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
			return null;
		}

		// The method may be on an interface, but we need attributes from the target class.
		// If the target class is null, the method will be unchanged.
		Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

		// First try is the method in the target class.
		Collection<CacheOperation> opDef = findCacheOperations(specificMethod);
		if (opDef != null) {
			return opDef;
		}

		// Second try is the caching operation on the target class.
		opDef = findCacheOperations(specificMethod.getDeclaringClass());
		if (opDef != null && ClassUtils.isUserLevelMethod(method)) {
			return opDef;
		}

		if (specificMethod != method) {
			// Fallback is to look at the original method.
			opDef = findCacheOperations(method);
			if (opDef != null) {
				return opDef;
			}
			// Last fallback is the class of the original method.
			opDef = findCacheOperations(method.getDeclaringClass());
			if (opDef != null && ClassUtils.isUserLevelMethod(method)) {
				return opDef;
			}
		}

		return null;
	}


	/**
	 * Subclasses need to implement this to return the caching attribute for the
	 * given class, if any.
	 * @param clazz the class to retrieve the attribute for
	 * @return all caching attribute associated with this class, or {@code null} if none
	 */
	@Nullable
	protected abstract Collection<CacheOperation> findCacheOperations(Class<?> clazz);

	/**
	 * Subclasses need to implement this to return the caching attribute for the
	 * given method, if any.
	 * @param method the method to retrieve the attribute for
	 * @return all caching attribute associated with this method, or {@code null} if none
	 */
	@Nullable
	protected abstract Collection<CacheOperation> findCacheOperations(Method method);

	/**
	 * Should only public methods be allowed to have caching semantics?
	 * <p>The default implementation returns {@code false}.
	 */
	protected boolean allowPublicMethodsOnly() {
		return false;
	}

}



```


#### ！！！！  CacheOperationSourcePointcut.java 的 matchs() 方法
#### 用来判断类是不是符合spring-cache 拦截条件 也就是 @Cachable @CachePut 等等的注解怎么识别的地方
#### 经过跟踪代码发现是 AnnotationCacheOperationSource.findCacheOperations() 调用的
#### 省略部分代码....
```

public class AnnotationCacheOperationSource extends AbstractFallbackCacheOperationSource implements Serializable {


	private final Set<CacheAnnotationParser> annotationParsers;


	@Override
	@Nullable
	protected Collection<CacheOperation> findCacheOperations(Class<?> clazz) {
		return determineCacheOperations(parser -> parser.parseCacheAnnotations(clazz));
	}

	@Override
	@Nullable
	protected Collection<CacheOperation> findCacheOperations(Method method) {
		return determineCacheOperations(parser -> parser.parseCacheAnnotations(method));
	}

	/**
	 * Determine the cache operation(s) for the given {@link CacheOperationProvider}.
	 * <p>This implementation delegates to configured
	 * {@link CacheAnnotationParser CacheAnnotationParsers}
	 * for parsing known annotations into Spring's metadata attribute class.
	 * <p>Can be overridden to support custom annotations that carry caching metadata.
	 * @param provider the cache operation provider to use
	 * @return the configured caching operations, or {@code null} if none found
	 */
	@Nullable
	protected Collection<CacheOperation> determineCacheOperations(CacheOperationProvider provider) {
		Collection<CacheOperation> ops = null;
		for (CacheAnnotationParser annotationParser : this.annotationParsers) {
			Collection<CacheOperation> annOps = provider.getCacheOperations(annotationParser);
			if (annOps != null) {
				if (ops == null) {
					ops = annOps;
				}
				else {
					Collection<CacheOperation> combined = new ArrayList<>(ops.size() + annOps.size());
					combined.addAll(ops);
					combined.addAll(annOps);
					ops = combined;
				}
			}
		}
		return ops;
	}


}



```

#### 然后就是注解的解析方法 SpringCacheAnnotationParser.java
#### 代码很简单-就不多说了
```


	@Nullable
	private Collection<CacheOperation> parseCacheAnnotations(
			DefaultCacheConfig cachingConfig, AnnotatedElement ae, boolean localOnly) {

		Collection<? extends Annotation> anns = (localOnly ?
				AnnotatedElementUtils.getAllMergedAnnotations(ae, CACHE_OPERATION_ANNOTATIONS) :
				AnnotatedElementUtils.findAllMergedAnnotations(ae, CACHE_OPERATION_ANNOTATIONS));
		if (anns.isEmpty()) {
			return null;
		}

		final Collection<CacheOperation> ops = new ArrayList<>(1);
		anns.stream().filter(ann -> ann instanceof Cacheable).forEach(
				ann -> ops.add(parseCacheableAnnotation(ae, cachingConfig, (Cacheable) ann)));
		anns.stream().filter(ann -> ann instanceof CacheEvict).forEach(
				ann -> ops.add(parseEvictAnnotation(ae, cachingConfig, (CacheEvict) ann)));
		anns.stream().filter(ann -> ann instanceof CachePut).forEach(
				ann -> ops.add(parsePutAnnotation(ae, cachingConfig, (CachePut) ann)));
		anns.stream().filter(ann -> ann instanceof Caching).forEach(
				ann -> parseCachingAnnotation(ae, cachingConfig, (Caching) ann, ops));
		return ops;
	}


```


### 总结
```
1.spring-cache 实现了 AbstractBeanFactoryPointcutAdvisor 提供 CacheOperationSourcePointcut (PointCut) 作切点判断，提供 CacheInterceptor (MethodInterceptor) 作方法拦截
2.spring-cache 提供 CacheOperationSource 作为 method 对应 CacheOperation(缓存操作) 的查询和加载
3.spring-cache 通过 SpringCacheAnnotationParser 来解析自己定义的 @Cacheable @CacheEvict @Caching 等注解类
所以 spring-cache 不使用 aspectj 的方式，通过 CacheOperationSource.getCacheOperations() 方式可以使jdk代理的类也能匹配到
```

### jdk代理的类的匹配
#### 代码类在 CacheOperationSource.getCacheOperations() 
#### 重点在于 targetClass 和 method ，如果是对应的 dao.xxx() 就能matchs() 并且拦截
#### CacheInterceptor -> CacheAspectSupport.execute() 方法
```
// 代码自己看吧。也很简单 -> 结果就是spring-cache 也可以拦截到mybatis的dao层接口，进行缓存

	@Nullable
	protected Object execute(CacheOperationInvoker invoker, Object target, Method method, Object[] args) {
		// Check whether aspect is enabled (to cope with cases where the AJ is pulled in automatically)
		if (this.initialized) {
			Class<?> targetClass = getTargetClass(target);
			CacheOperationSource cacheOperationSource = getCacheOperationSource();
			if (cacheOperationSource != null) {
				Collection<CacheOperation> operations = cacheOperationSource.getCacheOperations(method, targetClass);
				if (!CollectionUtils.isEmpty(operations)) {
					return execute(invoker, method,
							new CacheOperationContexts(operations, method, args, target, targetClass));
				}
			}
		}

		return invoker.invoke();
	}



```








