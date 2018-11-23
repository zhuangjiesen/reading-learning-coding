# bean的配置种类

### xml 中

```
 <bean id="testService"  class="com.java.service.TestService"></bean>
 
```

### 基于注解扫描的 

```

    <!-- 启动包扫描功能，以便注册带有@Controller注解的类成为spring的bean -->
     <context:component-scan base-package="*包名">
     	<context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
     </context:component-scan>


```
###  基于实现 FactoryBean 

```
通过在xml 或注解定义的一个实现 FactoryBean 的类，可以指定返回bean的类型是getType() 方法返回值
通过getObject() 获取返回bean的实例对象

```

### BeanDefinitionRegisty (mybatis 的实现) 代码中注册 bean 
#### 事例代码：

```

/*
BeanHelper 中实现了 BeanDefinitionRegistyAware 接口，获取到了 BeanDefinitionRegisty 对象
*/
BeanDefinitionRegistry registry = BeanHelper.beanDefinitionRegistry;

GenericBeanDefinition genericBeanDefinition = new GenericBeanDefinition();
/* 设置bean 的class 类型 这里如果设置成 FactoryBean类型的话，走FactoryBean 接口方法
 
 */
genericBeanDefinition.setBeanClass(MyDragsunTestService.class);

/*
 注册到ioc 容器中
*/
registry.registerBeanDefinition("myTypicalBean" , genericBeanDefinition);


/*
使用这个bean，结果成功
*/
MyDragsunTestService myDragsunTestService= (MyDragsunTestService)applicationContext.getBean("myTypicalBean");
myDragsunTestService.doMyDragsunTestServiceShow();

```

