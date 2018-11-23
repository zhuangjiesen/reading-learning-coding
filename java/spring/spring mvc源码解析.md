# spring mvc源码解析

### 简述：
#### 阅读源码的过程很枯燥，我的方法是通过打断点的形式，一步一步的分析出整个框架的流程，多处地方会使用模板方法设计模式，就有可能是抽象类反转控制实现类，所以
#### 有时候需要耐心找出实现类的实现方法

#### 阅读源码后的好处，可以对框架进一步熟悉，可以能用更有效率的办法解决业务上的问题，以及对框架拓展（多viewResolvers配置）

#### 源码很多设计模式，比看一头扎进设计模式这种书更有好处，因为你可以看到设计模式的具体使用场景，很有帮助



### DispatcherServlet

#### 结构
##### DispatcherServlet extends FrameworkServlet 
##### FrameworkServlet extends HttpServletBean
##### HttpServletBean extends HttpServlet 


两个流程
1.初始化
2.请求处理


可以看出 最终就是间接继承了 HttpServlet 
所以 通过HttpServlet类的生命周期以及方法 可以知道
初始化肯定入口是 init() 方法
处理请求即 doServcie() 方法



初始化
1.初始化 spring 容器
在 web.xml 中配置的 DispatcherServlet 
```XML
<init-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/applicationContext.xml</param-value>
</init-param>
```
中的 contextConfigLocation 属性是在 FrameworkServlet 类中的

调用流程
+ HttpServletBean init() 
+ HttpServletBean init() 中调用了子类 FrameworkServlet 的 initServletBean() 方法
+ FrameworkServlet 的 initServletBean() 中调用了 initWebApplicationContext() 初始化webApplicationContext（FrameworkServlet的全局变量） spring上下文（容器）
+ FrameworkServlet 的 initServletBean() 中调用了 initFrameworkServlet() 方法为空

以上是spring 容器的启动流程

spring MVC框架初始化调用

initStrategies()方法

DispatcherServlet.properties 文件中定义 每个处理类的默认实现类
/org/springframework/web/servlet/DispatcherServlet.properties

//  初始化处理  multipart/form-data 类型的 MultipartResolver类
initMultipartResolver(context);
initLocaleResolver(context);
initThemeResolver(context);
initHandlerMappings(context);
initHandlerAdapters(context);
initHandlerExceptionResolvers(context);
initRequestToViewNameTranslator(context);
initViewResolvers(context);
initFlashMapManager(context);



initHandlerMappings(context) 方法
*** 此部分源码初始化逻辑有点难，但是结果清晰。
原理：通过
1. 将 url字符串 与 controller中的method 加入到 map 集合中 
2. 将 </mvc:interceptors> 拦截器配置的拦截器  加载到 集合中(HandlerExecutionChain 中有拦截器链 HandlerInterceptor[] interceptors )
以至于最后能够通过url 匹配快速定位获取到 HandlerExecutionChain 类，
结果： 最后初始化 HandlerMappings 用来获取 HandlerExecutionChain  

断点进入
initHandlerMappings
```Java
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




initHandlerAdapters(context);
初始化
RequestMappingHandlerAdapter
HttpRequestHandlerAdapter
SimpleControllerHandlerAdapter




都继承 AbstractHandlerMethodAdapter 类，其中用模板方法模式，控制子类实现 两个方法 supportsInternal() 与 handleInternal() 方法
AbstractHandlerMethodAdapter 通过 supports() 与 handle() 调用 supportsInternal() 与 handleInternal()
部分代码块：
```java
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


