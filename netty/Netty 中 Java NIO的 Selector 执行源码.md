# Netty 中 Java NIO的 Selector 执行源码

# 启动代码
```
// boss线程池 - 用来处理accept
EventLoopGroup boss = new NioEventLoopGroup();
// worker 工作线程 read/write事件以及read完触发的业务层的事件执行(推荐使用线程池)
EventLoopGroup worker = new NioEventLoopGroup(workerCount);
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(boss, worker);
...省略配置代码、pipeline配置
//绑定端口，服务启动
ChannelFuture f = bootstrap.bind(port).sync();

```

#### 疑问点：
```
1.NioEventLoop 与 NioEventLoopGroup 关系
2.io事件怎么跟 NioEventLoop关联起来的
3.SocketChannel(或ServerSocketChannel)register() 触发时，SelectionKey指定的是0，那么对应的SocketChannel监听事件时如何指定的？
4.Unsafe 、 Channel之间的关系？
3.accept 和 读写处理的关系？
4.任务和io事件的处理逻辑
5.线程模型
```


#### NioEventLoop - 继承于SingleThreadEventLoop 类似于一个 Selector 的处理线程
```
Selector 原则上只是个监视器对象，因为它带有阻塞线程的特性，所以通常使用它的时候会通过一个线程来配合使用
所以 NioEventLoop 代码里会有run() 方法，并且可以看到 Selector 的操作

```

#### NioEventLoopGroup - 继承于MultithreadEventLoopGroup 类似于NioEventLoop的集合(也就是线程池)
```
在 EventLoopGroup.group(boss, worker) 方法中绑定parent线程组(boss) 和 child线程组(worker)
用来区分io事件的处理
boss线程只用来处理accept
worker是用来执行读写和系统任务task和定时任务task (SingleThreadEventExecutor.taskQueue 任务队列)

NioEventLoop.run() 方法是个复合的方法，也就是所有的accept事件，读写，任务处理，调用统计，selector的rebuild都是在这里，
做什么事取决于selector 被注册了什么事件，和任务队列的情况

accept事件只会在 boss 线程池(单线程)中执行

DefaultChannelPipeline.
ServerBootstrap.ServerBootstrapAcceptor.channelRead()方法
ServerBootstrap.init()的时候会将ServerBootstrap.ServerBootstrapAcceptor添加到全局的ChannelPipline调用链中
所以后面的channelPipline事件的触发，都会在 worker 工作线程中执行
```

### Netty中的 ServerSocketChannel 和 SocketChannel
```
Netty中的 ServerSocketChannel 对应 NioServerSocketChannel (在配置服务的时候可以指定，在ServerBootstrap 时候通过工厂模式创建的channel对象)
SocketChannel 对应 NioSocketChannel 
NioServerSocketChannel 与 SocketChannel 的父类都是 AbstractNioChannel (NioServerSocketChannel的父类还有一层 AbstractNioMessageChannel )

```

#### 看一下 AbstractNioChannel 的构造函数
```

    /**
     * Create a new instance
     *
     * @param parent            the parent {@link Channel} by which this instance was created. May be {@code null}
     * @param ch                the underlying {@link SelectableChannel} on which it operates
     * @param readInterestOp    the ops to set to receive data from the {@link SelectableChannel}
     * 这里的重点是 readInterestOp ，就是 SelectionKey 也就是 SocketChannel用来 register() 时的事件监听
     * 
     */
    protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
        super(parent);
        this.ch = ch;
        this.readInterestOp = readInterestOp;
        try {
            ch.configureBlocking(false);
        } catch (IOException e) {
            try {
                ch.close();
            } catch (IOException e2) {
                if (logger.isWarnEnabled()) {
                    logger.warn(
                            "Failed to close a partially initialized socket.", e2);
                }
            }

            throw new ChannelException("Failed to enter non-blocking mode.", e);
        }
    }

```

#### 注册的事件不同
```
NioServerSocketChannel 的构造函数是ACCEP事件
SocketChannel 的构造函数设置的则是 READ事件
因为在 AbstractNioChannel.doRegister() 方法中(就是SocketChannel 的register()绑定selector监听的方法 )
监听的 SelecttionKey 是0
```

#### AbstractNioChannel.doRegister() 方法

