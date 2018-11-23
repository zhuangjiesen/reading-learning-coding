# Java中的 nio 与 io理解
```
网上很多帖子都在说io 与 nio的区别：
1.io是面向流、nio是面向缓冲区
```

### 从具体的Java代码入手，概念太抽象了

### 在一般的 Java IO 操作中, 我们以流式的方式顺序地从一个 Stream 中读取一个或多个字节, 因此我们也就不能随意改变读取指针的位置.
#### 读取数据伪代码：
```
        Socket client = new Socket();
        //省略连接和配置部分代码....
        byte[] buf = new byte[1024];
        //这个 read() 方法调用是阻塞的，只有数据读就绪时候才能通过
        client.getInputStream().read(buf);
        //阻塞中....

```
### 基于 Buffer 就不一样，首先需要从 Channel 中读取数据到 Buffer 中, 当 Buffer 中有数据后, 我们就可以对这些数据进行操作了. 不像 IO 那样是顺序操作, NIO 中我们可以随意地读取任意位置的数据.
#### 读取数据伪代码：
```		

		//省略服务端配置和启动代码...
        //获取客户端连接
        SocketChannel socketChannel = serverSocketChannel.accept();
        //这里测试 socketChannel.read 操作是不阻塞的，直接返回
        socketChannel.configureBlocking(false);
        String message = null;
        //readChannelData() 的方法是非阻塞的，前提是configureBlocking() 设置成非阻塞
        message = NioServer.readChannelData(socketChannel);
        System.out.println("message1 : " + message);
        message = NioServer.readChannelData(socketChannel);
        System.out.println("message2 : " + message);
        message = NioServer.readChannelData(socketChannel);
        System.out.println("message3 : " + message);
        message = NioServer.readChannelData(socketChannel);
        System.out.println("message4 : " + message);




    /**
     * 读取数据操作
     *
     * @author zhuangjiesen
     * @date 2018/10/11 下午5:45
     * @param
     * @return
     */
    public static String readChannelData(SocketChannel socketChannel){
        try {
            ByteBuffer buf = ByteBuffer.allocate(1024);
            // configureBlocking() 设置成非阻塞 ，这里读不到数据直接返回
            int bytesRead = socketChannel.read(buf);
            buf.flip();
            byte[] data = new byte[buf.remaining()];
            buf.get(data);
            String message = new String(data);
            return message;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }


```


### 结论(我的理解)：
```
1. 假设不考虑阻塞浪费时间和效率
通过代码层面的设计，完全可以设计一个专门的线程用来等待read()事件，然后设计一个buffer类，将数据读到buffer
应用层，获取到的一样只是buffer，所以我要说的是，nio的面向buffer(缓冲区),是Java层面的api

2. 如果 configureBlocking() 设置成阻塞
那么nio的read() 操作也是一样阻塞的

3.所以区别到底在哪？
假设nio的read() 设置成非阻塞了，一直read() 读不到数据又有什么意思
*** 这时候 ***

Selector出现了！

```

### Selector 

#### 我的理解:
```
1. Selector 不是线程
一般需要开启线程，在线程中调用Selector 的方法进行操作

2. Selector 通过绑定(注册) socketChannel(SelectableChannel.java) 管理自己的Channel事件
通过注册自己想要处理的事件： 一般是先处理Accept(握手连接),然后在开启一组读写Selector线程用来管理读写操作

```

#### 代码demo ：
```
		//服务端启动
		serverSocketChannel = ServerSocketChannel.open();
		//创建Selector
		Selector acceptor = Selector.open();
		serverSocketChannel.configureBlocking(false);
		//注册前 serverSocketChannel 必须设置成非阻塞
		serverSocketChannel.register(acceptor, SelectionKey.OP_ACCEPT);
		//绑定端口
		serverSocketChannel.bind(new InetSocketAddress("127.0.0.1", 38888));
		System.out.println(" server start ....");
		while (true) {
		    int sel = acceptor.select();
		    if (sel > 0) {
		        // accept 事件触发
		        Set<SelectionKey> selectedKeys = acceptor.selectedKeys();
		        Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
		        while(keyIterator.hasNext()) {
		            SelectionKey key = keyIterator.next();
		            //获取完需要清除，不然会循环取出自己的事件
		            keyIterator.remove();

		            // skip if not valid
		            if (!key.isValid()) {
		                continue;
		            }

		            if(key.isAcceptable()) {
		                // a connection was accepted by a ServerSocketChannel.
		                System.out.println(" an accept event reach ....");
		                //处理accept 事件，取出客户端请求的SocketChannel对象，绑定到新的selector中监听读写事件
		                doAccept();
		            }
		        }

		    }

		}


```

### Tips 

```
1. SelectionKey.OP_WRITE 的触发时机
SelectKey注册了写事件，不在合适的时间去除掉，会一直触发写事件，因为写事件是代码触发的
client.register(selector, SelectionKey.OP_WRITE); 或者sk.interestOps(SelectionKey.OP_WRITE)
执行了这以上任一代码都会无限触发写事件，跟读事件不同，一定注意

2. seletor.select() 的时候，需要注册 socketChannel 
在 seletor.select() 情况下
	新注册 socketChannel 可能会阻塞
	如：
	try {
        socketChannel.configureBlocking(false);
        socketChannel.register(seletor , SelectionKey.OP_READ);
    } catch (IOException e) {
        e.printStackTrace();
    }
    所以：register() 操作前需要wakeUp();

```

