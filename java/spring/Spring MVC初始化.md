# Spring MVC初始化

### 经过上一篇分析请求的处理，可以得出spring MVC的请求处理过程中需要的组件，从而才知道初始化的时候需要哪些；

#### 因为是基于spring 的，所以需要加载spring 容器，并且组件都由spring容器进行管理

**所以初始化分为两部分：**
* **spring容器；**
* **spring MVC 即DispatcherServlet类中的属性；**

##### 调用流程
* HttpServletBean init() 
* HttpServletBean init() 中调用了子类 FrameworkServlet 的 initServletBean() 方法
* FrameworkServlet 的 initServletBean() 中调用了 initWebApplicationContext() 初始化webApplicationContext（FrameworkServlet的全局变量） spring上下文（容器）
* FrameworkServlet 的 initServletBean() 中调用了 initFrameworkServlet() 方法为空
* DispatcherServlet 的 onRefresh() 方法， spring 容器初始化完成，将applicationContext 对象传进 DispatcherServlet 
* DispatcherServlet 的 initStrategies(ApplicationContext context); 方法


##### DispatcherServlet 的 onRefresh() 方法，源码：
    
```
    protected void onRefresh(ApplicationContext context) {
        this.initStrategies(context);
    }
```

##### DispatcherServlet 的 initStrategies(ApplicationContext context); 源码：
```
    protected void initStrategies(ApplicationContext context) {
        this.initMultipartResolver(context);
        this.initLocaleResolver(context);
        this.initThemeResolver(context);
        this.initHandlerMappings(context);
        this.initHandlerAdapters(context);
        this.initHandlerExceptionResolvers(context);
        this.initRequestToViewNameTranslator(context);
        this.initViewResolvers(context);
        this.initFlashMapManager(context);
    }
```
#### 如果熟悉请求处理流程，可以直接根据方法名清楚这些初始化的组件；



#### initHandlerMappings(context) 方法
##### 此部分源码初始化逻辑有点难，但是结果清晰。
原理：
##### 1.将 url字符串 与 controller中的method 加入到 map 集合中 
##### 2. 将 </mvc:interceptors> 拦截器配置的拦截器  加载到 集合中(HandlerExecutionChain 中有拦截器链 HandlerInterceptor[] interceptors )
##### 以至于最后能够通过url 匹配快速定位获取到 HandlerExecutionChain 类，
##### 结果： 最后初始化 HandlerMappings 用来获取 HandlerExecutionChain 

断点进入
**initHandlerMappings** 代码：


```
/**
	 * Initialize the HandlerMappings used by this class.
	 * <p>If no HandlerMapping beans are defined in the BeanFactory for this namespace,
	 * we default to BeanNameUrlHandlerMapping.
	 */
	private void initHandlerMappings(ApplicationContext context) {
		this.handlerMappings = null;

		// 默认 detectAllHandlerMappings =true 则进入判断
		if (this.detectAllHandlerMappings) {
			// Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
			//翻译 ： 查找出所有应用中的HandlerMappings 类，包括祖先
			/*
				matchingBeans 结果 通过打断点 得出两个 HandlerMapping 类 （都是从spring 容器获取，已经实例化）
				1. org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
				2. org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping 
				处理请求时 ，是遍历 handlerMappings 取符合的第一个 HandlerMapping 类
					
				HandlerMapping 初始化时，将所有的 interceptors 与 handler 加载出来
			*/ 
			Map<String, HandlerMapping> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
				// We keep HandlerMappings in sorted order.
				OrderComparator.sort(this.handlerMappings);
			}
		}
		else {
			try {
				HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
				this.handlerMappings = Collections.singletonList(hm);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default HandlerMapping later.
			}
		}

		// Ensure we have at least one HandlerMapping, by registering
		// a default HandlerMapping if no other mappings are found.
		if (this.handlerMappings == null) {
			this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
			if (logger.isDebugEnabled()) {
				logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
			}
		}
	}
```




