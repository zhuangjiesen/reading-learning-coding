# ioc 容器的实现，一个bean的加载流程 , 参数解析  依赖循环解决  实例化策略




PropertyPlaceholderConfigurer 类用来加载配置文件 ，然后通过el 表达式直接引用：

```
	<!-- ========================================配置数据源========================================= -->
	<!-- 配置数据源，使用的是alibaba的Druid(德鲁伊)数据源 -->
	<!--引入配置属性文件  -->
	<context:property-placeholder location="/resources/jdbc.properties"/>


```


spring启动入口 AbstractApplicationContext 的refresh() 方法

步骤
1. 加载spring 配置到 Resource 以及一些初始环境


2. 初始化 BeanFactory 
obtainFreshBeanFactory()
   初始化 BeanFactory  
   读取spring 配置文件，加载解析xml 到内存 -> 类 AbstractRefreshableApplicationContext 的 refreshBeanFactory() 方法 createBeanFactory() 方法中创建 DefaultListableBeanFactory  接着 - > AbstractXmlApplicationContext 的 loadBeanDefinitions(beanFactory)加载  BeanDefinition ( XmlBeanDefinitionReader 与 BeanDefinitionParserDelegate 、BeanDefinitionRegistry 加载到 DefaultListableBeanFactory 的 beanDefinitionMap 中)


所有bean 都加载完毕 ，但是还未实例化
之后再进行一系列操作



3. 初始化 BeanFactoryPostProcessor  BeanDefinitionRegistryPostProcessor BeanPostProcessor (InstantiationAwareBeanPostProcessor ) 用于spring中生命周期的调用
4. 之后会扫描配置

spring Bean 的初始化使用 instantiationStrategy 策略模式
默认设置成 AbstractAutowireCapableBeanFactory 中的 CglibSubclassingInstantiationStrategy 模式，动态代理创建的 Bean 实例
类实例化或者实例化前都会调用 spring 的一些接口回调

循环依赖的解决，其实就是递归创建属性，直到依赖关系都实例化