### SelectionKey的 attachment

```
这玩意是啥为啥拿出来说
1.这个东西是Selector用来管理channel的，但是如果业务上的数据，需要跟channel进行关联的话，怎么办，这时候就需要attachment 

```


### By the Way
```
Netty 中的源码底层是通过原生nio的封装，最底层是把它的核心类 AbstractChannel 放到attachment中，
每次连接的事件触发，其实是直接获取attachment，
拿到 AbstractChannel,然后就操作自己netty上层的一系列代码


源码：AbstractNioChannel.doRegister()
```

### Netty源码入口

```

    @Override
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                //attachment 设置的 this 对象就是 AbstractNioChannel 的实例
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

#### 结论
```
所以attachment 可以用来传递连接(channel) 之间的共享对象
```

### 撸了个完整的nio 服务端，附带一个io的客户端

#### 目的：为了模拟完整的长连接交互

#### 功能说明
```
1.nio server 能够接收消息，并且对于每个消息发送一条ack的报文回去，主要为了模拟 读写Selector 的实现；
2.io客户端，2个线程，1.接受消息线程 2.定时发送消息；
```

#### NioServer.java 

```

/**
 * @param
 * @Author: zhuangjiesen
 * @Description:
 *
 * nio服务端实现
 *
 *
 * @Date: Created in 2018/10/11
 */
public class NioServer {

    private static ReadWriteThread readWriteThread = new ReadWriteThread();

    private static ServerSocketChannel serverSocketChannel = null;

    public static void main(String[] args) throws Exception {
        //开启读写监听线程
        Thread t1 = new Thread(readWriteThread);
        t1.start();

        //开启服务端监听
        ExecutorService serverService = Executors.newSingleThreadExecutor();
        serverService.execute(new Runnable() {
            @Override
            public void run() {

                try {
                    serverSocketChannel = ServerSocketChannel.open();
                    //创建Selector
                    Selector acceptor = Selector.open();
                    serverSocketChannel.configureBlocking(false);
                    //注册前 serverSocketChannel 必须设置成非阻塞
                    serverSocketChannel.register(acceptor, SelectionKey.OP_ACCEPT);
                    serverSocketChannel.bind(new InetSocketAddress("127.0.0.1", 38888));
                    System.out.println(" server start ....");
                    while (true) {
                        int sel = acceptor.select();
                        if (sel > 0) {
                            // accept 事件触发
                            Set<SelectionKey> selectedKeys = acceptor.selectedKeys();
                            Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
                            while(keyIterator.hasNext()) {
                                SelectionKey key = keyIterator.next();
                                //获取完需要清除，不然会循环取出自己的事件
                                keyIterator.remove();

                                // skip if not valid
                                if (!key.isValid()) {
                                    continue;
                                }

                                if(key.isAcceptable()) {
                                    // a connection was accepted by a ServerSocketChannel.
                                    System.out.println(" an accept event reach ....");
                                    //处理accept 事件，取出客户端请求的SocketChannel对象，绑定到新的selector中监听读写事件
                                    doAccept();
                                }
                            }

                        }

                    }

                } catch (Exception e) {
                    e.printStackTrace();
                }


            }
        });





    }


    public static void doAccept() throws Exception{
        //获取客户端连接
        SocketChannel socketChannel = serverSocketChannel.accept();

        //这里测试 socketChannel.read 操作是不阻塞的，直接返回
        /*
        socketChannel.configureBlocking(false);
        String message = null;
        message = NioServer.readChannelData(socketChannel);
        System.out.println("message1 : " + message);
        message = NioServer.readChannelData(socketChannel);
        System.out.println("message2 : " + message);
        message = NioServer.readChannelData(socketChannel);
        System.out.println("message3 : " + message);
        message = NioServer.readChannelData(socketChannel);
        System.out.println("message4 : " + message);


        * */


        //注册读写事件
        registReadWriter(socketChannel);
    }


    public static void registReadWriter(SocketChannel socketChannel) throws Exception{
        System.out.println(" doreading ..... ");
        //select先wakeup才能register
        readWriteThread.wakeUpSelect();
        //注册到读写selector
        readWriteThread.registerSocketChannel(socketChannel);
    }



    /**
     * 读写监听线程
     *
     * @author zhuangjiesen
     * @date 2018/10/11 下午5:27
     * @param
     * @return
     */
    public static class ReadWriteThread implements Runnable {
        /** **/
        private static Selector selector;

