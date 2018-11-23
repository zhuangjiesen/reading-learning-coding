# Netty学习

### AIO既然这么好，为什么Netty不用
#### 官方回答：
https://github.com/netty/netty/issues/2515

#### 总结：

```
·Not faster than NIO (epoll) on unix systems (which is true)
·There is no daragram suppport
·Unnecessary threading model (too much abstraction without usage)

1. 在unix 平台上 aio 并不比 nio 快
2. 没有数据报支持
3. 没有必要的线程模型

```


netty 使用注意

1. 在 ChannelHandler 中不宜使用耗时业务，否则会阻塞 io 操作
2. 在 ChannelHandler 中日志打印要注意，日志大量刷盘也会引起阻塞，
3. close_wait 会维持至少2个小时的时间，客户端大量的 close_wait 会导致句柄用尽报错
4. 线程数不能太大，否则频繁切换线程导致cpu 消耗
5. 当服务端处理海量客户端长连接的时候，不要在NioEventLoop中执行自定义Task，或者非心跳类的定时任务。

6. 常用的TCP参数，例如TCP层面的接收和发送缓冲区大小设置，在Netty中分别对应ChannelOption的SO_SNDBUF和SO_RCVBUF，需要根据推送消息的大小，合理设置，对于海量长连接，通常32K
7. 心跳机制，时间和包大小



