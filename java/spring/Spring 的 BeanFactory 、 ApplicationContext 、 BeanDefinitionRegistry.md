# Spring 的 BeanFactory 、 ApplicationContext 、 BeanDefinitionRegistry
### BeanFactory 只是一个单一功能的接口
### BeanFactory.java 作用:

```
基本的使用spring 的bean 容器接口(其实就是存有各种(类型 -> )bean的map)
这个接口提供了接口的基本功能(方法)
更多的实现在于 -> ListableBeanFactory 和 ConfigurableBeanFactory (DefaultListableBeanFactory) 提供明确的需求的实现方法

当对象持有 bean的定义的列表，需要实现BeanFactory 接口，bean的定义都有一个独一无二标识(id)，基于bean的定义，beanFactory 将会返回prototype 和 singleton对象，

BeanFactory 加载完成后，所有的bean都会存在于bean容器中，已经无法改变bean的定义，只能获取bean(同时可能初始化)

```


### BeanDefinitionRegistry 
```
用法 ： BeanDefinitionRegistryPostProcessor 可以通过回调，让spring把 BeanDefinitionRegistry传进来
作用：在beanFactory的bean列表中动态添加 beanDefinition

```



### ApplicationContext 是一个聚合接口实现了很多个接口类，其中就有 BeanFactory 的子类
### ApplicationContext.java 上的注释：
```
/**
 * Central interface to provide configuration for an application.
 * This is read-only while the application is running, but may be
 * reloaded if the implementation supports this.
 *
 * <p>An ApplicationContext provides:
 * <ul>
 * <li>Bean factory methods for accessing application components.
 * Inherited from {@link org.springframework.beans.factory.ListableBeanFactory}.
 * <li>The ability to load file resources in a generic fashion.
 * Inherited from the {@link org.springframework.core.io.ResourceLoader} interface.
 * <li>The ability to publish events to registered listeners.
 * Inherited from the {@link ApplicationEventPublisher} interface.
 * <li>The ability to resolve messages, supporting internationalization.
 * Inherited from the {@link MessageSource} interface.
 * <li>Inheritance from a parent context. Definitions in a descendant context
 * will always take priority. This means, for example, that a single parent
 * context can be used by an entire web application, while each servlet has
 * its own child context that is independent of that of any other servlet.
 * </ul>
 *

```

### 翻译：
```
应用程序中提供主要配置的主要接口。
这个接口，在应用程序运行时只能提供读操作，但是可能被实现方法中被重载(也就是自己搞逻辑)
ApplicationContext 提供了：
1.BeanFactory的功能 -> org.springframework.beans.factory.ListableBeanFactory
2.加载资源文件 -> org.springframework.core.io.ResourceLoader
3.注册和发布spring的事件 -> ApplicationEventPublisher
4.解决MessageSource -> MessageSource
5.继承自父类 ApplicationContext。子类ApplicationContext会优先执行，而且相对于别的功能独立运行




```
### 个人理解
```
ApplicationContext (上下文)，等于spring的应用程序入口，子类会初始化bean(AbstractApplicationContext.refresh()中初始化spring容器 )

BeanFactory 是
```


