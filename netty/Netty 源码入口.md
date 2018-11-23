# Netty 源码入口

```
看源码重要的是找到入口，找到源码的起点，看起来效率会更高
```
#### 众所周知，netty是基于Java Nio原生封装的一款网络框架
```
所以它肯定是基于 ServerSocketChannel 、 Selector 、 Buffer 这些封装的

所以、问题是：
1.找到 ServerSocketChannel
2.找到 Selector 
3.找到 ServerSocketChannel.register()方法的调用，将 socketChannel注册到Selector 上
```

#### 先不说Netty里复杂的Boostrap、Channel(netty自定义的)、Unsafe 

### Server 示例代码

```
// TODO Auto-generated method stub

                EventLoopGroup boss = new NioEventLoopGroup();
                EventLoopGroup worker = new NioEventLoopGroup(workerCount);
                try {
                    ServerBootstrap bootstrap = new ServerBootstrap();
                    bootstrap.group(boss, worker);
                    bootstrap.channel(NioServerSocketChannel.class);
                    bootstrap.option(ChannelOption.SO_BACKLOG, backlog); //连接数
                    bootstrap.option(ChannelOption.TCP_NODELAY, tcpNodelay);  //不延迟，消息立即发送
//		            bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000);  //超时时间
                    bootstrap.childOption(ChannelOption.SO_KEEPALIVE, keepalive); //长连接
                    bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel)
                                throws Exception {
                            ChannelPipeline p = socketChannel.pipeline();

                            //HttpServerCodec: 针对http协议进行编解码
                            p.addLast("http-codec", new HttpServerCodec());
                            /**
                             * 作用是将一个Http的消息组装成一个完成的HttpRequest或者HttpResponse，那么具体的是什么
                             * 取决于是请求还是响应, 该Handler必须放在HttpServerCodec后的后面
                             */
                            p.addLast("aggregator", new HttpObjectAggregator(65536));
                            //ChunkedWriteHandler分块写处理，文件过大会将内存撑爆
                            p.addLast("http-chunked", new ChunkedWriteHandler());
                            //请求处理
                            p.addLast("inboundHandler", webSocketChannelHandlerFactory.newWSInboundHandler());
                            //关闭处理
                            p.addLast("outboundHandler", webSocketChannelHandlerFactory.newWSOutboundHandler());

                        }
                    });


                    ChannelFuture f = bootstrap.bind(port).sync();


                    if (f.isSuccess()) {
                    }
                    f.channel().closeFuture().sync();
                } catch (Exception e) {
                    LogUtils.logError(this, e );
                } finally {
                    boss.shutdownGracefully();
		            worker.shutdownGracefully();
                }

```

### Tips:
```
ServerSocketChannel 的bind 和 register 方法是不用分先后的
所以netty是先register 然后再 bind

```

### 直接写重点代码段
```
从 bootstrap.bind() 开始
-> AbstractBootstrap.bind()
-> AbstractBootstrap.doBind()
    这里有2段重点
    // 这里初始化的netty 自定义的channel(AbstractChannel) 类，绑定线程池，绑定Selector、ChannelPipeline等等
    1.initAndRegister();
    // 绑定端口号开启服务监听
    2.doBind0();

```

#### 接着先讲 1.initAndRegister();
```

    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            //创建服务时会需要制定 Channnel类型 这里制定的是 NioServerSocketChannel.class 
            channel = channelFactory.newChannel();
            //初始化配置参数设置channel 属性，绑定 ChannelPipeline 、 EventLoopGroup
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        //register 绑定Selector 
        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regFuture;
    }

```

#### 接着是 config().group().register(channel) 

```
-> EmbeddedEventLoop.register();
-> promise.channel().unsafe().register(this, promise);

-> AbstractChannel.AbstractUnsafe.register()
-> AbstractChannel.AbstractUnsafe.register0()
-> AbstractNioChannel.doRegister()

重点就来了

-> selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);

这里讲ServerSocketChannel 注册到Selector 上，
并且把 AbstractNioChannel 实例(它本身)，注册到Selector ，
所以SocketChannel 的io事件触发的时候，直接取attachment，
然后拿到 SocketChannel,然后获取之前绑定的 eventLoop、channelPipeline等，进行Netty操作

```



