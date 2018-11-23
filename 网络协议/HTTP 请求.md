# HTTP 请求
### 特点：
1. **支持Client/Server 模式；**
2. **简单**：
    客户端向服务器请求服务时，只需要制定服务URL ，携带必要的请求参数或消息体；
3. **灵活** ：允许传输任意类型的数据对象，传输的内容由 HTTP请求头中的 Content-Type 加以标记
4. **无状态** : HTTP 协议是无状态协议，无状态指的是事务处理没有记忆能力；




### URL 

```

http://host[":"port][abs_path]

指定域名(host)，端口号(port 若无，默认是80端口) 请求的 uri路径（abs_path） 

```

### 请求消息（HttpRequest）

#### http请求分三部分：
1. HTTP 请求头
2. HTTP 消息头
3. HTTP 请求正文

##### 请求行以一个方法开头，以空格分开，后面跟着请求的 URI 和协议的版本，格式为:
```

Method Request-URI HTTP-Version CRLF

```
#### 解释：
1. Method 表示请求方法，
2. Request-URI 是一个统一资源定位符
3. HTTP-Version 表示请求的 HTTP  协议版本 
4. CRLF 表示回车和换行

请求方法有很多：GET,POST ,HEAD ,PUT ,DELETE ,TRACE ,CONNECT ,OPTIONS ;

常用的方法是:
1. **GET请求** ： 请求获取 Request-URI 所标识的资源；
2.**POST请求**： 在Request-URI 所标识的资源后附加新的提交数据；



#### get请求抓包数据：
![httpget](media/14929259206970/httpget.png)

#### GET / POST 区别

1. GET 一般用于获取/查询资源信息，POST 一般用于更新资源信息
2. 根据 HTTP 规范，GET用于信息获取，而且应该是安全的和幂等的；POST 则表示可能会改变服务器上的资源的请求；
3. GET 提交，请求数据会附在 URL 之后，就是把数据放置在请求行中，以 "?" 分隔URI 与传输数据，多个参数用 "&" 连接，POST 请求把数据放置在 HTTP 消息的包体中，数据不会在地址栏中显示出来；
4. 传输数据的大小不同，特定浏览器和服务器对于 URL  长度有限制，IE 对 URL 长度的限制是2083 字节 ，因此 GET 携带的参数长度会受到浏览器的限制，POST 不是通过 URL 传值，理论上数据长度不会受限；
5. 安全性， POST 安全性要比 GET 的安全性高；
6. GET 请求能被缓存


#### 最常见的请求头：
1. **Accept**：浏览器可接受的MIME类型。
2.     **Accept - Charset**：浏览器可接受的字符集。
3.     **Accept - Encoding**：浏览器能够进行解码的数据编码方式，比如gzip。Servlet能够向支持gzip的浏览器返回经gzip编码的HTML页面。许多情形下这可以减少5到10倍的下载时间。
4.     **Accept - Language**：浏览器所希望的语言种类，当服务器能够提供一种以上的语言版本时要用到。
5.     **Authorization**：授权信息，通常出现在对服务器发送的WWW - Authenticate头的应答中。
6.     **Connection**：表示是否需要持久连接。如果Servlet看到这里的值为“Keep - Alive”，或者看到请求使用的是HTTP 1.1（HTTP 1.1默认进行持久连接），它就可以利用持久连接的优点，当页面包含多个元素时（例如Applet，图片），显著地减少下载所需要的时间。要实现这一点，Servlet需要在应答中发送一个Content - Length头，最简单的实现方法是：先把内容写入ByteArrayOutputStream，然后在正式写出内容之前计算它的大小。
7.     **Content - Length**：表示请求消息正文的长度。
8.     **Cookie**：这是最重要的请求头信息之一，参见后面《Cookie处理》一章中的讨论。
9.     **From**：请求发送者的email地址，由一些特殊的Web客户程序使用，浏览器不会用到它。
1.     Host：初始URL中的主机和端口。
2.     **If - Modified - Since**：只有当所请求的内容在指定的日期之后又经过修改才返回它，否则返回304“Not Modified”应答。
3.     **Pragma**：指定“no - cache”值表示服务器必须返回一个刷新后的文档，即使它是代理服务器而且已经有了页面的本地拷贝。
4.     **Referer**：包含一个URL，用户从该URL代表的页面出发访问当前请求的页面。
5.     **User - Agent**：浏览器类型，如果Servlet返回的内容与浏览器类型有关则该值非常有用。
6.     **UA - Pixels，UA - Color，UA - OS，UA - CPU**：由某些版本的IE浏览器所发送的非标准的请求头，表示屏幕大小、颜色深度、操作系统和CPU类型。

#### 消息响应类型
1. **200 (OK)**: 找到了该资源，并且一切正常。
2. **304 (NOT MODIFIED)**: 该资源在上次请求之后没有任何修改。这通常用于浏览器的缓存机制。
3. **401 (UNAUTHORIZED)**: 客户端无权访问该资源。这通常会使得浏览器要求用户输入用户名和密码，以登录到服务器。
4. **403 (FORBIDDEN)**: 客户端未能获得授权。这通常是在401之后输入了不正确的用户名或密码。
5. **404 (NOT FOUND)**: 在指定的位置不存在所申请的资源。
6. **5xx** ：服务器端错误，服务器未能处理请求


    
####     HTTP响应头