AbstractHandlerMethodAdapter 实现 HandlerAdapter 接口 
```Java
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




RequestMappingHandlerAdapter 类
初始化后调用 afterPropertiesSet() 方法(因为实现了InitializingBean接口具体看spring 生命周期) 
代码块：
```java

	@Override
	public void afterPropertiesSet() {
		// Do this first, it may add ResponseBody advice beans
		initControllerAdviceCache();

		if (this.argumentResolvers == null) {
			//暂时还找不到调用这个处理的列表
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.initBinderArgumentResolvers == null) {
			//暂时还找不到调用这个处理的列表
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



initHandlerExceptionResolvers 方法，初始化异常解决处理器

简单异常处理类
SimpleMappingExceptionResolver
继承 AbstractHandlerExceptionResolver 
实现 doResolveException 
AbstractHandlerExceptionResolver 类中通过模板方法模式，resolveException() 方法中 调用子类实现的 doResolveException 处理请求

通过在spring 中配置继承 AbstractHandlerExceptionResolver 的bean 可以注册自定义异常处理类
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



initViewResolvers(context) 初始化视图处理器

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







2.初始化 spring MVC 处理请求使用到的各种 resolvers 组件
initStrategies 方法












处理请求

HttpServlet中 用来处理请求的是 doService方法

doService方法中 对 request 进行预处理 ，之后调用 doDispatch(request, response) 方法进入请求处理流程
代码：
```Java



	/**
	 * Exposes the DispatcherServlet-specific request attributes and delegates to {@link #doDispatch}
	 * for the actual dispatching.
	 */
	@Override
	protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		if (logger.isDebugEnabled()) {
			String resumed = WebAsyncUtils.getAsyncManager(request).hasConcurrentResult() ? " resumed" : "";
			logger.debug("DispatcherServlet with name '" + getServletName() + "'" + resumed +
					" processing " + request.getMethod() + " request for [" + getRequestUri(request) + "]");
		}

		// Keep a snapshot of the request attributes in case of an include,
		// to be able to restore the original attributes after the include.
		Map<String, Object> attributesSnapshot = null;
		if (WebUtils.isIncludeRequest(request)) {
			attributesSnapshot = new HashMap<String, Object>();
			Enumeration<?> attrNames = request.getAttributeNames();
			while (attrNames.hasMoreElements()) {
				String attrName = (String) attrNames.nextElement();
				if (this.cleanupAfterInclude || attrName.startsWith("org.springframework.web.servlet")) {
					attributesSnapshot.put(attrName, request.getAttribute(attrName));
				}
			}
		}

		// Make framework objects available to handlers and view objects.
		//将 spring容器的 ApplicationContext 放到request 中 翻译： 使 请求处理 handlers 和 视图view 能使用spring 容器中配置的bean 
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());


		//mark ...  等待有缘人讲解...
		FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
		if (inputFlashMap != null) {
			request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
		}
		request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
		request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

		try {

			//springMVC处理请求的核心
			doDispatch(request, response);
		}
		finally {
			if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
				// Restore the original attribute snapshot, in case of an include.
				if (attributesSnapshot != null) {
					restoreAttributesAfterInclude(request, attributesSnapshot);
				}
			}
		}
	}

```

doDispatch(request, response);
代码：
```Java

	/**
	 * Process the actual dispatching to the handler.
	 * <p>The handler will be obtained by applying the servlet's HandlerMappings in order.
	 * The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
	 * to find the first that supports the handler class.
	 * <p>All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
	 * themselves to decide which methods are acceptable.
	 * @param request current HTTP request
	 * @param response current HTTP response
	 * @throws Exception in case of any kind of processing failure
	 */
	 /*
		大致翻译： 
		所有http请求都是由这个方法处理，取决于 HandlerAdapters 或 handlers 决定哪个方法处理
		HandlerAdapter 将会通过查找 servlect 初始化后（安装后）的 HandlerAdapters 列表，找到第一个满足的 HandlerAdapter
		所有请求处理 handler 将会通过 HandlerMappings 查找到


	 */
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;


		//mark 
		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			// 视图类
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				//检查request类型 判断 http请求是否 multipart/form-data 类型
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.  
				// 获取 匹配当前请求的 HandlerExecutionChain类 的 handler
				mappedHandler = getHandler(processedRequest);
				// mappedHandler.getHandler() 返回值是个Object 类型  通过打断点证实是个 HandlerMethod 类 
				/*

					public class HandlerMethod {

						/** Logger that is available to subclasses */
						protected final Log logger = LogFactory.getLog(HandlerMethod.class);
						// controller 的类
						private final Object bean;
						// spring 的beanFactory
						private final BeanFactory beanFactory;
						//匹配url（@RequestMapping） 处理的method 方法
						private final Method method;
						//mark.... 
						private final Method bridgedMethod;
						//参数
						private final MethodParameter[] parameters;
					}
					

				*/
				if (mappedHandler == null || mappedHandler.getHandler() == null) {
					//未获取到请求即返回
					noHandlerFound(processedRequest, response);
					return;
				}


				// Determine handler adapter for the current request.
				//(适配器模式)通过遍历 handlerAdapters 来查找 supports 这个handler 的 handlerAdapter 类， 返回第一个匹配，查找不到即抛异常
				//HandlerAdapter 的 supports 方法源码在 AbstractHandlerMethodAdapter(模板方法模式)控制子类的 supportsInternal 方法
				/*
					在 RequestMappingHandlerAdapter 中 supportsInternal 方法中 写死了return true 
					此处 supportsInternal 方法可以用来 自己创建 HandlerAdapter 与 自己定义 HandlerMethod 处理对象的格式

					判断 handler 是否是 HandlerMethod 类型 此处也可以知道 handler 的类型 是 HandlerMethod
						@Override
						public final boolean supports(Object handler) {
							return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
						}
				*/
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());


				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (logger.isDebugEnabled()) {
						logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
					}
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}


				// applyPreHandle 方法中调用了 HandlerExecutionChain 类中的 HandlerInterceptor 拦截器对请求进行拦截处理（preHandle方法）
				/*
					HandlerInterceptor 可以通过配置    <mvc:interceptors> 拦截器 </mvc:interceptors>  对请求进行拦截
				*/
				if (!mappedHandler.applyPreHandle 方法中 调用了 (processedRequest, response)) {
					return;
				}


				
				// Actually invoke the handler.
				// 翻译： 真正调用请求方法主体 即controller 中处理请求的那个 method 也是 HandlerMethod 中的method 属性
				// handler 方法中还有很多逻辑
				// 返回视图
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(request, mv);

				// 调用 HandlerInterceptor 拦截器的 postHandle 方法
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}

			//返回视图，浏览器上展示效果
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Error err) {
			triggerAfterCompletionWithError(processedRequest, response, mappedHandler, err);
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}



```


#### 看完代码，分析出四个关键点（拦截器调用的忽略了）
+  getHandler(processedRequest) 方法 获取类 HandlerExecutionChain 我理解成存放这个 url 请求的各种信息的类（bean , method ,参数 , 拦截器 ,beanFactory 等等）
+  getHandlerAdapter(mappedHandler.getHandler()) 方法 获取请求处理类 handlerAdapter 
+  ha.handle(processedRequest, response, mappedHandler.getHandler()) 方法返回 ModelAndView 视图
+  processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException) 结果处理方法以及异常处理






getHandler(processedRequest) 方法 返回HandlerExecutionChain 对象

RequestMappingInfoHandlerMapping 类进行处理
结构：
继承 AbstractHandlerMethodMapping 类
继承 AbstractHandlerMapping 类
继承 AbstractHandlerMapping 类
实现 HandlerMapping 接口实现 HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception; 方法
AbstractHandlerMapping 调用 getHandler() 方法，通过模板方法模式，控制子类实现方法来处理；
```Java
	/**
	 * Look up a handler for the given request, falling back to the default
	 * handler if no specific one is found.
	 * @param request current HTTP request
	 * @return the corresponding handler instance, or the default handler
	 * @see #getHandlerInternal
	 */
	@Override
	public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		// 通过 request 获取 handler （HandlerMethod 类型 ）
		/*
			getHandlerInternal 通过获取request 的url 请求字符串 ，去初始化时创建的urlMap 中取 Controller主体Method 封装成 handlder( HandlerMethod 类 )
		*/
		Object handler = getHandlerInternal(request);


		if (handler == null) {
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}
		// Bean name or resolved handler?
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = getApplicationContext().getBean(handlerName);
		}
		/*
			将 handlder( HandlerMethod 类 ) 与 request 方法 并添加拦截器数组封装成 HandlerExecutionChain 的对象返回
				代码块：
				protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
					HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
							(HandlerExecutionChain) handler : new HandlerExecutionChain(handler));
					chain.addInterceptors(getAdaptedInterceptors());

					String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
					for (MappedInterceptor mappedInterceptor : this.mappedInterceptors) {
						if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
							chain.addInterceptor(mappedInterceptor.getInterceptor());
						}
					}

					return chain;
				}

		*/
		return getHandlerExecutionChain(handler, request);
	}

```







 getHandlerAdapter(mappedHandler.getHandler()) 方法，返回 HandlerAdapter 对象

 RequestMappingHandlerAdapter 类进行处理
 结构 
 继承 AbstractHandlerMethodAdapter 类
 实现 HandlerAdapter 接口 	 
 实现三个方法： 
	boolean supports(Object handler);
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
	long getLastModified(HttpServletRequest request, Object handler);

AbstractHandlerMethodAdapter 类中
1. handle() 方法，控制继承子类 handleInternal() 方法 （还是模板方法模式）
2. supports() 方法， 判断了 handler的类型 且 控制子类的 supportsInternal() 方法
	代码：
	```Java
	public final boolean supports(Object handler) {
		return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
	}
	```

RequestMappingHandlerAdapter 子类中：
supportsInternal 直接返回true ,即只判断handler 类型是否为 HandlerMethod；

```Java
	/**
	 * Return the HandlerAdapter for this handler object.
	 * @param handler the handler object to find an adapter for
	 * @throws ServletException if no HandlerAdapter can be found for the handler. This is a fatal error.
	 */
	protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		for (HandlerAdapter ha : this.handlerAdapters) {
			if (logger.isTraceEnabled()) {
				logger.trace("Testing handler adapter [" + ha + "]");
			}

			/*
				此处判断 spring MVC加载的 HandlerAdapter 列表，是否符合处理这个 handler 能力 （是否支持）

			*/
			if (ha.supports(handler)) {
				return ha;
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
```



ha.handle(processedRequest, response, mappedHandler.getHandler()) 方法，最终返回视图 ModelAndView 对象
即 AbstractHandlerMethodAdapter 类中的handle() 方法

RequestMappingHandlerAdapter 子类中：
handleInternal() 方法中，调用了 invokeHandleMethod() 方法；
invokeHandleMethod() 方法中，
1. 调用 requestMappingMethod.invokeAndHandle(webRequest, mavContainer);
	invokeAndHandle() 方法中 ：
	1.Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs); 调用handler 处理请求获取到返回值 如果是String类型返回值，返回的是页面名称，responseBody的map 结果，返回的是map 对象；
	2.调用 returnValueHandlers 的 returnValueHandlers.handleReturnValue() 方法  (也就是 messageConverters 出现的地方) 对请求返回值进行处理；
2. 调用 getModelAndView() 方法
   将处理结果封装 ModelAndView 对象
   1.如果是responseBody 返回 null 的 ModelAndView对象
   2.如果是String 返回页面视图类




注意：
如果是ResponseBody ，ModelAndView对象是null，不会进入ViewResolver进行处理  则返回的是response对象，response对象outputStream就存储着map（json）返回值；


整个dispatcherServlet 处理代码就是这样。



