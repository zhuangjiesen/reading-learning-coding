# aop的实现流程以及入口 , 具体怎么样对bean进行切面 ，aop热交换

### 入口在spring aop 包下 META-INF Resource Bundle 'spring'

```
ConfigBeanDefinitionParser 类中 configureAutoProxyCreator() 方法中初始化并向容器中注入了一个bean 
aop处理的核心 bean 
beanName -> org.springframework.aop.config.internalAutoProxyCreator

class -> org.springframework.aop.framework.autoproxy.AspectJAwareAdvisorAutoProxyCreator  -> 继承 AbstractAutoProxyCreator (也是aop 切面代理的处理类)

处理核心是实现 SmartInstantiationAwareBeanPostProcessor 接口 (BeanPostProcessor 接口)
通过在 postProcessAfterInitialization() 用动态代理( ProxyFactory 类代理生成新的动态类) return 新的动态代理结果

```


**疑问1：**

```
动态代理也是需要实例化对象 而postProcessAfterInitialization() 的调用时，bean已经实例化了，怎么关联
答: aop动态代理的类将实例化的bean对象封装成 targetSource 类，传到代理类中，methodProxy.invoke时，调用的还是bean对象本身，等于加了一层封装 (具体在 AopProxy 实现类 ：JdkDynamicAopProxy、Cglib2AopProxy)

```

### aop 中如果是配置了自动注入

```
<aop:aspectj-autoproxy proxy-target-class="true" />

```
#### aop源码中判断 目标类不是接口类型的话也是默认cglib 方式进行切面代理



#### 目标被代理类 TargetSource (类似 BeanDefinition 存放目标类信息)

```
advice -> DefaultBeanFactoryPointcutAdvisor
pointcut -> AspectJExpressionPointcut
aspect -> AspectComponentDefinition
```

