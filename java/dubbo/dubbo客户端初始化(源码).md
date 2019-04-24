# dubbo客户端初始化(源码)
### dubbo 客户端初始化疑问
```
都知道是用netty初始化，但是细节不是清楚，比如：
1. netty连接是否复用
2. dubbo连接数量管理
3. dubbo异步同步请求的实现
4. dubbo单连接，怎么(多线程下)管理request(dubbo)/response(dubbo)
5. 负载均衡
```
#### 重点类 ReferenceBean , ReferenceConfig , DubboProtocol , DubboInvoker , HeaderExchanger , HeaderExchangeHandler(netty事件调度类) , DefaultFuture

#### dubbo客户端的调用都是走的 DubboInvoker.java , 但是调用是层层封装的

### Dubbo 客户端配置(xml配置)
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans          
    http://www.springframework.org/schema/beans/spring-beans.xsd          
    http://code.alibabatech.com/schema/dubbo          
    http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

     <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->  
    <dubbo:application name="hjy_consumer" />
  
    <!-- 使用zookeeper注册中心暴露服务地址 -->  
    <!-- <dubbo:registry address="multicast://224.5.6.7:1234" /> -->  
    <!--<dubbo:registry id="zkRegisty" address="zookeeper://127.0.0.1:2181" />-->

    <dubbo:registry id="redisRegister" address="customerRedis://127.0.0.1:6379?password=redis"  />

    <!-- 生成远程服务代理，可以像使用本地bean一样使用demoService -->  
    <dubbo:reference registry="redisRegister"   protocol="dubbo"  id="dubboTestService"
        interface="com.java.core.rpc.dubbo.service.IDubboTestService" />
    <!-- 生成远程服务代理，可以像使用本地bean一样使用demoService -->
    <dubbo:reference registry="redisRegister"   protocol="dubbo" id="dubboInfoTestService"
        interface="com.java.core.rpc.dubbo.service.IDubboInfoTestService"  />

    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol id="dubbo"  name="dubbo" port="20880" />
    <dubbo:protocol    name="thrift" port="30880" />
</beans>

```


### 客户端源码入口 ReferenceBean
#### 初始化Dubbo 的时候，dubbo初始化了 ReferenceBean.java
```
ReferenceBean.java 实现 InitializingBean.java , FactoryBean.java 接口

FactoryBean.java 接口，重点是 getObject() 和 getObjectType() 
用来实现抽象化代理 reference 配置里的接口(一般抽象化框架都是用这个套路)

InitializingBean.java 接口，重点是  afterPropertiesSet() 方法
一般基于spring的框架的代码初始入口都在这里
1. getObject() 方法中初始化了 reference 对于dubbo客户端的请求
-> 调用的是-> ReferenceConfig.get() -> ReferenceConfig.init() -> ReferenceConfig.createProxy(map) 初始化代理

invoker = refprotocol.refer(interfaceClass, urls.get(0));
// refprotocol -> DubboProtocol 初始化invoker

```


### Dubbo初始化连接入口 DubboProtocol

#### DubboProtocol.java 
```


    private ExchangeClient[] getClients(URL url) {
        // whether to share connection
        boolean service_share_connect = false;
        int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
        // if not configured, connection is shared, otherwise, one connection for one service
        // 如果没有配置，连接就是共享的，否则一个连接一个客户端
        if (connections == 0) {
            service_share_connect = true;
            connections = 1;
        }

        ExchangeClient[] clients = new ExchangeClient[connections];
        for (int i = 0; i < clients.length; i++) {
            if (service_share_connect) {
                //如果是共享连接，就获取map里的连接，dubboProtocol是个单例，所以数据不需要静态(static)
                clients[i] = getSharedClient(url);
            } else {
                clients[i] = initClient(url);
            }
        }
        return clients;
    }
