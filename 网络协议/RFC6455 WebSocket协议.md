# RFC6455 WebSocket协议


##### 翻译自：
[RFC6455 WebSocket协议](https://tools.ietf.org/html/rfc6455)

### 简介

历史上地(从以前开始)创建一个可以使客户端和服务端进行双向通信需要利用HTTP请求轮询服务器才能获取更新的信息

#### 这可能导致一些列的问题
1. 服务器被被强制使用底层的TCP连接来维护每个客户端，一个连接用来发送信息到客户端，新的连接用于到达的新消息
2. HTTP请求负担太重了，每个客户端的请求都包含了HTTP的请求头
3. 客户端需要维护从发送的请求与返回的答复的映射关系

### WebSocket 双向通信协议来自于浏览器网页

WebSocket支持HTTP的默认端口 80 和 443 支持HTTP代理服务器，它的设计并不局限于HTTP请求，未来的握手方式可能会用一个专有的端口，但是不会去重新设计一个完整的协议

#### WebSocket协议消息格式并不完全匹配标准的HTTP协议

WebSocket 协议分为两部分
1.握手
2.数据传输

#### 客户端握手请求协议：
客户端握手协议

```
        GET /chat HTTP/1.1
        Host: server.example.com
        Upgrade: websocket
        Connection: Upgrade
        Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
        Origin: http://example.com
        Sec-WebSocket-Protocol: chat, superchat
        Sec-WebSocket-Version: 13

```

Request-URI 如果存在URI 时，会存在于 Sec-WebSocket-Protocol 子协议中；
为了保证服务端收到客户端发送的握手消息 Sec-WebSocket-Key ， 服务端通过一系列加密编解码运算返回 Sec-WebSocket-Accept ；
Origin 保护未授权的跨域请求，防止攻击；

#### 服务端应答报文

服务端返回协议的报文：

```

        HTTP/1.1 101 Switching Protocols
        Upgrade: websocket
        Connection: Upgrade
        Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
        Sec-WebSocket-Protocol: chat


```



### 关闭连接 （握手）

客户端/服务端都可以发送Close Frame (包含具体控制序列的消息) 来开始关闭连接的握手；
通过发送 Close Frame 和等待 Close Frame 的返回为了避免数据的不必要的丢失，在一些平台上 socket 关闭接受队列也同时关闭，导致另一端正在发送的数据读取失败；
关闭握手类似于 TCP (FIN/ACK) 的四次挥手关闭


### 设计哲学

WebSocket 被设计成一种有最小帧数据的规则，这种规则使协议基于Frame 而不是基于流，可以使用 unicode 文本 或者 二级制帧数据；

#### 从概念上来说：WebSocket 协议是TCP传输层上的通信协议

WebSocket 协议握手消息只是比 HTTP请求多一些 SEC 开头的请求头

### WebSocket 与 TCP 和 HTTP 请求的区别

1. 独立的基于TCP 的协议，跟HTTP请求的区别也只是使用了HTTP请求进行了握手，升级请求



### WebSocket URI 

```
ws-uri : ws://host : port / path ? query 

wss-uri : wss://host : port / path ? query 


```


支持多版本的 WebSocket 
Sec-WebSokcet-Version 头部

客户端发送协议版本号，服务端支持版本号则接受这个版本协议进行处理，若不支持，服务器端返回支持版本(400) ，客户端重新使用了新版本协议

### 数据帧

#### 控制帧数据
通过 opcodes 进行区分

```
0x8 Close 
0x9 Ping 
0xA Pong 

0xB ~ 0xF 保留的状态码

```

#### Close 
opcode 0x8 
Close Frame 可以包含应用数据的消息体，也可以设置关闭原因 (reason)，可能是UTF-8编码的数据，也可以是2字节的整型数据用来表示状态值

**客户端/服务端发完 Close Frame 的数据之后，不应该在发送任何的数据**

如果后端接受 Close Frame 且之前未发送 Close Frame ，那后端必须再发一个 Close Frame 作为回应，发送返回值后，后端将改变一个状态值 (readsytate = 2 Closing)  Close Frame 可能会延迟到达，因为目前发送消息队列中可能还存在发送消息；

**经过了一接一发的 Close Frame 后端开始进行 TCP 关闭握手**

服务端必须关闭 TCP 连接，客户端应等服务端关闭连接，但可能在接受和发送完 Close Frame 的任何时候关闭连接，前提是在可接受的时间段内未接收到服务端发送的 TCP close 的消息；

#### PING 

opcode 0x9 

Ping Frame 可能包含应用数据，**一收到Ping Frame 的一段必须发送 Pong Frame 作为返回** (可以用来进行心跳),除非已经收到了Close Frame ，Ping 消息，可以在建立连接成功的任何时候进行发送

Ping 消息可以作为保活方式也可以用来确认对端是否存活


#### PONG 

opcode 0xA

Pong Frame 用来回复Ping Frame 可以作为心跳回复， 对于接受到 Pong 消息的端，不需要进行回复

### Data Frames 数据帧 (non-control fram)

同样是用 opcode 来区分

目前定义的 data Frame 包括

```
0x1  Text 
0x2 二进制数据 Binary 
Text 是 UTF-8 编码数据
Binary 二进制数据

```


### Websocket API

[WSAPI](https://www.w3.org/TR/websockets/)

IDL 接口描述

```
[Constructor(DOMString url, optional (DOMString or DOMString[]) protocols)]
interface WebSocket : EventTarget {
  readonly attribute DOMString url;

  // ready state
  const unsigned short CONNECTING = 0;
  const unsigned short OPEN = 1;
  const unsigned short CLOSING = 2;
  const unsigned short CLOSED = 3;
  readonly attribute unsigned short readyState;
  readonly attribute unsigned long bufferedAmount;

  // networking
           attribute EventHandler onopen;
           attribute EventHandler onerror;
           attribute EventHandler onclose;
  readonly attribute DOMString extensions;
  readonly attribute DOMString protocol;
  void close([Clamp] optional unsigned short code, optional DOMString reason);

  // messaging
           attribute EventHandler onmessage;
           attribute DOMString binaryType;
  void send(DOMString data);
  void send(Blob data);
  void send(ArrayBuffer data);
  void send(ArrayBufferView data);
};

```

通过页面 js 构造函数

var ws = new WebSokcet (url , protocols );


则websocket 连接创建成功 readystate = 1 ;
bufferedAmount 返回当前在发送队列中的数据的字节数

send 方法发送前可以判断一下 bufferdAmount 的数据为0 则进行发送

当 readystate 为 0 时 会抛出异常 INVALID_STATE_ERR 的异常

客户端必须执行精确步骤当webSocket 对象被创建， binaryType 属性被设置成 blob ,接受的数据类型必须与设置的数据类型一直，否则返回 SYNTAL_ERR 

#### 当 websocket 连接建立，客户端任务队列执行下列的步骤

1. readystate -> open(1)
2. extensions 设置
3. protocol 属性设置
4. 参加梅河口am瑟吉欧 set-cookie 属性的处理
5. open事件的触发 (onopen)

#### 当消息到达时，客户端任务队列时：

1. readystate 不是 open （1） 或者 closing (2) 则执行完毕；
2. 触发message 事件 （onmessage）
3. 初始化 origin 属性
4. 解析数据
5. 调度 websocket 对象的事件


#### 关闭连接
1. readystate -> closed （3）
2. 触发error 事件 (onerror)
3. 创建 close 事件，触发close 事件 (onclose) ，并带着 事件code 和 reason ，属性wasClean 代表连接是否完全关闭














