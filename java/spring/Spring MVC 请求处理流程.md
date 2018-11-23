# Spring MVC 请求处理流程




##### 先从请求处理开始吧，知道请求处理的流程，就能大概清楚，初始化都需要对哪些组件进行实例化。


-------

### DispatcherServlet中的 doService();
代码：

```



	/**
	 * Exposes the DispatcherServlet-specific request attributes and delegates to {@link #doDispatch}
	 * for the actual dispatching.
	 */
	 /*
	 翻译：暴露（个人理解是讲参数取出，或者传入新的参数）请求参数并指派（传递）给 doDispatch() 方法
	 目的： 为了实际调度（处理请求）
	 个人理解： 即对request进行预加工，设置默认属性，然后将 request,response 传入 doDispatch() 方法，对请求进行处理
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
		
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());


		//mark ...  
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

### doDispatch(request, response) 方法

代码：

```

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
			// 异常类
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
				/*
				(适配器模式)通过遍历 handlerAdapters 来查找 supports 这个handler 的 handlerAdapter 类， 返回第一个匹配，查找不到即抛异常
				HandlerAdapter 的 supports 方法源码在 AbstractHandlerMethodAdapter(模板方法模式)控制子类的 supportsInternal 方法
				
					在 RequestMappingHandlerAdapter 中 supportsInternal 方法中 写死了return true 
					此处 supportsInternal 方法可以用来 自己创建 HandlerAdapter 与 自己定义 HandlerMethod 处理对象的格式 

		     判断 handler 是否是 HandlerMethod 类型 
					AbstractHandlerMethodAdapter 类中的 supports() 方法:
						@Override
						public final boolean supports(Object handler) {
							return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
						}
				此处也可以知道 handler 的类型 是 HandlerMethod
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


				/*
				    applyPreHandle 方法中调用了 HandlerExecutionChain 类中的 HandlerInterceptor 拦截器对请求进行拦截处理（preHandle方法）
					HandlerInterceptor 可以通过配置    <mvc:interceptors> 拦截器 </mvc:interceptors>  对请求进行拦截
					HandlerInterceptor 的 preHandle() 方法

					
				*/
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
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

				// 调用 HandlerInterceptor 拦截器的 postHandle 方法mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}

			//返回视图，浏览器上展示效果
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
		
		  /*
        调用拦截器的 afterCompletion() 方法
		  */
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


#### 看完 doDispatch() 方法，总结出该方法中的主角 
1. **HttpServletRequest** : http请求 ；
2. **HandlerExecutionChain** : 处理请求执行链(翻译) ，里面包含（处理请求的controller的method 对象， beanFactory, http请求，还有配置的拦截器 HandlerInterceptor 数组链，等.. ）；
3. **ModelAndView** :视图返回对象，两个属性（view视图，和 modelMap属性集合 ）,当请求是@ResponseBody 类型，ModelAndView 对象为null ；
4. **Exception** : 异常请求，若请求处理抛出异常，Exception 不为空，则需要 HandlerExceptionResolver 进异常处理（根据业务可以返回异常页面，也可以返回出异常的json消息）；
5. **HandlerMapping** : getHandler()方法 根据请求的url ，到已经初始化的 urlMap（请求与处理主体的map映射集合） 中, 找到匹配请求处理的 HandlerExecutionChain 对象 ；
6. **HandlerAdapter** : 请求适配器， supports() 方法判断请求是否能处理，handle()方法处理请求，返回 ModelAndView 对象 （其实还包含response对象，因为处理请求后返回值最后回归到response属性上， @ResponseBody 的ModelAndView 返回是null ，所以只有response对象 ）；
7. **ViewResolver** : 视图解释器， InternalResourceViewResolver(spring MVC默认配置的视图解释器)  ，也可以自定义，主要是渲染成 html(jsp) 页面文档(一大串字符串)返回 ；
8. **HandlerMethod** :  HandlerExecutionChain 对象中的 handler (在类中定义的是 Object 类型)，但是请求处理过程中，它是一个 HandlerMethod 类型；
9. **HandlerInterceptor** : 请求拦截器，请求处理前（preHandle方法），请求处理后（postHandle()方法），请求完成结束后（afterCompletion()方法 ）；
10. **HttpServletResponse** : http response对象
11. **HandlerExceptionResolver** : 异常处理解释器 resolveException() 方法，返回ModelAndView 视图对象， 或者个性化设计；

#### doDispatch() 方法中的几个重要方法：
* **getHandler(processedRequest) **方法 获取类 HandlerExecutionChain 我理解成存放这个 url 请求的各种信息的类（bean , method ,参数 , 拦截器 ,beanFactory 等等）
*  **getHandlerAdapter(mappedHandler.getHandler()) **方法 获取请求处理类 handlerAdapter ( 通过handlerAdapter类的 supports() 方法判断)
*  **ha.handle(processedRequest, response, mappedHandler.getHandler()) **方法返回 ModelAndView 视图(以及response)
*  **processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException)** 结果处理方法以及异常处理

