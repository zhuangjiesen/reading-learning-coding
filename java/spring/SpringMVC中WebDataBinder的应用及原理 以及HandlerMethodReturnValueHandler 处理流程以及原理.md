# SpringMVC中WebDataBinder的应用及原理 以及HandlerMethodReturnValueHandler 处理流程以及原理

```
@InitBinder用于在@Controller中标注于方法，表示为当前控制器注册一个属性编辑器或者其他，只对当前的Controller有效
也可以通过定义父类BaseController.class ,全局设置 @InitBinder 方法
```

```
http参数类型以及Controller中method定义的方法参数类型

GET 1.直接类型 如：String.class Integer.class 参数处理类：RequestParamMethodArgumentResolver.class
    2.业务AO 如：User.class Project.class 参数处理类：ModelAttributeMethodProcessor.class (ServletModelAttributeMethodProcessor.class)
    
POST @RequestBody 如：
   1.Map<String , Object>
   2.业务AO 如：User.class Project.class
参数处理类 ：RequestResponseBodyMethodProcessor.class 
```

### ModelAttributeMethodProcessor.class
#### ModelAttributeMethodProcessor.class 是用来解析get请求参数并赋值(格式化)给AO对象的 (例子 DataBinderController.testValidateGet()方法 )


#### resolveArgument()方法
```


	/**
	 * Resolve the argument from the model or if not found instantiate it with
	 * its default if it is available. The model attribute is then populated
	 * with request values via data binding and optionally validated
	 * if {@code @java.validation.Valid} is present on the argument.
	 * @throws BindException if data binding and validation result in an error
	 * and the next method parameter is not of type {@link Errors}
	 * @throws Exception if WebDataBinder initialization fails
	 */
	@Override
	@Nullable
	public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		Assert.state(mavContainer != null, "ModelAttributeMethodProcessor requires ModelAndViewContainer");
		Assert.state(binderFactory != null, "ModelAttributeMethodProcessor requires WebDataBinderFactory");

		String name = ModelFactory.getNameForParameter(parameter);
		ModelAttribute ann = parameter.getParameterAnnotation(ModelAttribute.class);
		if (ann != null) {
			mavContainer.setBinding(name, ann.binding());
		}

		Object attribute = null;
		BindingResult bindingResult = null;

		if (mavContainer.containsAttribute(name)) {
			attribute = mavContainer.getModel().get(name);
		}
		else {
			// Create attribute instance
			try {
				attribute = createAttribute(name, parameter, binderFactory, webRequest);
			}
			catch (BindException ex) {
				if (isBindExceptionRequired(parameter)) {
					// No BindingResult parameter -> fail with BindException
					throw ex;
				}
				// Otherwise, expose null/empty value and associated BindingResult
				if (parameter.getParameterType() == Optional.class) {
					attribute = Optional.empty();
				}
				bindingResult = ex.getBindingResult();
			}
		}

		if (bindingResult == null) {
			// Bean property binding and validation;
			// skipped in case of binding failure on construction.
			WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
			if (binder.getTarget() != null) {
				if (!mavContainer.isBindingDisabled(name)) {
				//参数绑定操作
					bindRequestParameters(binder, webRequest);
				}
				//校验参数合法性 (Validator.class 的validate()方法)
				validateIfApplicable(binder, parameter);
				if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
					throw new BindException(binder.getBindingResult());
				}
			}
			// Value type adaptation, also covering java.util.Optional
			if (!parameter.getParameterType().isInstance(attribute)) {
				attribute = binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
			}
			//校验完成后，把bindingResult 结果返回给业务层，自己处理
			bindingResult = binder.getBindingResult();
		}

		// Add resolved attribute and BindingResult at the end of the model
		Map<String, Object> bindingResultModel = bindingResult.getModel();
		//把校验结果添加到了视图层的属性中
		mavContainer.removeAttributes(bindingResultModel);
		mavContainer.addAllAttributes(bindingResultModel);

		return attribute;
	}

```

#### bindRequestParameters()方法ServletModelAttributeMethodProcessor.class 用来绑定get请求和AO对象的
```

	/**
	 * This implementation downcasts {@link WebDataBinder} to
	 * {@link ServletRequestDataBinder} before binding.
	 * @see ServletRequestDataBinderFactory
	 */
	@Override
	protected void bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
		ServletRequest servletRequest = request.getNativeRequest(ServletRequest.class);
		Assert.state(servletRequest != null, "No ServletRequest");
		ServletRequestDataBinder servletBinder = (ServletRequestDataBinder) binder;
		servletBinder.bind(servletRequest);
	}


```

```	
public void bind(ServletRequest request) {
		MutablePropertyValues mpvs = new ServletRequestParameterPropertyValues(request);
		MultipartRequest multipartRequest = WebUtils.getNativeRequest(request, MultipartRequest.class);
		if (multipartRequest != null) {
			bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
		}
		addBindValues(mpvs, request);
		doBind(mpvs);
	}
```

