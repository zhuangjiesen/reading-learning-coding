# Spring的aspect无法拦截有注解的jdk代理的接口方法的原因

### 背景
```
项目A中需要多数据源的实现，比如UserDao.getAllUserList() 需要从readonly库中读取，但是UserDao.insert() 需要插入主(写)库 
就需要在dao层的方法调用上面添加注解！

```
### 介绍无法代理现象

#### 定义注解 DataNotification.java
```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DataNotification {
}

```

#### 定义mybatis的mapper接口
```
@Mapper
public interface UserMapper {

    @DataNotification
    @CachePut(cacheNames = "testCache", key = "11111")
    public List<User> selectAllUserList();

    @Select("select * from `user` where user_id =#{id}")
    @DataNotification
    public User getUser(Long id);

}

```

#### 然后定义了一个@Aspect 为了拦截注解 @DataNotification 
```
@Aspect
@Component
public class DaoAspect {

    @Pointcut("@annotation(com.jason.core.annotation.DataNotification)")
    public void daoAnnotation(){}

    @Around(value = "daoAnnotation()")
    public void around(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println();
        pjp.proceed();
    }

}

```

## 结果
### 在 UserMapper.selectAllUserList() 方法被调用的时候，并没有触发 DaoAspect.around() 方法

## 为什么呢！！！！

### 讲解


### @Aspect 的注解
#### 解析类是: InstantiationModelAwarePointcutAdvisorImpl
### jdk生成的代理类 Proxy$xxx.class 无法被aop解析
#### 源码
解析注解的类( AnnotationFinder )：
	Java15AnnotationFinder

	1.AbstractAutoProxyCreator.postProcessBeforeInstantiation()
		AbstractAdvisorAutoProxyCreator.findEligibleAdvisors(Class<?> beanClass, String beanName) 用来扫描方法类对应符合的advisor
		AopUtils.findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) 判断bean和对应advisor 能否对应
		这里for 循环 advisor列表(这里还是所有adivor都在) -> 通过 canApply(candidate, clazz, hasIntroductions) -> 筛选出符合的advisor 
			- canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions)ß
				先判断类是否符合
				在遍历method 判断是否符合
				重点判断注解的代码 

					！！！没有jdk代理的 matchesExecution() 时，把method转成 member 里有注解
					所以- 通过打断点在 AspectJExpressionPointcut.getShadowMatch(Method targetMethod, Method originalMethod) 发现 originalMethod 的注解转到 targetMethod 注解就没了
					然后接着网上翻，发现是 AspectJExpressionPointcut.getTargetShadowMatch() 这个方法里生成的 targetMethod 并调用 getShadowMatch() 的
					然后重点就是生成 targetMethod 的地方 Method targetMethod = AopUtils.getMostSpecificMethod(method, targetClass);
					这个方法:

					``` 
						//jdk 的 targetClass 是Proxy&xxx 匿名类 UserMapper.class 可能会是 Proxy$xxx.class
						//cglib 的 targetClass 是原生的类，就是 TestService就是TestServcice.class
						public static Method getMostSpecificMethod(Method method, @Nullable Class<?> targetClass) {
							if (targetClass != null && targetClass != method.getDeclaringClass() && isOverridable(method, targetClass)) {
								try {
									if (Modifier.isPublic(method.getModifiers())) {
										try {
											//这个是获取生成的新 proxy$xxx 实例的方法 -
											return targetClass.getMethod(method.getName(), method.getParameterTypes());
										}
										catch (NoSuchMethodException ex) {
											return method;
										}
									}
									else {
										Method specificMethod =
												ReflectionUtils.findMethod(targetClass, method.getName(), method.getParameterTypes());
										return (specificMethod != null ? specificMethod : method);
									}
								}
								catch (SecurityException ex) {
									// Security settings are disallowing reflective access; fall back to 'method' below.
								}
							}
							return method;
						}

					```
					总结：
						1.jdk代理后生成的是继承 Proxy 实现目标接口的类
						比如
						```
						class $Proxy0 extends java.lang.reflect.Proxy implements com.java.core.proxy.jdk.Subject
						新生成的动态类是不会带注解的，所以通过反射获取不到

						2.cglib 代理生成是直接继承目标类的
						比如
						class RealSubject$$EnhancerByCGLIB$$f0c1daec extends com.java.core.proxy.cglib.RealSubject implements net.sf.cglib.proxy.Factory

						```
						所以生成的类还是当前目标类的子类，所以通过反射还是获取得到注解


					1. AnnotationPointcut.fastMatch() -> AnnotationPointcut.matchInternal()
						判断 ResolvedMemberImpl.resolve(World world);

		Expression 和 targetClass 的校验 
			1. AspectJExpressionPointcut.matches(Method method, @Nullable Class<?> targetClass, boolean hasIntroductions)
			2. PointcutExpressionImpl.couldMatchJoinPointsInType(class) 
			注解PointCut 的识别
				1. AnnotationPointcut.fastMatch()
	2.
		AopUtils.canApply



