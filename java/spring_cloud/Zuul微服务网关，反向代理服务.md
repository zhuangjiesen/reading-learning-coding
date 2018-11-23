# Zuul微服务网关，反向代理服务

#### 运行逻辑
```
配置文件中配置路由规则
找注册中心的服务与路由路径的关系
通过ZuulServlet.class
service()方法，加载到服务的路径，
加载hystrix处理模块封装实体请求，
通过okHttp去请求http请求，
收到的http请求结果(ZuulFilterResult.class)成功的话直接返回->写入response的输出流(body)
不然就走错误处理机制返回前端错误信息

注解进入@EnableZuulProxy

@Import(ZuulProxyConfiguration.class) 的ZuulProxyConfiguration.class 类
ZuulConfiguration.class类
加载：ZuulServlet.class
ZuulHandlerMapping.class 用来映射请求的
```


#### 核心 ZuulServlet.class 代码：
```
service()方法中处理请求

	init() -> 初始化 RequestContext.class (继承自ConcurrentHashMap.class)
	主要把request、response、还有其他的属性封装
	以及一些代理请求的参数
	用来流转到各个 ZuulFilter.class
	route() 方法中加载ZuulFilter.class 的实现：
		RibbonRoutingFilter.class
		SimpleHostRoutingFilter.class
		SendForwardFilter.class
	RibbonCommandContext.class 将代理的路由请求
	OkHttpRibbonCommandFactory.class 创建http请求
	OkHttpRibbonCommand.class 发起请求
```

#### ZuulFilter.class 核心处理链路
```

public abstract class ZuulFilter implements IZuulFilter, Comparable<ZuulFilter> {

    /**
     * to classify a filter by type. Standard types in Zuul are "pre" for pre-routing filtering,
     * "route" for routing to an origin, "post" for post-routing filters, "error" for error handling.
     * We also support a "static" type for static responses see  StaticResponseFilter.
     * Any filterType made be created or added and run by calling FilterProcessor.runFilters(type)
     *
     * filter的类型： pre 、route 、post、error
     * @return A String representing that type
     */
    abstract public String filterType();

    /**
     * filterOrder() must also be defined for a filter. Filters may have the same  filterOrder if precedence is not
     * important for a filter. filterOrders do not need to be sequential.
     *
     * filter被执行的顺序
     * @return the int order of a filter
     */
    abstract public int filterOrder();

  	....省略部分代码

    /**
     * runFilter checks !isFilterDisabled() and shouldFilter(). The run() method is invoked if both are true.
     * 调用filter的run()方法，返回封装的 ZuulFilterResult.class 结果
     * @return the return from ZuulFilterResult
     */
    public ZuulFilterResult runFilter() {
        ZuulFilterResult zr = new ZuulFilterResult();
        if (!isFilterDisabled()) {
            if (shouldFilter()) {
                Tracer t = TracerFactory.instance().startMicroTracer("ZUUL::" + this.getClass().getSimpleName());
                try {
                	//调用接口的实现方法
                    Object res = run();
                    zr = new ZuulFilterResult(res, ExecutionStatus.SUCCESS);
                } catch (Throwable e) {
                    t.setName("ZUUL::" + this.getClass().getSimpleName() + " failed");
                    zr = new ZuulFilterResult(ExecutionStatus.FAILED);
                    zr.setException(e);
                } finally {
                    t.stopAndLog();
                }
            } else {
                zr = new ZuulFilterResult(ExecutionStatus.SKIPPED);
            }
        }
        return zr;
    }


}
```

#### IZuulFilter.class 
```
public interface IZuulFilter {
    /**
     * a "true" return from this method means that the run() method should be invoked
     * 执行前会调用，返回true才会接着执行run()方法
     * @return true if the run() method should be invoked. false will not invoke the run() method
     */
    boolean shouldFilter();

    /**
     * if shouldFilter() is true, this method will be invoked. this method is the core method of a ZuulFilter
     * 实现 IZuulFilter.class 接口需要实现的方法
     * @return Some arbitrary artifact may be returned. Current implementation ignores it.
     */
    Object run();

}
```


https://blog.csdn.net/forezp/article/details/76211680

hystrix 源码
okhttp源码