#### 参数解析小结
```
1.先将request中的参数解析成key/value格式 (MutablePropertyValues.class)
2.获取binder的target对象（Controller中的method参数对象）
3.判空(requered判断) , 格式化 ，赋值
```


### DataBinder 与 MessageConverter 区别

```
RequestResponseBodyMethodProcessor.resolveArgument()方法中，调用了messageConverter解析(格式化)请求体数据参数，获取 WebDataBinder 校验参数合法性

1.MessageConverter 用来解析(格式化)@RequestBody(http请求体)的报文，格式化成参数

2.DataBinder 可以用来解析(格式化)参数，也可以用来校验参数合法性 (Post请求体的参数是通过messageConvertor解析的)
(Validator.class)
```

#### 代码片段
```

	/**
	 * Throws MethodArgumentNotValidException if validation fails.
	 * @throws HttpMessageNotReadableException if {@link RequestBody#required()}
	 * is {@code true} and there is no body content or if there is no suitable
	 * converter to read the content with.
	 */
	@Override
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		parameter = parameter.nestedIfOptional();
		//通过 messageConverter 获取参数值
		Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
		String name = Conventions.getVariableNameForParameter(parameter);

		if (binderFactory != null) {
		//获取 WebDataBinder
			WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
			if (arg != null) {
			//校验参数调用Validator.class 接口
				validateIfApplicable(binder, parameter);
				if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
					throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
				}
			}
			if (mavContainer != null) {
				mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
			}
		}

		return adaptArgumentIfNecessary(arg, parameter);
	}
```


### 测试全部代码片段：

```


/**
 * @param
 * @Author: zhuangjiesen
 * @Description:
 * @Date: Created in 2018/9/9
 */
@RestController
@Slf4j
@RequestMapping(value = "/data/binder")
public class DataBinderController {

    @Autowired
    private CommonValidator commonValidator;
    @InitBinder
    public void initBainder(DataBinder binder){
        //参数校验器
        binder.addValidators(commonValidator);
        DateFormatter dateFormatter = new DateFormatter();
        //时间格式转换器
        binder.addCustomFormatter(dateFormatter);
  }


    /**
     * 请求参数是User
     *
     * @author zhuangjiesen
     * @date 2018/9/9 下午4:14
     * @param
     * @return
     */
    @RequestMapping(value = "/validate/test/post", method = RequestMethod.POST)
    public String testValidatePost(@RequestBody @Validated User user , BindingResult bindingResult){
        log.info(String.format(" user : %s " , JSONObject.toJSONString(user)));

        if (CollectionUtils.isNotEmpty(bindingResult.getFieldErrors())) {
            bindingResult.getFieldErrors().forEach(new Consumer<FieldError>() {
                @Override
                public void accept(FieldError fieldError) {
                    log.info(String.format("field : %s , Error : %s " , fieldError.getField(), fieldError.getCode()));
                }
            });
        }
        log.info(String.format(" bindingResult : %s " , bindingResult.toString()));

        return "success";
    }




    /**
     *  普通请求，不参与校验
     *
     * @author zhuangjiesen
     * @date 2018/9/9 下午4:13
     * @param
     * @return
     */
    @RequestMapping(value = "/validate/test/postNone", method = RequestMethod.POST)
    public String testValidatePost(@RequestBody User user ){
        log.info(String.format(" user : %s " , JSONObject.toJSONString(user)));

        return "postnone";
    }


    /**
     * 请求参数是map
     *
     * @author zhuangjiesen
     * @date 2018/9/9 下午4:14
     * @param
     * @return
     */
    @RequestMapping(value = "/validate/test/postMap", method = RequestMethod.POST)
    public String testValidatePostMap(@RequestBody @Validated Map<String , Object> user , BindingResult bindingResult){
        log.info(String.format(" user : %s " , JSONObject.toJSONString(user)));

        if (CollectionUtils.isNotEmpty(bindingResult.getFieldErrors())) {
            bindingResult.getFieldErrors().forEach(new Consumer<FieldError>() {
                @Override
                public void accept(FieldError fieldError) {
                    log.info(String.format("field : %s , Error : %s " , fieldError.getField(), fieldError.getCode()));
                }
            });
        }
        log.info(String.format(" bindingResult : %s " , bindingResult.toString()));

        return "success";
    }


    /**
     * get方法的参数校验
     *
     * @author zhuangjiesen
     * @date 2018/9/9 下午4:14
     * @param
     * @return
     */
    @RequestMapping(value = "/validate/test/get", method = RequestMethod.GET)
    public String testValidateGet( @Validated User user , BindingResult bindingResult){
        log.info(String.format(" user : %s " , JSONObject.toJSONString(user)));

        if (CollectionUtils.isNotEmpty(bindingResult.getFieldErrors())) {
            bindingResult.getFieldErrors().forEach(new Consumer<FieldError>() {
                @Override
                public void accept(FieldError fieldError) {
                    log.info(String.format("field : %s , Error : %s " , fieldError.getField(), fieldError.getCode()));
                }
            });
        }
        log.info(String.format(" bindingResult : %s " , bindingResult.toString()));

        return "success";
    }



    /**
     * 普通的get参数校验
     *
     * @author zhuangjiesen
     * @date 2018/9/9 下午4:14
     * @param
     * @return
     */
    @RequestMapping(value = "/validate/test/getCommon", method = RequestMethod.GET)
    public String getCommon( String name , Date createTime){

        return "success";
    }



}

```

