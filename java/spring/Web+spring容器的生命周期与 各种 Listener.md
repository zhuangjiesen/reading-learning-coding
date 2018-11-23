#Web+spring容器的生命周期与 各种 Listener

## web容器中的 Listener 配置 
### Listener 种类与配置
#### 监听器Listener就是在application,session,request三个对象创建、销毁或者往其中添加修改删除属性时自动执行代码的功能组件。
### Listener是Servlet的监听器，可以监听客户端的请求，服务端的操作等。
#### ServletContextListener 
##### 主要是基于对 ServletContext  初始化与销毁的监听
#### 配置
##### 在 web.xml 中
```xml
    
    <listener>
    	<listener-class>com.java.listener.web.AppServletContextListener</listener-class>
    </listener>
```
##### AppServletContextListener 类
```Java
package com.java.listener.web;
import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;
public class AppServletContextListener implements ServletContextListener {

	@Override
	public void contextDestroyed(ServletContextEvent event) {
		// TODO serletContext销毁		
	}

	@Override
	public void contextInitialized(ServletContextEvent event) {
		// TODO serletContext创建		
  }
}
```
#### ServletRequestListener 
##### 主要是基于对 ServletRequest  初始化与销毁的监听 即 一个request 生命周期 当你访问服务器时生成一个request 访问成功后 request即被销毁
#### 配置
##### 在 web.xml 中
```xml
    
     <listener>
    	<listener-class>com.java.listener.web.AppServletRequestListener</listener-class>
    </listener>
```
##### AppServletRequestListener 类
```Java
package com.java.listener.web;

import javax.servlet.ServletRequestEvent;
import javax.servlet.ServletRequestListener;

public class AppServletRequestListener implements ServletRequestListener {

	@Override
	public void requestDestroyed(ServletRequestEvent request) {
		// TODO request销毁事件
 }
	@Override
	public void requestInitialized(ServletRequestEvent request) {
		// TODO request初始化事件
  }
}
```

#### HttpSessionListener 
##### 主要是基于对 HttpSession  初始化与销毁的监听 即 一个 HttpSession 生命周期 
#### 配置
##### 在 web.xml 中
```xml
    
    <listener>
    	<listener-class>com.java.listener.web.AppHttpSessionListener</listener-class>
    </listener>
```
##### AppHttpSessionListener 类
```Java
package com.java.listener.web;

import javax.servlet.http.HttpSessionEvent;
import javax.servlet.http.HttpSessionListener;

public class AppHttpSessionListener implements HttpSessionListener {

	@Override
	public void sessionCreated(HttpSessionEvent session) {
		// TODO session创建	
	}
	@Override
	public void sessionDestroyed(HttpSessionEvent session) {
		// TODO session销毁	
	}
}
```


## spring 容器的生命周期
### web 容器与 spring 容器类似，都是存放java 对象实例的容器 都是有一个 上下文 管理各种对象实例（bean）
#### 监听容器以及容器中的 bean 的一系列操作 的接口有
##### ApplicationContextAware
##### BeanFactoryAware
##### BeanNameAware
##### InitializingBean
##### DiposableBean
##### BeanFactoryPostProcessor
##### BeanPostProcessor
##### InstantiationAwareBeanPostProcessorAdapter

当我们创建类并实现这些接口，就会调用这些接口的方法，经过测试这些接口bean的调用顺序并没有绝对关系（有部分有关系），谁配在前面谁先加载并执行实现方法
#### ApplicationContextAware 类
```Java 
public class SpringApplicationContextAware implements ApplicationContextAware {

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		// TODO ApplicationContextAware接口方法执行 	setApplicationContext
		//通常这个方法用来获取	applicationContext 类以便后续能通过	applicationContext 获取bean对象 等操作	
	}

}
```
#### BeanFactoryAware 类
```Java 
public class AppBeanFactoryAware implements BeanFactoryAware {

	@Override
	public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
		// TODO Auto-generated method stub
		//可以获取 	beanFactory 对象 进行操作		
	}

}
```

#### BeanNameAware 类
```java
public class AppBeanNameAware implements BeanNameAware{

	@Override
	public void setBeanName(String name) {
		// TODO Auto-generated method stub
	}
}
```
#### InitializingBean 类
```java
public class AppInitializingBean implements InitializingBean {

	@Override
	public void afterPropertiesSet() throws Exception {
		// TODO Auto-generated method stub
	}

}
```


#### DiposableBean 类
```java
public class AppDiposableBean implements DisposableBean {

	@Override
	public void destroy() throws Exception {
		// TODO Auto-generated method stub
	}

}


```

#### BeanFactoryPostProcessor 类
```java
public class AppBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		// TODO Auto-generated method stub
		//同样可以获取beanFactory
	
	}

}


```

#### BeanPostProcessor 类
```java
public class AppBeanPostProcessor implements BeanPostProcessor {

	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		// TODO 用于bean被初始化前	
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		// TODO 用于bean被初始化后		
		return bean;
	}

}


```

