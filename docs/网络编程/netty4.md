## Netty核心

##### 总结

- NioEventLoopGroup 实际上就是个线程池，一个 EventLoopGroup 包含一个或者多个 EventLoop；

- 一个 EventLoop 在它的生命周期内只和一个 Thread 绑定；

- 所有有 EnventLoop 处理的 I/O 事件都将在它专有的 Thread 上被处理；

- 一个 Channel 在它的生命周期内只注册于一个 EventLoop；

- 每一个 EventLoop 负责处理一个或多个 Channel；

  以下为netty模型详细版：

  ![netty_reactor](E:\note\docs\image\netty_reactor.jpg)

  - 抽象出2组线程池NioEventLoopGroup，boosGroup负责接收客户端连接<font color="red">setp 1</font> accept，并封装channel成NioSocketChannel注册到workGroup的selector中NioEventLoop <font color="red">setp 2</font>；

  - <font color="red">setp 3</font>中处理的任务是通过以下方式写入的任务，包括普通任务task和延迟任务ScheduledTask。

    ~~~java
    ctx.channel().eventLoop().execute(()->{}); //Runnable接口
    ctx.channel().eventLoop().schedule(()->{ },5, TimeUnit.SECONDS);//Runnable接口
    ~~~

  - workGroup负责read、write事件，即在对于NioSocketChannel上处理。

##### NioEventLoopGroup -线程池

1. Netty抽象出两组线程池，boosGroup负责接收客户端连接，并封装channel注册到workGroup的selector中；workGroup负责网络读写操作。
2. NioEventLoop表示一个不断循环执行处理任务的线程，每个NioEventLoop都有一个selector、taskQueue，selector用于监听绑定在其上的socket网络通道NioSocketChannel；
3. NioEventLoop内部采用串行化设置，从消息的读取、解码、处理、编码、发送，始终由IO线程NioEventLoop负责。

##### ServerBootstrap -启动类

配置整个Netty程序，串联各个组件；客户端是Bootstrap。

- group(EventLoopGroup group) 设置NioEventLoopGroup 
- channel(Class channelClass) 设置一个服务器端的通道实现
- option(ChannelOption<T>,T value) 给ServerChannel添加配置
- childOption(ChannelOption<T>,T value)  给接收到的通道添加配置
- childHandler(ChannelHandler childHandler) 设置workGroup业务处理类
- handler(ChannelHandler childHandler) 设置bossGroup业务处理类
- bind(int prot) 服务器端设置占用监听端口号
- connect(String ip,int prot) 客户端连接服务器

```java
		NioEventLoopGroup boss = new NioEventLoopGroup(1);//默认创建cpu*2个线程
        NioEventLoopGroup worker = new NioEventLoopGroup(4);
        try {
            // 创建服务器端的启动对象，配置启动参数
            ServerBootstrap bootstrap = new ServerBootstrap();
            //使用链式编程进行设置
            bootstrap.group(boss, worker)
                    .channel(NioServerSocketChannel.class)//设置channel类型
                    .option(ChannelOption.SO_BACKLOG, 128)//设置线程队列等待连接个数
                    .childOption(ChannelOption.SO_KEEPALIVE, true)//设置保持活动连接状态
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {//给pipeline设置处理器
                            ch.pipeline().addLast(new NettyServerHandler());
                        }
                    });//给workergroup的EventLoop对应的管道设置处理器handler
            System.out.println("服务器准备好了.....");
            //绑定端口并同步，生成一个ChannelFuture对象
            //sync 方法用于阻塞当前 Thread，一直到端口绑定操作完成
            ChannelFuture cf = bootstrap.bind(6668).sync();
            //对关闭通道进行监听
            //应用程序将会阻塞等待直到服务器的 Channel 关闭
            cf.channel().closeFuture().sync();
        }finally {
            boss.shutdownGracefully();
            worker.shutdownGracefully();
        }
```

##### Channel

不同协议、不同的阻塞类型的连接都有不同的Channel与之对应，常用的有：