　　响应头向客户端提供一些额外信息，比如谁在发送响应、响应者的功能，甚至与响应相关的一些特殊指令。这些头部有助于客户端处理响应，并在将来发起更好的请求。响应头域包含Age、Location、Proxy-Authenticate、Public、Retry- After、Server、Vary、Warning、WWW-Authenticate。对响应头域的扩展要求通讯双方都支持，如果存在不支持的响应头域，一般将会作为实体头域处理。

1. **Age** ：当代理服务器用自己缓存的实体去响应请求时，用该头部表明该实体从产生到现在经过多长时间了。

2. **Server** ：WEB 服务器表明自己是什么软件及版本等信息。例如：Server：Apache/2.0.61 (Unix)

3. **Accept-Ranges** ：WEB服务器表明自己是否接受获取其某个实体的一部分（比如文件的一部分）的请求。bytes：表示接受，none：表示不接受。

4. **Vary** ：WEB服务器用该头部的内容告诉 Cache 服务器，在什么条件下才能用本响应所返回的对象响应后续的请求。假如源WEB服务器在接到第一个请求消息时，其响应消息的头部为：Content-Encoding: gzip; Vary: Content-Encoding，那么Cache服务器会分析后续请求消息的头部，检查其Accept-Encoding，是否跟先前响应的Vary头部值一致，即是否使用相同的内容编码方法，这样就可以防止Cache服务器用自己Cache 里面压缩后的实体响应给不具备解压能力的浏览器。例如：Vary：Accept-Encoding。

#### 　HTTP实体头

##### 　　实体头部提供了有关实体及其内容的大量信息，从有关对象类型的信息，到能够对资源使用的各种有效的请求方法。总之，实体头部可以告知接收者它在对什么进行处理。请求消息和响应消息都可以包含实体信息，实体信息一般由实体头域和实体组成。实体头域包含关于实体的原信息，实体头包括信息性头部Allow、Location，内容头部Content-Base、Content-Encoding、Content-Language、Content-Length、Content-Location、Content-MD5、Content-Range、Content-Type，缓存头部Etag、Expires、Last-Modified、extension-header。

1. **Allow** ：服务器支持哪些请求方法（如GET、POST等）。

2. **Location** ：表示客户应当到哪里去提取文档，用于将接收端定位到资源的位置（URL）上。Location通常不是直接设置的，而是通过HttpServletResponse的sendRedirect方法，该方法同时设置状态代码为302。

3. **Content-Base** ：解析主体中的相对URL时使用的基础URL。

4. **Content-Encoding** ：WEB服务器表明自己使用了什么压缩方法（gzip，deflate）压缩响应中的对象。例如：Content-Encoding：gzip

5. **Content-Language** ：WEB 服务器告诉浏览器理解主体时最适宜使用的自然语言。

6. **Content-Length** ：WEB服务器告诉浏览器自己响应的对象的长度或尺寸，例如：Content-Length: 26012

7. **Content-Location** ：资源实际所处的位置。

8. **Content-MD5** ：主体的MD5校验和。

9. **Content-Range** ：实体头用于指定整个实体中的一部分的插入位置，他也指示了整个实体的长度。在服务器向客户返回一个部分响应，它必须描述响应覆盖的范围和整个实体长度。一般格式： Content-Range:bytes-unitSPfirst-byte-pos-last-byte-pos/entity-legth。例如，传送头500个字节次字段的形式：Content-Range:bytes0- 499/1234如果一个http消息包含此节（例如，对范围请求的响应或对一系列范围的重叠请求），Content-Range表示传送的范围，Content-Length表示实际传送的字节数。

1. **Content-Type** ：WEB 服务器告诉浏览器自己响应的对象的类型。例如：Content-Type：application/xml

2. **Etag** ：就是一个对象（比如URL）的标志值，就一个对象而言，比如一个html文件，如果被修改了，其Etag也会别修改，所以，ETag的作用跟Last-Modified的作用差不多，主要供WEB服务器判断一个对象是否改变了。比如前一次请求某个html文件时，获得了其 ETag，当这次又请求这个文件时，浏览器就会把先前获得ETag值发送给WEB服务器，然后WEB服务器会把这个ETag跟该文件的当前ETag进行对比，然后就知道这个文件有没有改变了。

3. **Expires** ：WEB服务器表明该实体将在什么时候过期，对于过期了的对象，只有在跟WEB服务器验证了其有效性后，才能用来响应客户请求。是 HTTP/1.0 的头部。例如：Expires：Sat, 23 May 2009 10:02:12 GMT

4. **Last-Modified** ：WEB服务器认为对象的最后修改时间，比如文件的最后修改时间，动态页面的最后产生时间等等。例如：Last-Modified：Tue, 06 May 2008 02:42:43 GMT
    