#### 一个个来啊！
##### getHandler(processedRequest) 方法 返回 HandlerExecutionChain 对象
代码：
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
			getHandlerInternal 通过获取request 的url 请求 ，从urlMap 找到RequestMapping的Method 封装成 handlder( HandlerMethod 类 )
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
// url请求
					String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
					//找到匹配该请求的 MappedInterceptor 拦截器
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
##### RequestMappingInfoHandlerMapping 类进行处理
##### 结构：
##### 继承 AbstractHandlerMethodMapping 类
##### 继承 AbstractHandlerMapping 类
##### 实现 HandlerMapping 接口实现 getHandler(HttpServletRequest request) throws Exception; 方法
AbstractHandlerMapping 调用 getHandler() 方法，通过模板方法模式，控制子类实现方法(**AbstractHandlerMethodMapping 的 getHandlerInternal()方法** 返回的是HandlerMethod对象 )来处理

**AbstractHandlerMethodMapping 的 getHandlerInternal()方法** 根据url 获取到请求处理的method ，封装成了  HandlerMethod (这不就是 HandlerAdapter supports()方法里判断的那个类型吗！！ )




##### getHandlerAdapter(mappedHandler.getHandler()) 方法，返回 HandlerAdapter 对象
代码：
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

#####  RequestMappingHandlerAdapter 类进行处理
#####  结构 
#####  继承 AbstractHandlerMethodAdapter 类
#####  实现 HandlerAdapter 接口 	 
 实现三个方法： 
* 	**boolean supports(Object handler);**
* 	**ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;**
* 	**long getLastModified(HttpServletRequest request, Object handler);**

AbstractHandlerMethodAdapter 类中
1. handle() 方法，控制继承子类实现 handleInternal() 方法 （还是模板方法模式）
2. supports() 方法， 判断了 handler的类型 且 控制子类的实现 supportsInternal() 方法
	AbstractHandlerMethodAdapter 代码：
	```Java
	public final boolean supports(Object handler) {
		return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
	}
	```

又观子类 **RequestMappingHandlerAdapter** 类中：
##### supportsInternal()方法 直接返回true ,即只判断handler 类型是否为 HandlerMethod；




#### ha.handle(processedRequest, response, mappedHandler.getHandler()) 方法，最终返回视图 ModelAndView 对象
**RequestMappingHandlerAdapter** 类处理
结构：
继承 **AbstractHandlerMethodAdapter** 类
实现 **HandlerAdapter** 接口 **supports() , handle() , getLastModified()** 方法
 

**AbstractHandlerMethodAdapter** 类中实现了 **handle()** 方法
子类 RequestMappingHandlerAdapter 类中：
* **handleInternal()** 方法中，调用了 invokeHandleMethod() 方法；
* **invokeHandleMethod()** 方法中，
1. 调用 **requestMappingMethod.invokeAndHandle(webRequest, mavContainer);**
	**invokeAndHandle() 方法中 ：**	1.Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs); 调用handler 处理请求获取到返回值 如果是String类型返回值，返回的是页面名称，responseBody的map 结果，返回的是map 对象 ， 也就是@RequestMapping 方法的返回值；
	2.调用 returnValueHandlers 的 returnValueHandlers.handleReturnValue() 方法  (也就是 messageConverters 出现的地方) 对请求返回值进行处理；
2. 调用 getModelAndView() 方法
   将处理结果封装 ModelAndView 对象
   1.如果是responseBody 返回 null 的 ModelAndView对象
   2.如果是String 返回页面视图类






#### processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException) 方法处理视图并跳转到页面 （处理异常HandlerExceptionResolvers 出现的地方）
代码：


```

	/**
	 * Handle the result of handler selection and handler invocation, which is
	 * either a ModelAndView or an Exception to be resolved to a ModelAndView.
	 */
	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {
		boolean errorView = false;

		if (exception != null) {
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
				/*
					处理异常，
					异常HandlerExceptionResolvers处理器
				*/
				mv = processHandlerException(request, response, handler, exception);
				errorView = (mv != null);
			}
		}

		// Did the handler return a view to render?
		if (mv != null && !mv.wasCleared()) {
			/*
				跳转页面 
				ViewResolver 用来解析页面
			*/
			render(mv, request, response);
			if (errorView) {
				WebUtils.clearErrorRequestAttributes(request);
			}
		}
		else {
			if (logger.isDebugEnabled()) {
				logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + getServletName() +
						"': assuming HandlerAdapter completed request handling");
			}
		}

		if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Concurrent handling started during a forward
			return;
		}

		if (mappedHandler != null) {
			// 调用了 拦截器 HandlerInterceptor 的afterCompletion() 方法
			mappedHandler.triggerAfterCompletion(request, response, null);
		}
	}
```

#### 处理请求分3类(controller 中的方法)：
* 返回值是String (跳转页面): ModelAndView 对象不为空，最后通过ViewResolver处理 ；
* 定义了 @ResponseBody ( 返回json ): ModelAndView 对象为空(null)，请求处理结果都在response中最后返回 ；
* controller中抛异常 ： HandlerExceptionResolver 进行处理；