#### initHandlerAdapters(context);
初始化
* **RequestMappingHandlerAdapter**
* **HttpRequestHandlerAdapter**
* **SimpleControllerHandlerAdapter**

都继承 **AbstractHandlerMethodAdapter** 类，其中用模板方法模式，控制子类实现 两个方法 **supportsInternal()** 与 **handleInternal()** 方法
**AbstractHandlerMethodAdapter** 通过 supports() 与 handle() 调用 supportsInternal() 与 handleInternal()
##### 部分代码块：

```

public abstract class AbstractHandlerMethodAdapter extends WebContentGenerator implements HandlerAdapter, Ordered {

	@Override
	public final boolean supports(Object handler) {
		return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
	}

	@Override
	public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		return handleInternal(request, response, (HandlerMethod) handler);
	}

}
```


**AbstractHandlerMethodAdapter** 实现 **HandlerAdapter** 接口 



```

public interface HandlerAdapter {

	/**
	 用来判断 HandlerExecutionChain 的 handler 属性是否支持，若是支持即当前的 HandlerAdapter实现类就是request的处理类
	 */
	boolean supports(Object handler);

	/*
		处理请求的方法 
	*/
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;

	long getLastModified(HttpServletRequest request, Object handler);

}


```



**RequestMappingHandlerAdapter** 类
初始化后调用 **afterPropertiesSet() **方法(因为实现了InitializingBean接口具体看spring 生命周期) 
代码块：



```


	@Override
	public void afterPropertiesSet() {
		// Do this first, it may add ResponseBody advice beans
		initControllerAdviceCache();

		if (this.argumentResolvers == null) {
			//mark 
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.initBinderArgumentResolvers == null) {
			//mark 
			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.returnValueHandlers == null) {
			/*
				加载默认返回值处理器， 这部分涉及配置 
				<mvc:annotation-driven > 
			    	<mvc:message-converters register-defaults="true">
							<bean id="mappingJacksonHttpMessageConverter"
								class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
								<property name="supportedMediaTypes">
									<list>
										<value>text/html;charset=UTF-8</value>
									</list>
								</property>
							</bean>
			    	</mvc:message-converters>
			    </mvc:annotation-driven>

			    message-converters 组件来处理从controller 返回的结果,比如502错误（需要jackson转map成json格式）、或者进行字符格式转换


			    只处理 responseBody 注解的返回值 RequestResponseBodyMethodProcessor 类处理


			*/
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}



```


#### initHandlerExceptionResolvers 方法，初始化异常解决处理器

简单异常处理类
**SimpleMappingExceptionResolver**
继承 **AbstractHandlerExceptionResolver** 
实现 **doResolveException** 
**AbstractHandlerExceptionResolver** 类中通过模板方法模式，**resolveException()** 方法中 调用子类实现的 **doResolveException** 处理请求

通过在spring 中配置继承 **AbstractHandlerExceptionResolver** 的bean 可以注册自定义异常处理类
```xml 
    <bean id="appHandlerExceptionResolver"   class="com.java.exception.resolvers.AppHandlerExceptionResolver" >
    </bean>
```

AppHandlerExceptionResolver 类
```Java
/*
在这个处理类中可以定制个性化的异常返回视图，以及异常json结果
*/
public class AppHandlerExceptionResolver extends AbstractHandlerExceptionResolver {

	@Override
	protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response, Object handler,
			Exception ex) {
		// TODO Auto-generated method stub
		
		ModelAndView exView=new ModelAndView("exception");
		exView.addObject("exception", ex);
		response.setStatus(500);
		
		
		return exView;
	}

}

```



#### initViewResolvers(context) 初始化视图处理器

springMVC配置的jsp视图解释器 InternalResourceViewResolver 的结构
```xml
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/pages/" />
        <property name="suffix" value=".jsp" />
    </bean>
```
InternalResourceViewResolver 
继承 UrlBasedViewResolver 类
继承 AbstractCachingViewResolver 类
实现 ViewResolver 接口
最终通过 resolveViewName() 方法返回 org.springframework.web.servlet.View 视图对象