#### CommonValidator.class

```   
/**
 * @param
 * @Author: zhuangjiesen
 * @Description:
 * @Date: Created in 2018/9/4
 */
@Component
@Slf4j
public class CommonValidator implements Validator {
    
    @Override
    public boolean supports(Class<?> clazz) {
        log.info(String.format("supports - clazz : %s " , clazz.getName()));
        return clazz.equals(User.class) || clazz.asSubclass(Map.class) != null;
    }

    @Override
    public void validate(@Nullable Object target, Errors errors) {
        if (target instanceof User) {
            User user = (User) target;
            if (user.getPassword() == null) {
                errors.rejectValue("password" , "密码不能为空哈哈哈");
            }
        } else if (target instanceof Map) {
            Map<String,Object> user = (Map) target;
            if (user.get("password") == null) {
                // rejectValue 会报错，因为Map中需要有get/set 的field
                errors.reject("密码不能为空哈哈哈");
            }
        }
        log.info(String.format("supports - target : %s , errors : %s " , target , errors));
    }
}


```


### 调试日志(例子)

见项目



### HandlerMethodReturnValueHandler.class
```
Controller中method执行完的returnValue处理
```
#### 代码：
```

	/**
	 * Whether the given {@linkplain MethodParameter method return type} is
	 * supported by this handler.
	 * @param returnType the method return type to check
	 * @return {@code true} if this handler supports the supplied return type;
	 * {@code false} otherwise
	 */
	boolean supportsReturnType(MethodParameter returnType);

	/**
	 * Handle the given return value by adding attributes to the model and
	 * setting a view or setting the
	 * {@link ModelAndViewContainer#setRequestHandled} flag to {@code true}
	 * to indicate the response has been handled directly.
	 * @param returnValue the value returned from the handler method
	 * @param returnType the type of the return value. This type must have
	 * previously been passed to {@link #supportsReturnType} which must
	 * have returned {@code true}.
	 * @param mavContainer the ModelAndViewContainer for the current request
	 * @param webRequest the current request
	 * @throws Exception if the return value handling results in an error
	 */
	void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;

```

### 也是排好序数组，也是很多种，也是遍历执行supportsReturnType()方法获取处理类
```
HttpEntityMethodProcessor.class
纯文本的返回值，直接写到响应体中

RequestResponseBodyMethodProcessor.class 处理 @ResonseBody 注解的返回，通过messageConverter封装成对应(ContentType)的response报文

```


### HandlerMethodReturnValueHandlerComposite.class 聚合对象，获取支持处理的处理类，并处理


### 执行入口：
```
RequestMappingHandlerAdapter.invokeHandlerMethod()方法

ServletInvocableHandlerMethod.invokeAndHandle() 方法

```


#### ServletInvocableHandlerMethod.invokeAndHandle() 方法
```


	/**
	 * Invoke the method and handle the return value through one of the
	 * configured {@link HandlerMethodReturnValueHandler}s.
	 * @param webRequest the current request
	 * @param mavContainer the ModelAndViewContainer for this request
	 * @param providedArgs "given" arguments matched by type (not resolved)
	 */
	public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		setResponseStatus(webRequest);

		if (returnValue == null) {
			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
				mavContainer.setRequestHandled(true);
				return;
			}
		}
		else if (StringUtils.hasText(getResponseStatusReason())) {
			mavContainer.setRequestHandled(true);
			return;
		}

		mavContainer.setRequestHandled(false);
		Assert.state(this.returnValueHandlers != null, "No return value handlers");
		try {
		//返回值处理入口
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
			}
			throw ex;
		}
	}
```

### RequestResponseBodyMethodProcessor.class 

```
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
				returnType.hasMethodAnnotation(ResponseBody.class));
	}


	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

		mavContainer.setRequestHandled(true);
		ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
		ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

		// Try even with null return value. ResponseBodyAdvice could get involved.
		
		//根据contenttype以及配置的对象格式，进行序列化返回，写入reponse对象的输出流
		writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
	}
```