```

### 初始化 ExchangeClient.java (基于 LazyConnectExchangeClient.java(dubbo客户端管理) , NettyClient.java(原生netty客户端) )
#### 共享连接 ReferenceCountExchangeClient.java (通过封装LazyConnectExchangeClient.java)

#### DubboProtocol.java 的 getSharedClient() 方法
```


    /**
     * Get shared connection
     */
    private ExchangeClient getSharedClient(URL url) {
        String key = url.getAddress();
        ReferenceCountExchangeClient client = referenceClientMap.get(key);
        if (client != null) {
            if (!client.isClosed()) {
                client.incrementAndGetCount();
                return client;
            } else {
                referenceClientMap.remove(key);
            }
        }
        //map锁(分段锁？)
        locks.putIfAbsent(key, new Object());
        synchronized (locks.get(key)) {
            if (referenceClientMap.containsKey(key)) {
                return referenceClientMap.get(key);
            }
            //初始化 LazyConnectExchangeClient.java 
            ExchangeClient exchangeClient = initClient(url);
            //封装 ReferenceCountExchangeClient.java 
            client = new ReferenceCountExchangeClient(exchangeClient, ghostClientMap);
            referenceClientMap.put(key, client);
            ghostClientMap.remove(key);
            locks.remove(key);
            return client;
        }
    }
......



    /**
     * Create new connection
     * 创建新连接
     */
    private ExchangeClient initClient(URL url) {

        // client type setting.
        String str = url.getParameter(Constants.CLIENT_KEY, url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_CLIENT));

        url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
        // enable heartbeat by default
        url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));

        // BIO is not allowed since it has severe performance issue.
        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported client type: " + str + "," +
                    " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
        }

        ExchangeClient client;
        try {
            // connection should be lazy
            if (url.getParameter(Constants.LAZY_CONNECT_KEY, false)) {
                //lazy 配置，只初始化对象，并没有建立连接
                client = new LazyConnectExchangeClient(url, requestHandler);
            } else {
                //建立tcp连接
                client = Exchangers.connect(url, requestHandler);
            }
        } catch (RemotingException e) {
            throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
        }
        return client;
    }



```

#### 建立tcp连接，Exchangers.connect(url ,requestHandler);
#### requestHandler是netty的handlerAdapter，dubbo服务端用来处理请求的

```

public class HeaderExchanger implements Exchanger {
    
    public static final String NAME = "header";

    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        //建立客户端连接请求 -> 这里封装一层client是为了 heartbeat 心跳任务，默认不执行
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

}

```


#### requestHandler -> DubboProtocol里有个匿名内部类变量 ->  requestHandler
```

    private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
        
        public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
            if (message instanceof Invocation) {
                Invocation inv = (Invocation) message;
                //获取invoker ， 并提供连接 channel
                Invoker<?> invoker = getInvoker(channel, inv);
                //如果是callback 需要处理高版本调用低版本的问题
                if (Boolean.TRUE.toString().equals(inv.getAttachments().get(IS_CALLBACK_SERVICE_INVOKE))){
                    String methodsStr = invoker.getUrl().getParameters().get("methods");
                    boolean hasMethod = false;
                    if (methodsStr == null || methodsStr.indexOf(",") == -1){
                        hasMethod = inv.getMethodName().equals(methodsStr);
                    } else {
                        String[] methods = methodsStr.split(",");
                        for (String method : methods){
                            if (inv.getMethodName().equals(method)){
                                hasMethod = true;
                                break;
                            }
                        }
                    }
                    if (!hasMethod){
                        logger.warn(new IllegalStateException("The methodName "+inv.getMethodName()+" not found in callback service interface ,invoke will be ignored. please update the api interface. url is:" + invoker.getUrl()) +" ,invocation is :"+inv );
                        return null;
                    }
                }
                RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
                return invoker.invoke(inv);
            }
            throw new RemotingException(channel, "Unsupported request: " + message == null ? null : (message.getClass().getName() + ": " + message) + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
        }

        @Override
        public void received(Channel channel, Object message) throws RemotingException {
            //获取请求后判断参数
            if (message instanceof Invocation) {
                //执行apply() 方法，在apply中调用服务端invoker，调用服务端代码返回结果
                reply((ExchangeChannel) channel, message);
            } else {
                super.received(channel, message);
            }
        }

        @Override
        public void connected(Channel channel) throws RemotingException {
            invoke(channel, Constants.ON_CONNECT_KEY);
        }

        @Override
        public void disconnected(Channel channel) throws RemotingException {
            if(logger.isInfoEnabled()){
                logger.info("disconected from "+ channel.getRemoteAddress() + ",url:" + channel.getUrl());
            }
            invoke(channel, Constants.ON_DISCONNECT_KEY);
        }
        
        private void invoke(Channel channel, String methodKey) {
            Invocation invocation = createInvocation(channel, channel.getUrl(), methodKey);
            if (invocation != null) {
                try {
                    received(channel, invocation);
                } catch (Throwable t) {
                    logger.warn("Failed to invoke event method " + invocation.getMethodName() + "(), cause: " + t.getMessage(), t);
                }
            }
        }
        
        private Invocation createInvocation(Channel channel, URL url, String methodKey) {
            String method = url.getParameter(methodKey);
            if (method == null || method.length() == 0) {
                return null;
            }
            RpcInvocation invocation = new RpcInvocation(method, new Class<?>[0], new Object[0]);
            invocation.setAttachment(Constants.PATH_KEY, url.getPath());
            invocation.setAttachment(Constants.GROUP_KEY, url.getParameter(Constants.GROUP_KEY));
            invocation.setAttachment(Constants.INTERFACE_KEY, url.getParameter(Constants.INTERFACE_KEY));
            invocation.setAttachment(Constants.VERSION_KEY, url.getParameter(Constants.VERSION_KEY));
            if (url.getParameter(Constants.STUB_EVENT_KEY, false)){
                invocation.setAttachment(Constants.STUB_EVENT_KEY, Boolean.TRUE.toString());
            }
            return invocation;
        }
    };
    