## 动态代理的原理 
### jdk 代理

#### 被代理接口
```
public interface Subject {

	@ProxyTag
	public void show();

}

```

#### InvocationHandler 的拦截类
```
public class SubjectProxyHandler implements InvocationHandler {
	private Object proxySubject;
	
	public SubjectProxyHandler(Object proxySubject) {
		super();
		this.proxySubject = proxySubject;
	}

	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		Method[] methods = proxy.getClass().getMethods();
		Method[] declaredMethods = proxy.getClass().getDeclaredMethods();
		// TODO Auto-generated method stub
		Annotation[] annotations = method.getDeclaredAnnotations();

		System.out.println("dosomething___before");
		method.invoke(proxySubject, args);
		
		
		System.out.println("dosomething___after");

		
		
		return null;
	}

}


```

#### 启动代理
```
public class ProxyDemo {

	public static void main(String[] args) {
		// TODO Auto-generated method stub

		System.out.println("我是动态代理！！！");
		System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
		 
		RealSubject realSubject=new RealSubject();
		Subject subject=(Subject)Proxy.newProxyInstance
				(RealSubject.class.getClassLoader(), 
						realSubject.getClass().getInterfaces(), 
						new SubjectProxyHandler(realSubject));
		subject.show();
		Subject subject01=new RealSubject();
		
		System.out.println("------------------");
		subject01.show();
		
	}
}


```

### 生成的类
```

public final class $Proxy0 extends java.lang.reflect.Proxy implements com.java.core.proxy.jdk.Subject {
    private static java.lang.reflect.Method m1;
    private static java.lang.reflect.Method m3;
    private static java.lang.reflect.Method m2;
    private static java.lang.reflect.Method m0;

    public $Proxy0(java.lang.reflect.InvocationHandler invocationHandler) { /* compiled code */ }

    public final boolean equals(java.lang.Object o) { /* compiled code */ }

    public final void show() { /* compiled code */ }

    public final java.lang.String toString() { /* compiled code */ }

    public final int hashCode() { /* compiled code */ }
}
```

### 小结
```
1.jdk 动态后的新类是不带注解的

```


### cglib 代理
#### 被代理的类

```
public class RealSubject implements Subject {

	@Override
	@ProxyTag
	public void show() {
		// TODO Auto-generated method stub
		System.out.println("我是  RealSubject 的show 方法！！！");

	}
	@ProxyTag
	public void showSelf(){
		
		System.out.println("我是  RealSubject 的showSelf 方法！！！");
		
	}

}

```

#### MethodInterceptor 拦截类
```
public class SubjectCglibProxy implements MethodInterceptor {
	
	@Override
	public Object intercept(Object proxy, Method method, Object[] arg2, MethodProxy arg3) throws Throwable {
        Method[] methods = proxy.getClass().getMethods();
        Method[] declaredMethods = proxy.getClass().getDeclaredMethods();

        Method targetMethod = AopUtils.getMostSpecificMethod(method, proxy.getClass());
        // TODO Auto-generated method stub
        Annotation[] annotations = method.getDeclaredAnnotations();
		System.out.println("++++++before " + arg3.getSuperName() + "++++++");
        System.out.println(method.getName());  
        Object o1 = arg3.invokeSuper(proxy, arg2);
        System.out.println("++++++before " + arg3.getSuperName() + "++++++");  
        return o1;  
	}

}

```