- <font color="#1E90FF">`NioSocketChannel`</font> 异步的客户端 TCP Socket连接

- <font color="#1E90FF">`NioServerSocketChannel`</font> 异步的服务器端 TCP Socket连接

- <font color="#1E90FF">`NioDatagramChannel`</font> 异步的UDP连接

  ~~~java
  bootsrap.channel(NioDatagramChannel.class);// 1.
  bootsrap.channel.handler(new MyHandler())// 2.由于没有连接，不需要设置pipeline
  public class MyHandler extends SimpleChannelInboundHandler<DatagramPacket> {
      public void messageReceived(ChannelHandlerContext ctx, DatagramPacket packet){
      }
  }
  ~~~

  

- <font color="#1E90FF">`NioSctpChannel`</font> 异步的客户端 Sctp连接

- <font color="#1E90FF">`NioSctpServerChannel`</font> 异步的服务器端 Sctp连接

##### ChannelHandler

具体业务处理器handler，处理I/O事件或拦截I/O操作，并将其转发到其ChannelPipeline中的下一个处理程序。

ChannelInboundHandler pipeline中事件运动方向从服务端->客户端

ChannelOutboundHandler 事件运动方向从客户端-服务端

ChannelHandlerAdapter

- ChannelInboundHandlerAdapter
  - SimpleChannelInboundHandler<I>
- ChannelOutboundHandlerAdapter

常用handler：

~~~java
// 日志处理器
public class LoggingHandler extends ChannelDuplexHandler {
    public LoggingHandler(LogLevel level) {}
}
// 心跳检测处理器
public class IdleStateHandler extends ChannelDuplexHandler {
    // 服务器读空闲、写空闲、读写空闲时，任意就会发送一个心跳检测包检测是否连接
    // 检测 客户端是否异常断开连接，当触发后，就会传递给管道的下一个handler去处理，即在下一个handler的userEventTriggered方法中实现处理。
     public IdleStateHandler(long readerIdleTime, long writerIdleTime, long allIdleTime, TimeUnit unit) {
        this(false, readerIdleTime, writerIdleTime, allIdleTime, unit);
    }
}
~~~

~~~java
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        // server will close channel when server don't receive any heartbeat from client util timeout.
        if (evt instanceof IdleStateEvent) {
            NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
            try {
                logger.info("IdleStateEvent triggered, close channel " + channel);
                channel.close();
            } finally {
                NettyChannel.removeChannelIfDisconnected(ctx.channel());
            }
        }
        super.userEventTriggered(ctx, evt);
    }
~~~

- HttpServerCodec http协议编码解码handler
- ChunkedWriteHandler
- HttpObjectAggregator
- WebSocketServerProtocolHandler

##### ChannelHandlerContext 

​      代表了 ChannelHandler 和 ChannelPipeline 之间的关联，每当有 ChannelHandler 添加到 ChannelPipeline 中时，都会创建 ChannelHandlerContext。

​      保存Channel相关的所有上下文信息，同时关联一个ChannelHandler对象；即ChannelHandlerContext 中包含一个具体的事件处理器ChannelHandler，同时ChannelHandlerContext 也绑定了对应的pipeline和channel信息，方法对ChannelHandler进行调用。常用方法：

- writeAndFlush(Object msg) 将数据写到pipeline中当前的channelhandler的下一个channelhandler开始处理（出站）
- close() 关闭通道
- flush() 刷新

##### ChannelPipeline

是一个ChannelHandler的集合，它负责处理和拦截出入站的事件和操作，相当于一个贯穿netty的链；实现了一种高级形式的拦截过滤器模式，使用户可以完全控制事件的处理方式，以及channel中各个handler如何相互交互。

##### ChannelOption

- SO_BACKLOG 初始化服务器可连接队列大小，超出的连接请求放在队列中等待处理。
- SO_KEEPALIVE 一致保持连接的活动状态

##### TaskQueue -任务队列

```java
ctx.channel().eventLoop().execute(()->{}); //Runnable接口
```

##### ScheduledTaskQueue -延迟任务队列