```


    @Override
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
            	/*
            		这里的第二个参数， SelecttionKey 设置成0 令人难以理解
            		之后发现这个是个父类，这是个通用方法，设置监听为 0 ，表示先绑定Selector;
            		后面再通过 doBeginRead() 设定自己的监听事件 (ACCEPT 或 READ);


            	*/

                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    // Force the Selector to select now as the "canceled" SelectionKey may still be
                    // cached and not removed because no Select.select(..) operation was called yet.
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    // We forced a select operation on the selector before but the SelectionKey is still cached
                    // for whatever reason. JDK bug ?
                    throw e;
                }
            }
        }
    }

```
#### 始终找不到 doBeginRead() 的调用
#### 虽然知道它是bind()端口前回去调用 doBeginRead() 方法，将监听事件修改，但是想知道它的调用时机
```
调用链(用调用链路图可能会晕，其实一个一个点进去就能发现，或者每个以下代码行上打上断点就能看到了)
- ServerBootstrap.bind()
- ServerBootstrap.doBind()
- ServerBootstrap.doBind0()
- channel.bind() -> AbstractChannel.bind()
- AbstractChannelHandlerContext.bind()
- next.invokeBind() -> HeadContext.bind()
- unsafe.bind() -> AbstractChannel 里的 AbstractUnsafe.bind() 
- pipeline.fireChannelActive() -> DefaultChannelPipeline.HeadContext.channelActive() 
- DefaultChannelPipeline.HeadContext.read() -> AbstractNioChannel.doBeginRead() 最后将 SelectionKey 设置成对应的 ACCEPT 或 READ 

```

#### AbstractUnsafe.bind() 代码

```

        @Override
        public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
            assertEventLoop();

            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }

            // See: https://github.com/netty/netty/issues/576
            if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
                localAddress instanceof InetSocketAddress &&
                !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
                !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
                // Warn a user about the fact that a non-root user can't receive a
                // broadcast packet on *nix if the socket is bound on non-wildcard address.
                logger.warn(
                        "A non-root user can't receive a broadcast packet if the socket " +
                        "is not bound to a wildcard address; binding to a non-wildcard " +
                        "address (" + localAddress + ") anyway as requested.");
            }

            boolean wasActive = isActive();
            try {
            	//这里是给ServerSocketChannel 对象绑定端口和backlog 值
                doBind(localAddress);
            } catch (Throwable t) {
                safeSetFailure(promise, t);
                closeIfClosed();
                return;
            }
            //重点是下面
            if (!wasActive && isActive()) {
                invokeLater(new Runnable() {
                    @Override
                    public void run() {
                    	//触发active事件，这里就是触发 doBeginRead() 方法的源头
                        pipeline.fireChannelActive();
                    }
                });
            }

            safeSetSuccess(promise);
        }

```


#### pipeline.fireChannelActive() 的源头就是 DefaultChannelPipeline.HeadContext.channelActive() 方法

```

        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelActive();
            /* 
            这里触发的 doBeginRead() 方法,最终通过pipeline 还是调用自己的 HeadContext.read()方法，触发 doBeginRead()
            */
            readIfIsAutoRead();
        }


```


#### 结论
```
1.NioEventLoop 与 NioEventLoopGroup 关系
	答： NioEventLoop(继承于 SingleThreadEventLoop ) 类似一个 Selector和任务操作的线程，具体操作看run() 方法
		NioEventLoopGroup(继承于 MultithreadEventLoopGroup ) 是个 NioEventLoop 的集合 children数组的属性；
		NioEventLoop 只需要轮询任务队列和Selector 监听的事件就行

2.io事件怎么跟 NioEventLoop 关联起来的
	答： NioEventLoop 在 run() 方法中轮询 Selector 监听的 io 事件，获取绑定Selector 的 attachment对象(ACCEPT事件是 NioServerSocketChannel 、READ事件是 NioSocketChannel ) ，接着就是调用pipeline间接调用Unsafe读写 Channel 消息
3.SocketChannel(或ServerSocketChannel)register() 触发时，SelectionKey指定的是0，那么对应的SocketChannel监听事件时如何指定的？
	答：AbstractBoostrap.initAndRegister() 方法只是将 ServerSocketChannel(Java) register() 上 Selector ，
	注册事件的SelectionKey = 0 (SelectionKey 没有0 ，等于没有绑定事件)
	之后是doBind0() 后，在绑定端口前，触发pipeline(DefaultChannelPipeline.HeadContext)的active事件，
	触发 NioServerSocketChannel.AbstractNioMessageChannel 的 doBeginRead() 方法，将SelectionKey 改成ACCEPT (selectionKey.interestOps()方法)

4.Unsafe 、 Channel之间的关系？
	答： Unsafe 是个io操作的工具类，用来做 读、写、连接、注册、绑定端口、获取配置信息等；
		Channel 是Netty自己定义的Channel ，跟JavaNio 中的Channel 的概念相似，但不是一个东西，
		在register() 时，通过attachment绑定到Selector上，每次的io事件触发，就都可以获取对应的Channel (ServerSocketChannel 或 SocketChannel)
3.ACCEPT 和 读写处理的关系？
	答：服务端通过监听ACCEPT事件，获取连接的客户端SocketChannel，在把这些SocketChannel 注册到监听 读(写) 的Selector上
4.任务和io事件的处理逻辑
	答：逻辑复杂，单独讲 NioEventLoop.run() 中逻辑
5.线程模型
	答：REACTOR模型，主从多线程模型，见《Netty权威指南》

```


