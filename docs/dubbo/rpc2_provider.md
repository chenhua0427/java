## 白话Dubbo（二）：RPC之服务提供端

##### Dubbo RPC 服务端组件

服务端的模块顺序正好和消费者端相反。反序列化和协议解析，请求传输，服务暴露、本地调用。

##### 请求传输

Provider要提供服务，必须在某一端口上监听来接收数据，所以跟Client对于要有个Server。

- **Server** 为了支持多协议，Dubbo将Server抽象为ExchangeServer接口

  ~~~java
  public interface ExchangeServer extends RemotingServer {
      /**
       * 获取所有和client的Channel
       */
      Collection<ExchangeChannel> getExchangeChannels();
      /**
       * 获取指定Client的Channel
       */
      ExchangeChannel getExchangeChannel(InetSocketAddress remoteAddress);
  }
  ~~~

  ExchangeServer启动后，每个客户端连接对应者一个channel。channel代表是两个端点之间的连接通道，所以是没有Client和Server之分，接口自然只有一个。发送数据用request方法，接收数据用ExchangeHandler/只是大部分请情况下Server端不会主动发送request，而是Handler接收请求后，恢复Response。

  ~~~java
  public interface ExchangeChannel extends Channel {
      /**
       * 发送请求
       */
      CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException;
      /**
       * 接收数据的handler
       */
      ExchangeHandler getExchangeHandler();
      /**
       * 关闭Channel
       */
      @Override
      void close(int timeout);
  }
  ~~~

  ExchangeServer和ExchangeClient一样，都是从Exchanger接口来的，底层依赖Transporter。

  ~~~java
  @SPI(HeaderExchanger.NAME)
  public interface Exchanger {
      /**
       * 开启一个服务端Server
       */
      @Adaptive({Constants.EXCHANGER_KEY})
      ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException;
      /**
       * 获取Client
       */
      @Adaptive({Constants.EXCHANGER_KEY})
      ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException;
  }
  ~~~

- **Handler** 回到数据流上，请求到达后，经过协议解析和对象的反序列化，以Request对象的形式到达ExchangeHandler。handler负责调用接口实现类，然后通过网络返回。

  ~~~java
  public interface ExchangeHandler extends ChannelHandler, TelnetHandler {
      /**
       * reply.
       */
      CompletableFuture<Object> reply(ExchangeChannel channel, Object request) throws RemotingException;
  }
  ~~~

##### 服务暴露

请求到达服务端后，需要调用办呢滴接口的实现类。同样是使用Invoker.invoke(Invocation invo)。

- **Exporter** Invoker调用本地方法，首要的自然是找到本地实现类在什么地方，这时候就需要引入另外一个接口

  ~~~java
  public interface Exporter<T> {
      /**
       * 获取Invoker
       */
      Invoker<T> getInvoker();
      /**
       * 取消接口暴露
       */
      void unexport();
  }
  ~~~

  <u>Exporter代表本地接口服务的暴露，即封装了一个本地调用Invoker</u>。可以这么理解，本地有一个接口的实现，想让其他服务能够远程调用到，那就需要按照一定的协议注册成一个Exporter，否则即使远程发送请求过来要访问该实现，Dubbo也不允许。Protocol负责管理接口的暴露，DubboProtocaol中的DubbolExproter。

  ~~~java
  @SPI("dubbo")
  public interface Protocol {
      //暴露服务接口
      @Adaptive
      <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
  
       //获取接口调用的Invoker
      @Adaptive
      <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
  }
  ~~~

![image](https://github.com/chenhua0427/java/tree/master/docs/images/dubbo-rpc2.jpg)

<u>Exchanger</u>-开启一个服务端Server，`ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException;` ExchangeServer接口包含获取和客户端的Channel的方法。

<u>ExchangeHandler</u>-负责回复Response，即handler负责调用接口的实现类，然后将结果通过网络返回。

<u>Protocol</u>-对接口协议(默认Dubbo:)的抽象，是Dubbo的核心模块(获取服务、暴露服务)，提供服务暴露的方法<T> Exporter<T> export(Invoker<T> invoker);。

<u>Exporter</u>-代表本地服务的暴露，包含获取Invoker方法`Invoker<T> getInvoker();`和取消暴露接口方法`void unexport();` 。

<u>Invoker</u>-调用动作封装类，提供invoke(Invocation invocation)方法，发起一次本地调用。Invocation 封装了真实调用的方法Method和参数。