#### InstantiationAwareBeanPostProcessorAdapter 类
```java

public class AppInstantiationAwareBeanPostProcessorAdapter extends InstantiationAwareBeanPostProcessorAdapter {

	@Override
	public Class<?> predictBeanType(Class<?> beanClass, String beanName) {
		// TODO Auto-generated method stub
		return super.predictBeanType(beanClass, beanName);
	}

	@Override
	public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) throws BeansException {
		// TODO Auto-generated method stub
		return super.determineCandidateConstructors(beanClass, beanName);
	}

	@Override
	public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
		// TODO Auto-generated method stub
		return super.getEarlyBeanReference(bean, beanName);
	}

	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		// TODO bean对象实例化前调用		
		
		return super.postProcessBeforeInstantiation(beanClass, beanName);
	}

	@Override
	public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
		// TODO bean对象实例化后调用
		
		return super.postProcessAfterInstantiation(bean, beanName);
	}

	@Override
	public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean,
			String beanName) throws BeansException {
		// TODO 设置bean的属性时调用		
		return super.postProcessPropertyValues(pvs, pds, bean, beanName);
	}

	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		// TODO bean对象初始化时调用		
		return super.postProcessBeforeInitialization(bean, beanName);
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		// TODO bean对象初始化后调用
		
		return super.postProcessAfterInitialization(bean, beanName);
	}
	
}

```

#### 由于都是对于 bean 的操作 ,BeanPostProcessor 类 与 InstantiationAwareBeanPostProcessorAdapter 的调用上有交集;

#### 调用流程：

##### AppInstantiationAwareBeanPostProcessorAdapter 的 postProcessBeforeInstantiation
##### AppInstantiationAwareBeanPostProcessorAdapter 的 postProcessAfterInstantiation 方法
##### AppInstantiationAwareBeanPostProcessorAdapter 的 postProcessPropertyValues  方法
##### AppInstantiationAwareBeanPostProcessorAdapter 的 postProcessBeforeInitialization  方法
##### AppBeanPostProcessor 的 postProcessBeforeInitialization 方法
##### AppInstantiationAwareBeanPostProcessorAdapter 的 postProcessAfterInitialization 方法
##### AppBeanPostProcessor 的 postProcessAfterInitialization 方法

#### spring 的 ContextLoaderListener 类
##### 用于对 ServletContext 对象的监听 ，ServletContext对象 对于web容器是单例的，所有用户对应同一个，通过 getServletContext().setAttribute 方法可以设置全局应用的属性
```Java


public class AppContextLoaderListener extends ContextLoaderListener {

	@Override
	public void closeWebApplicationContext(ServletContext servletContext) {
		// TODO Auto-generated method stub
		super.closeWebApplicationContext(servletContext);
	}



	@Override
	public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
		// TODO 初始化	ServletContext 上下文 这个方法在contextInitialized 方法之前调用		
		return super.initWebApplicationContext(servletContext);
	}



	@Override
	public void contextInitialized(ServletContextEvent event) {
		// TODO 上下文初始化成功		
		super.contextInitialized(event);		
	}
	
	
	
	

	@Override
	public void contextDestroyed(ServletContextEvent event) {
		// TODO Auto-generated method stub
		super.contextDestroyed(event);
	}

	
	
}

```
#### 以上就是spring + web 容器的各种监听类的配置，可以用于应用初始化时，全局变量与配置，以及应用启动时需要做的一下逻辑处理 等
##### 最后附上配置文件总代码
##### web.xml
```xml
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <display-name>Archetype Created Web Application</display-name>
  
    

    <listener>
    	<listener-class>com.java.listener.web.AppServletContextListener</listener-class>
    </listener>
    <listener>
    	<listener-class>com.java.listener.web.AppServletRequestListener</listener-class>
    </listener>
    <listener>
    	<listener-class>com.java.listener.web.AppHttpSessionListener</listener-class>
    </listener>
    
    <listener>
    <listener-class>com.java.listener.spring.AppContextLoaderListener</listener-class>
	</listener>



      <servlet>
        <servlet-name>springMVC</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/applicationContext.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>springMVC</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
    
    <servlet>
    	<servlet-name>InitServlet</servlet-name>
    	<display-name>InitServlet</display-name>
    	<description></description>
    	<servlet-class>com.java.servlet.InitServlet</servlet-class>
    	<load-on-startup>1</load-on-startup>
    </servlet>
    
    
</web-app>


```
##### spring bean配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-3.0.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc-3.0.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx-3.0.xsd">

 <bean id="springApplicationContextAware" class="com.java.listener.spring.SpringApplicationContextAware"></bean>
 
 <bean id="appBeanFactoryAware" class="com.java.listener.spring.AppBeanFactoryAware"></bean>
 <bean id="appBeanNameAware" class="com.java.listener.spring.AppBeanNameAware"></bean>
 <bean id="appInitializingBean" class="com.java.listener.spring.AppInitializingBean"></bean>
 <bean id="appDiposableBean" class="com.java.listener.spring.AppDiposableBean"></bean>
 <bean id="appBeanFactoryPostProcessor" class="com.java.listener.spring.AppBeanFactoryPostProcessor"></bean>
 <bean id="appInstantiationAwareBeanPostProcessorAdapter" class="com.java.listener.spring.AppInstantiationAwareBeanPostProcessorAdapter"></bean>
 <bean id="appBeanPostProcessor" class="com.java.listener.spring.AppBeanPostProcessor"></bean>

</beans>
```