```java
ctx.channel().eventLoop().schedule(()->{ },5, TimeUnit.SECONDS);//Runnable接口
```

##### ChannelFuture -异步结果接口

调用者不能立刻获得结果，而是通过Futur-Listener机制，用户可以通过主动获取或者通过通知机制获得IO操作结果。`ChannelFuture cf = bootstrap.bind(6668).sync();`

- 主动获取

  ~~~java
  cf.isDone();// isSucess()、getCause()、isCancelled()
  ~~~

- 添加监听器

  ```java
  cf.addListener(new ChannelFutureListener() {
      @Override
      public void operationComplete(ChannelFuture future) throws Exception {
          if(future.isSuccess()){
              System.out.println("监听端口 成功");
          }else {
              Throwable cause = channelFuture.cause();
          }
      }
  });
  ```

##### ByteBuf

~~~java
ByteBuf buf = Unpooled.buffer(10);// 创建一个数组byte[10]
for(int i=0;i<10;i++){
    buf.writeByte(i);
}
//.capacity()表示容量，.readableBytes()可读取的字节数
for(int i=0;i<buf.capacity();i++){
    buf.readByte();
}
~~~

##### Unpooled

专门用来操作缓冲区(netty的数据容器)的类。

- copiedBuffer(CharSequence str,Charset cs)

  ~~~java
  ByteBuf buf = Unpooled.copiedBuffer("hello",CharsetUtil.UTF_8);
  if(buf.hasArray()){
      byte[] content = buf.array();
  }
  ~~~

##### 编码解码器

netty提供了很多，比如StringDecoder对字符串、ObjectDecoder对java对象；由于其底层还是使用的java序列化技术，其存在如下问题：

- 无法跨语言
- 序列化后体积太大，是二进制编码的5倍多
- 序列化性能太低

引入新技术- protobuf，适合做数据存储或RPC数据交换格式；支持跨平台、跨语言。

Netty常用ChildHandler：

- <font color="#1E90FF">`LoggingHandler("DEBUG")`</font> 开启日志

- <font color="#1E90FF">`SslHandler`</font>

  ~~~java
      protected void initChannel(Channel ch) throws Exception {
          SSLEngine engine = context.newEngine(ch.alloc());  //2
          engine.setUseClientMode(client); //3
          // addFirst 确保所有其他 ChannelHandler 应用他们的逻辑到数据后加密后才发生
          ch.pipeline().addFirst("ssl", new SslHandler(engine, startTls));  //4
      }
  /* 1.使用构造函数来传递 SSLContext 用于使用(startTls 是否启用)
     2.从 SslContext 获得一个新的 SslEngine 。给每个 SslHandler 实例使用一个新的 SslEngine
     3.设置 SslEngine 是 client 或者是 server 模式
     4.添加 SslHandler 到 pipeline 作为第一个处理器 */
  ~~~

- <font color="#1E90FF">`HttpServerCodec`</font> http协议编解码器

- <font color="#1E90FF">`HttpObjectAggregator`</font> http协议解码器，需要放在`HttpServerCodec` 后面

  ~~~java
  // 配合FullHttpRequest使用
  public class XXX extends SimpleChannelInboundHandler<FullHttpRequest> {
      protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest msg){
          // msg中包含了请求头和请求体msg.content()
      }
  }
  ~~~

- <font color="#1E90FF">`LineBasedFrameDecoder`</font> 
- <font color="#1E90FF">`ChunkedWriteHandler`</font> 支持异步写大数据流不引起高内存消耗
- <font color="#1E90FF">`WebSocketServerProtocolHandler`</font> WebSocket编解器
- <font color="#1E90FF">`IdleStateHandler`</font> 心跳检测

##### websocket通讯

- 帧frame

  - 数据帧

    TextWebSocketFrame 文本帧，用来传输文字

    BinaryWebSocketFrame 二进制帧，传输图片、音频、视频等

  - 状态帧

    PingWebSocketFrame ping帧(客户端发出)

    PongWebSocktFrame pomg帧(服务端响应)

    CloseWebSocketFrame 关闭帧