# http各版本，websocket, tcpIp三次握手


### http1.0

#### HTTP 1.0规定浏览器与服务器只保持短暂的连接，浏览器的每次请求都需要与服务器建立一个TCP连接，服务器完成请求处理后立即断开TCP连接，服务器不跟踪每个客户也不记录过去的请求。

#### 访问一个包含有许多图像的网页文件的整个过程包含了多次请求和响应，每次请求和响应都需要建立一个单独的连接，每次连接只是传输一个文档和图像，上一次和下一次请求完全分离。即使图像文件都很小，但是客户端和服务器端每次建立和关闭连接却是一个相对比较费时的过程，并且会严重影响客户机和服务器的性 能。当一个网页文件中包含Applet，JavaScript文件，CSS文件等内容时，也会出现类似上述的情况。

```
每次请求都重新建立 tcp 连接，完成三次握手，性能浪费
```
#### HTTP 1.0不支持Host请求头字段


### http1.1

#### HTTP 1.1支持持久连接，在一个TCP连接上可以传送多个HTTP请求和响应，减少了建立和关闭连接的消耗和延迟。

#### HTTP 1.1还允许客户端不用等待上一次请求结果返回，就可以发出下一次请求，但服务器端必须按照接收到客户端请求的先后顺序依次回送响应结果，以保证客户端能够区分出每次请求的响应内容，这样也显著地减少了整个下载过程所需要的时间。

![](http://p.blog.csdn.net/images/p_blog_csdn_net/elifefly/EntryImages/20090306/2.jpg)

```
一个包含有许多图像的网页文件的多个请求和应答可以在一个连接中传输，但每个单独的网页文件的请求和应答仍然需要使用各自的连接。
```


#### 请求头
* Host请求头
* Connection请求头 :

    ```
        Keep-Alive : 客户端通知服务器返回本次请求结果后保持连接;
        close : 客户端通知服务器返回本次请求结果后关闭连接;
    ```
* 身份认证、状态管理和Cache缓存等机制相关的请求头和响应头


#### http1.1 浏览器在同一时间，针对同一个域名下的请求有一定数量的限制，超过限制数目的请求会被阻塞
#### http1.1 所以很多网站有多个 静态cdn域名，增加下载限制

### http2.0
#### http2.0的多路复用技术，允许单一的http2.0

### WebSocket
#### 出于兼容性的考虑，WS 的握手使用 HTTP 来实现，客户端的握手消息就是一个「普通的，带有 Upgrade 头的，HTTP Request 消息」。所以这一个小节到内容大部分都来自于 RFC2616，这里只是它的一种应用形式，下面是 RFC6455 文档中给出的一个客户端握手消息示例：

```
    GET /chat HTTP/1.1            //1
    Host: server.example.com   //2
    Upgrade: websocket            //升级为websocket
    Connection: Upgrade            //升级连接
    Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==            //5
    Origin: http://example.com            //6
    Sec-WebSocket-Protocol: chat, superchat            //7
    Sec-WebSocket-Version: 13            //8
```

#### 如果服务器接受了这个请求，可能会发送如下这样的返回信息，这是一个标准的 HTTP 的 Response 消息。101 表示服务器收到了客户端切换协议的请求，并且同意切换到此协议。RFC2616 规定只有 HTTP1.1 及 HHTTP1.1 以上版本的时候才能同意切换。

```
    HTTP/1.1 101 Switching Protocols //1
    Upgrade: websocket. //2
    Connection: Upgrade. //3
    Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=  //4
    Sec-WebSocket-Protocol: chat. //5
```
#### ws 协议默认使用 80 端口，wss 协议默认使用 443 端口

#### 客户端和服务端都能在任意时候发送数据，(不管是从客户端到服务端还是相反) 

### 三次握手