```

### 解答问题：
```
1. 连接复用问题，如果整个工程一个nettyChannel，涉及到的是netty的channel.write() 是否线程安全，通过看netty源码，netty的writeAndFlush() 是线程安全的，通过ioEventLoop执行ioTask，判断了io任务，如果获取不到锁放入io任务队列
2. 连接数量通过DubboProtocol 管理(clients)
```

#### DubboInvoker -> 请求结果事件，封装成 Future 对象
```


    @Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);
        
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            //是否同步请求
            boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY,Constants.DEFAULT_TIMEOUT);
            if (isOneway) {
            	boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                RpcContext.getContext().setFuture(null);
                return new RpcResult();
            } else if (isAsync) {
            //如果异步请求，注册future，设置到 RpcContext中，返回空结果，如果异步获取，就需要从RpcContext.getContext() 获取
            	ResponseFuture future = currentClient.request(inv, timeout) ;
                RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                return new RpcResult();
            } else {
            //同步请求，通过future.get() 阻塞当前线程等待结果，但是netty的线程池不是独占
            	RpcContext.getContext().setFuture(null);
                return (Result) currentClient.request(inv, timeout).get();
            }
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

```

### RpcContext.java 
```
这个类通过一个ThreadLocal存储future的数据，异步请求中，将请求结果的监听future 注册到RpcContext的 LOCAL变量中，直接返回空Result

```

#### 3.异步同步请求的实现
```
1.异步请求，请求前将future设置到 RpcContext中的ThreadLocal，然后直接返回，之后在当前线程RpcContext.getContext().getFuture()获取future,然后获取请求结果
2.同步请求，请求后直接用future.get() 阻塞线程等待结果

```

#### 4.单连接下，怎么管理的request/response 
```
1.Request.java 中有个 AtomicLong id自增，给request设置成id也是reponse的id
DefaultFuture 里有个concurrentHashMap ，请求返回时，获取得到对应future对象，然后返回结果，把结果set给future,然后对应的future的get() 方法就能直接返回了

```

### dubbo客户端/服务端与netty对接
#### HeaderExchanger.java 
```


public class HeaderExchanger implements Exchanger {
    
    public static final String NAME = "header";

    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

}

```
#### HeaderExchangeHandler.java
#### 这个是个重点类，个人理解是netty请求的调度类，比如说客户端发起请求吧，request发送需要invoker的显式执行，但是reponse需要框架统一处理，这时候就是 HeaderExchangeHandler.java类里的 received() -> response那行
```
public class HeaderExchangeHandler implements ChannelHandlerDelegate {

    protected static final Logger logger              = LoggerFactory.getLogger(HeaderExchangeHandler.class);

    public static String          KEY_READ_TIMESTAMP  = HeartbeatHandler.KEY_READ_TIMESTAMP;

    public static String          KEY_WRITE_TIMESTAMP = HeartbeatHandler.KEY_WRITE_TIMESTAMP;

    private final ExchangeHandler handler;
    
    
    
