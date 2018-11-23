# Java WebSocket客户端实现

### 开始使用的是 Java-WebSocket 
```
<dependency>
        <groupId>org.java-websocket</groupId>
        <artifactId>Java-WebSocket</artifactId>
        <version>1.3.4</version>
        <scope>test</scope>
</dependency>

```

#### 发现问题：
对于wss (SSL)协议，无法支持，甚至都没有报错

#### 换成 org.eclipse.jetty.websocket

### Jetty WebSocket官方文档
http://www.eclipse.org/jetty/documentation/current/websocket-jetty.html

```

    <dependency>
      <groupId>org.eclipse.jetty.websocket</groupId>
      <artifactId>websocket-api</artifactId>
      <version>9.3.8.v20160314</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.jetty.websocket</groupId>
      <artifactId>websocket-common</artifactId>
      <version>9.3.8.v20160314</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.jetty.websocket</groupId>
      <artifactId>websocket-client</artifactId>
      <version>9.3.8.v20160314</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.jetty</groupId>
      <artifactId>jetty-io</artifactId>
      <version>9.3.8.v20160314</version>
    </dependency>
    <dependency>
      <groupId>org.eclipse.jetty</groupId>
      <artifactId>jetty-util</artifactId>
      <version>9.3.8.v20160314</version>
    </dependency>
```


### 示例代码

#### WebSocketClientApp.java 示例程序的入口
```

/**
 * @param
 * @Author: zhuangjiesen
 * @Description:
 * @Date: Created in 2018/5/23
 */
public class WebSocketClientApp {

    public static void main(String[] args) {

        //用来处理wss 请求
        SslContextFactory sslContextFactory = new SslContextFactory();
        WebSocketClient client = new WebSocketClient(sslContextFactory);

        //通过注解定义监听websocket的事件
        SimpleEchoSocket simpleEchoSocket = new SimpleEchoSocket();
        try
        {
            client.start();
            URI echoUri = new URI("ws://127.0.0.1:38888");
            ClientUpgradeRequest request = new ClientUpgradeRequest();
            //添加请求头参数
//            request.setHeader("" , "");
            //尝试连接
            Future<Session> fuq = client.connect(simpleEchoSocket , echoUri,request);
            fuq.get();
        }
        catch (Throwable t)
        {
            t.printStackTrace();
            //关闭连接
            if (!client.isStopped()) {
//                client.stop();
            }
        }
        finally
        {

        }
    }


}
```


#### SimpleEchoSocket.java WebSocket连接事件触发器

```


/**
 * @param
 * @Author: zhuangjiesen
 * @Description:
 * @Date: Created in 2018/5/18
 */
@WebSocket(maxTextMessageSize = 64 * 1024)
public class SimpleEchoSocket {
    private final CountDownLatch closeLatch;
    @SuppressWarnings("unused")
    /** websocket 连接对象 **/
    private Session session;

    public SimpleEchoSocket()
    {
        this.closeLatch = new CountDownLatch(1);
    }

    public boolean awaitClose(int duration, TimeUnit unit) throws InterruptedException
    {
        return this.closeLatch.await(duration,unit);
    }


    @OnWebSocketError
    public void onError(Throwable throwable)
    {
        System.out.printf("throwable : %s ", throwable);

    }

    @OnWebSocketClose
    public void onClose(int statusCode, String reason)
    {
        System.out.printf("Connection closed: %d - %s%n",statusCode,reason);
        this.session = null;
        this.closeLatch.countDown(); // trigger latch
    }

    @OnWebSocketConnect
    public void onConnect(Session session)
    {
        System.out.printf("Got connect: %s%n",session);
        this.session = session;
        try
        {
            Future<Void> fut;

            fut = session.getRemote().sendStringByFuture("hello websocket server ....");
            fut.get(2,TimeUnit.SECONDS); // wait for send to complete.


        }
        catch (Throwable t)
        {
            t.printStackTrace();
        }
    }

    @OnWebSocketMessage
    public void onMessage(String msg)
    {
        System.out.printf("Got msg: %s%n",msg);
        System.out.println("----------------- \r\n \r\n\r\n ");
    }


}

```

### 后续自己封装业务