#### 启动类
```
public class CglibProxyDemo {

	public static void main(String[] args) {
		// TODO Auto-generated method stub
		System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "/Users/zhuangjiesen/develop/my_github/CustomerAndProvider/JavaProCustomer/src/main");
		SubjectCglibProxy cglibProxy = new SubjectCglibProxy();
		Enhancer enhancer = new Enhancer();
		enhancer.setSuperclass(RealSubject.class);
		enhancer.setCallback(cglibProxy);
		RealSubject realSubject=(RealSubject)enhancer.create();
		realSubject.show();
		realSubject.showSelf();
		
	}

}

```
#### 生成的新类
```

public class RealSubject$$EnhancerByCGLIB$$f0c1daec extends com.java.core.proxy.cglib.RealSubject implements net.sf.cglib.proxy.Factory {
    private boolean CGLIB$BOUND;
    private static final java.lang.ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final net.sf.cglib.proxy.Callback[] CGLIB$STATIC_CALLBACKS;
    private net.sf.cglib.proxy.MethodInterceptor CGLIB$CALLBACK_0;
    private static final java.lang.reflect.Method CGLIB$show$0$Method;
    private static final net.sf.cglib.proxy.MethodProxy CGLIB$show$0$Proxy;
    private static final java.lang.Object[] CGLIB$emptyArgs;
    private static final java.lang.reflect.Method CGLIB$showSelf$1$Method;
    private static final net.sf.cglib.proxy.MethodProxy CGLIB$showSelf$1$Proxy;
    private static final java.lang.reflect.Method CGLIB$finalize$2$Method;
    private static final net.sf.cglib.proxy.MethodProxy CGLIB$finalize$2$Proxy;
    private static final java.lang.reflect.Method CGLIB$equals$3$Method;
    private static final net.sf.cglib.proxy.MethodProxy CGLIB$equals$3$Proxy;
    private static final java.lang.reflect.Method CGLIB$toString$4$Method;
    private static final net.sf.cglib.proxy.MethodProxy CGLIB$toString$4$Proxy;
    private static final java.lang.reflect.Method CGLIB$hashCode$5$Method;
    private static final net.sf.cglib.proxy.MethodProxy CGLIB$hashCode$5$Proxy;
    private static final java.lang.reflect.Method CGLIB$clone$6$Method;
    private static final net.sf.cglib.proxy.MethodProxy CGLIB$clone$6$Proxy;

    static void CGLIB$STATICHOOK1() { /* compiled code */ }

    final void CGLIB$show$0() { /* compiled code */ }

    public final void show() { /* compiled code */ }

    final void CGLIB$showSelf$1() { /* compiled code */ }

    public final void showSelf() { /* compiled code */ }

    final void CGLIB$finalize$2() throws java.lang.Throwable { /* compiled code */ }

    protected final void finalize() throws java.lang.Throwable { /* compiled code */ }

    final boolean CGLIB$equals$3(java.lang.Object o) { /* compiled code */ }

    public final boolean equals(java.lang.Object o) { /* compiled code */ }

    final java.lang.String CGLIB$toString$4() { /* compiled code */ }

    public final java.lang.String toString() { /* compiled code */ }

    final int CGLIB$hashCode$5() { /* compiled code */ }

    public final int hashCode() { /* compiled code */ }

    final java.lang.Object CGLIB$clone$6() throws java.lang.CloneNotSupportedException { /* compiled code */ }

    protected final java.lang.Object clone() throws java.lang.CloneNotSupportedException { /* compiled code */ }

    public static net.sf.cglib.proxy.MethodProxy CGLIB$findMethodProxy(net.sf.cglib.core.Signature signature) { /* compiled code */ }

    public RealSubject$$EnhancerByCGLIB$$f0c1daec() { /* compiled code */ }

    public static void CGLIB$SET_THREAD_CALLBACKS(net.sf.cglib.proxy.Callback[] callbacks) { /* compiled code */ }

    public static void CGLIB$SET_STATIC_CALLBACKS(net.sf.cglib.proxy.Callback[] callbacks) { /* compiled code */ }

    private static final void CGLIB$BIND_CALLBACKS(java.lang.Object o) { /* compiled code */ }

    public java.lang.Object newInstance(net.sf.cglib.proxy.Callback[] callbacks) { /* compiled code */ }

    public java.lang.Object newInstance(net.sf.cglib.proxy.Callback callback) { /* compiled code */ }

    public java.lang.Object newInstance(java.lang.Class[] classes, java.lang.Object[] objects, net.sf.cglib.proxy.Callback[] callbacks) { /* compiled code */ }

    public net.sf.cglib.proxy.Callback getCallback(int i) { /* compiled code */ }

    public void setCallback(int i, net.sf.cglib.proxy.Callback callback) { /* compiled code */ }

    public net.sf.cglib.proxy.Callback[] getCallbacks() { /* compiled code */ }

    public void setCallbacks(net.sf.cglib.proxy.Callback[] callbacks) { /* compiled code */ }
}
```
### 小结