        public void registerSocketChannel(SocketChannel socketChannel) {
            try {
                socketChannel.configureBlocking(false);
                socketChannel.register(selector , SelectionKey.OP_READ);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }


        public void wakeUpSelect() {
            try {
                //唤醒selector
                selector.wakeup();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        public ReadWriteThread() {
            try {
                selector = Selector.open();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void run() {
            try {
                while (true) {
                    //调用即阻塞
                    int sel = selector.select();
                    if (sel > 0) {
                        System.out.println("ReadWriteThread 有读写事件产生！");
                        Set<SelectionKey> selectedKeys = selector.selectedKeys();
                        Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
                        while(keyIterator.hasNext()) {
                            SelectionKey key = keyIterator.next();
                            keyIterator.remove();
                            // skip if not valid
                            if (!key.isValid()) {
                                continue;
                            }

                            if (key.isReadable()) {
                                //处理读事件
                                this.doRead(key);
                            } else if(key.isWritable()) {
                                //处理写事件
                                this.doWrite(key);
                            } else {
                                System.out.println("没有事件匹配！");
                            }
                        }
                    }


                }
            } catch (Exception e) {
                e.printStackTrace();
            }

        }


        /**
         * 读操作
         *
         * @author zhuangjiesen
         * @date 2018/10/11 下午2:41
         * @param
         * @return
         */
        private void doRead(SelectionKey key) throws Exception {
            //读取socketChannel 的数据
            SocketChannel socketChannel = (SocketChannel) key.channel();
            String message = NioServer.readChannelData(socketChannel);
            System.out.println("读事件:内容: " + message);
            //将数据存到 SelectionKey 中，在write事件时可以返回
            key.attach(message);
            //触发写事件 OP_WRITE
            key.interestOps(SelectionKey.OP_WRITE);
        }





        /**
         * 发送 读消息的ack
         *
         * @author zhuangjiesen
         * @date 2018/10/11 下午2:41
         * @param
         * @return
         */
        private void doReadAck(SocketChannel socketChannel , String readmsg) throws Exception {
            System.out.println("读事件ack ");
            //拼接消息
            StringBuilder ackSb = new StringBuilder(readmsg);
            ackSb.append(":ack:");
            ackSb.append(System.currentTimeMillis());
            ackSb.append("\r\n");
            //写消息
            ByteBuffer buf = ByteBuffer.allocate(48);
            String response = ackSb.toString();
            System.out.println("写ack : " + response);
            buf.put(response.getBytes());
            buf.flip();
            while(buf.hasRemaining()) {
                socketChannel.write(buf);
            }
        }

        /**
         * 写操作
         *
         * @author zhuangjiesen
         * @date 2018/10/11 下午2:41
         * @param
         * @return
         */
        private void doWrite(SelectionKey key) throws Exception {
            System.out.println("写事件.... ");
            //获取 SelectionKey存的 attachment
            String message = (String) key.attachment();
            SocketChannel socketChannel = (SocketChannel) key.channel();
            //发送ack
            doReadAck(socketChannel, message);
            //消息置成 OP_READ
            key.interestOps(SelectionKey.OP_READ);
        }
    }


    /**
     * 读取数据操作
     *
     * @author zhuangjiesen
     * @date 2018/10/11 下午5:45
     * @param
     * @return
     */
    public static String readChannelData(SocketChannel socketChannel){
        try {
            ByteBuffer buf = ByteBuffer.allocate(1024);
            // configureBlocking() 设置成非阻塞 ，这里读不到数据直接返回
            int bytesRead = socketChannel.read(buf);
            buf.flip();
            byte[] data = new byte[buf.remaining()];
            buf.get(data);
            String message = new String(data);
            return message;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }


}
```


#### NioClient.java

```


/**
 * @param
 * @Author: zhuangjiesen
 * @Description:
 * @Date: Created in 2018/10/11
 */
public class NioClient {


    public static void main(String[] args) throws Exception {

        Socket client = new Socket();
        client.setKeepAlive(true);
        client.setSendBufferSize(1024);
        client.setTcpNoDelay(true);
        client.setReuseAddress(false);
        client.setReceiveBufferSize(48);

        client.connect(new InetSocketAddress("127.0.0.1", 38888));
        Thread readThread = new Thread(new ReadThread(client.getInputStream()));
        Thread writeThread = new Thread(new WriteThread(client.getOutputStream()));

        readThread.setDaemon(false);
        writeThread.setDaemon(false);

        readThread.start();
        writeThread.start();

        while(true) {}

    }


    public static class ReadThread implements Runnable {

        private InputStream inputStream;

        public ReadThread (InputStream ins) {
            this.inputStream = ins;
        }
        
        @Override
        public void run() {
            try {
                while (true) {
                    BufferedReader reader = new BufferedReader(new InputStreamReader(this.inputStream));
                    System.out.println("尝试读取数据...");
                    String line = null;
                    while ((line = reader.readLine()) != null) {
                        System.out.println("读取数据: " + line);
                    }
                    Thread.currentThread().sleep(3000);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }

        }
    }



    public static class WriteThread implements Runnable {

        private OutputStream outputStream;

        public WriteThread (OutputStream outs) {
            this.outputStream = outs;
        }


        @Override
        public void run() {
            try {
                BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(this.outputStream));
                while (true) {
                    String line = "定时数据_:" + System.currentTimeMillis();
                    System.out.println(line);
                    writer.write(line);
                    writer.flush();
                    Thread.currentThread().sleep(3000);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

}

```