    public void received(Channel channel, Object message) throws RemotingException {
        channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
        ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
        try {
            if (message instanceof Request) {
                // handle request.
                Request request = (Request) message;
                if (request.isEvent()) {
                    handlerEvent(channel, request);
                } else {
                    if (request.isTwoWay()) {
                        Response response = handleRequest(exchangeChannel, request);
                        channel.send(response);
                    } else {
                        handler.received(exchangeChannel, request.getData());
                    }
                }
            } else if (message instanceof Response) {
                //当返回的是reponse，就在这里处理
                handleResponse(channel, (Response) message);
            } else if (message instanceof String) {
                if (isClientSide(channel)) {
                    Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
                    logger.error(e.getMessage(), e);
                } else {
                    String echo = handler.telnet(channel, (String) message);
                    if (echo != null && echo.length() > 0) {
                        channel.send(echo);
                    }
                }
            } else {
                handler.received(exchangeChannel, message);
            }
        } finally {
            HeaderExchangeChannel.removeChannelIfDisconnected(channel);
        }
    }

    
    static void handleResponse(Channel channel, Response response) throws RemotingException {
        if (response != null && !response.isHeartbeat()) {
            
            DefaultFuture.received(channel, response);
        }
    }

    
    
}

```

#### DefaultFuture.received() 方法
#### 这个类是future的统一处理类，也是个实例类，初始化每个请求的future
```

public class DefaultFuture implements ResponseFuture {

    private static final Logger                   logger = LoggerFactory.getLogger(DefaultFuture.class);

    private static final Map<Long, Channel>       CHANNELS   = new ConcurrentHashMap<Long, Channel>();

    private static final Map<Long, DefaultFuture> FUTURES   = new ConcurrentHashMap<Long, DefaultFuture>();

    // invoke id.
    private final long                            id;
    private final Channel                         channel;
    private final Request                         request;
    private final int                             timeout;
    private final Lock                            lock = new ReentrantLock();
    private final Condition                       done = lock.newCondition();
    private final long                            start = System.currentTimeMillis();
    private volatile long                         sent;
    private volatile Response                     response;
    private volatile ResponseCallback             callback;


    public static void received(Channel channel, Response response) {
        try {
            //通过id获取到future,设置请求结果的值
            DefaultFuture future = FUTURES.remove(response.getId());
            if (future != null) {
                future.doReceived(response);
            } else {
                logger.warn("The timeout response finally returned at " 
                            + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date())) 
                            + ", response " + response 
                            + (channel == null ? "" : ", channel: " + channel.getLocalAddress() 
                                + " -> " + channel.getRemoteAddress()));
            }
        } finally {
            CHANNELS.remove(response.getId());
        }
    }

    private void doReceived(Response res) {
        lock.lock();
        try {
            response = res;
            if (done != null) {
                done.signal();
            }
        } finally {
            lock.unlock();
        }
        if (callback != null) {
            invokeCallback(callback);
        }
    }
    
    
    
    private static class RemotingInvocationTimeoutScan implements Runnable {

        public void run() {
            while (true) {
                try {
                    for (DefaultFuture future : FUTURES.values()) {
                        if (future == null || future.isDone()) {
                            continue;
                        }
                        if (System.currentTimeMillis() - future.getStartTimestamp() > future.getTimeout()) {
                            // create exception response.
                            Response timeoutResponse = new Response(future.getId());
                            // set timeout status.
                            timeoutResponse.setStatus(future.isSent() ? Response.SERVER_TIMEOUT : Response.CLIENT_TIMEOUT);
                            timeoutResponse.setErrorMessage(future.getTimeoutMessage(true));
                            // handle response.
                            DefaultFuture.received(future.getChannel(), timeoutResponse);
                        }
                    }
                    Thread.sleep(30);
                } catch (Throwable e) {
                    logger.error("Exception when scan the timeout invocation of remoting.", e);
                }
            }
        }
    }

    static {
        //初始化一个线程，轮询map，处理请求的future超时
        Thread th = new Thread(new RemotingInvocationTimeoutScan(), "DubboResponseTimeoutScanTimer");
        th.setDaemon(true);
        th.start();
    }

}
```

#### 5.负载均衡
#### 负载均衡不是client(netty客户端)的负载均衡，是invoker的负载均衡(多个invoker对应不同的server)
#### FailoverClusterInvoker.java中实现的，负载均衡无非是轮询，权重，具体看代码吧