```
    1. cglib代理生成的新类也是不会带注解的
```


### 那为什么cglib生成的代理类是拿得到注解并进行aop拦截的呢

#### 原因
```

在 AopUtils.getMostSpecificMethod(method, targetClass) 方法中
1.jdk代理的targetClass 是 Proxy的子类，获取到的method是新类的method ，新类的方法不会带注解
2.cglib 代理后，但是它的targetClass 还是父类 (比如 RealSubject.java) ，获取到的则是 RealSubject.java 的方法而不是新类的方法，所以拿得到注解


```


### 解决
```
可以通过实现BeanPostProcessor 接口，在方法获取到对应注解的method ，用 map缓存
```

#### 例子
```
@Aspect
@Component
public class MultipleDataSourceAspectAdvice implements Ordered, BeanPostProcessor {


    /** 容量刚好到时再扩容 - 因为是在spring加载后已经不会put了，但是怕有些dao的bean设置成lazy-init **/
    private final Map<Method, DataSourceSelect> daoDataSourceSelectMap = new ConcurrentHashMap<>(256);
    //mybatis的dao方法上带注解aop识别不了
    @Pointcut("execution(* xx包名xx.xxx..*.*(..))")
    public void daoPackage() {
    }

    /**
     * @param pjp
     * @throws Throwable
     * @Description: 在加了DataSourceSelect注解的方法前设置数据源
     */
    @Around(value = "(daoPackage() && target(obj)")
    public Object changeDataSource(ProceedingJoinPoint pjp, Object obj) throws Throwable {
        Signature signature = pjp.getSignature();
        MethodSignature methodSignature = (MethodSignature) signature;
        Method method = methodSignature.getMethod();
        String before = MultipleDataSource.getDataSourceKey();
        DataSourceSelect dataSourceSelect = null;
        //dao的map里拿不到
        if (null == (dataSourceSelect = daoDataSourceSelectMap.get(method))) {
            //获取方法级别的annotation
            Method targetMethod = getMethod(obj.getClass(), method.getName(), method.getParameterTypes());
            dataSourceSelect = targetMethod.getAnnotation(DataSourceSelect.class);
        }
        if (null == dataSourceSelect) {
            dataSourceSelect = obj.getClass().getAnnotation(DataSourceSelect.class);
        }
        if (dataSourceSelect != null) {
            MultipleDataSource.setDataSourceKey(dataSourceSelect.value().key());
        }
        try {
            return pjp.proceed();
        } finally {
            MultipleDataSource.setDataSourceKey(before);
        }
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        //如果是mybatis封装的类
        if (bean instanceof MapperFactoryBean) {
            //mybatis的加载
            try {
                Field mapperInterfaceField = MapperFactoryBean.class.getDeclaredField("mapperInterface");
                mapperInterfaceField.setAccessible(true);
                //获取Mybatis代理的接口 - 遍历方法拿到带切换数据源的方法 - 塞到map中
                Class mapperInterfaceClazz = (Class) mapperInterfaceField.get(bean);
                Method[] daoMethods = mapperInterfaceClazz.getDeclaredMethods();
                for (Method daoMethod : daoMethods) {
                    DataSourceSelect dataSourceSelect = daoMethod.getDeclaredAnnotation(DataSourceSelect.class);
                    if (dataSourceSelect != null) {
                        daoDataSourceSelectMap.put(daoMethod, dataSourceSelect);
                    }
                }
            } catch (NoSuchFieldException noField) {
                LogConstant.runLogger.error(noField.getMessage(), noField);
            } catch (IllegalAccessException e) {
                LogConstant.runLogger.error(e.getMessage(), e);
            }
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}


```






