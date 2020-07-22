## 白话Dubbo（一）：RPC之消费者端

##### 序列化

程序中接口调用传的是对象，实际上是一堆只有VM能读的内存地址，而网络上能传的是0和1组成的字节数据，所以就有一个转换的过程，<u>序列化</u>。常见的序列化协议由`json`、`protobuf`等。

##### 网络通信

网络上传输的是一个个数据包组成的数据流，数据在经过一个个网络中间节点的时候，会不断的被合并成大哥数据包或拆分成小的。所以要依赖网络协议来对端重新组合数据包。流行的RPC框架一般都采用基于tcp协议的上层协议，比如http，当然也有很多跟Dubbo一样使用基于tcp的私有协议(dubbo协议)。封装网络协议的模块一般叫`Codec`，包含encode、decode方法。

Codec只负责对象和0/1之间转换，但是网络通信还包括建立和Provider的连接并发送和接收数据包。这个时候又会抽象出一个传输层`Transproter`专门负责建立和关闭连接，维护连接池，处理连接和断开事件。在Java中，**netty**是应用最多的框架。

传输层通常不会直接接收Consumer发过来的对象，通常会封装一个`Request`和`Response`模块。

##### 代理

Consumer调用一个接口的方法，实际上是调用一个接口实现的引用。而远程调用只能选择容器来注入对象的引用，RPC需要在容器中构造一个接口的实现，在方法实现中调用远程网络接口。这就是代理`Proxy`，Java中一般使用动态代理实现。

~~~java
@EnableAutoConfiguration
public class DubboAutoConfigurationConsumerBootstrap {

    private final Logger logger = LoggerFactory.getLogger(getClass());

    @Reference(version = "1.0.0", url = "dubbo://127.0.0.1:12345")
    private DemoService demoService;

    public static void main(String[] args) {
        SpringApplication.run(DubboAutoConfigurationConsumerBootstrap.class).close();
    }

    @Bean
    public ApplicationRunner runner() {
        return args -> logger.info(demoService.sayHello("mercyblitz"));
    }
}
~~~

代码中DemoService就是远程API接口，Dubbo使用自定义的@Reference注解来注入Proxy实现，对这个接口的调用，会通过Proxy将请求发到url的Provider。

Dubbo通过调用代理工厂ProxyFactory.getProxy()来获得一个接口的代理实现类。

~~~java
@SPI("javassist")
public interface ProxyFactory {
    /**
     * create proxy.
     *
     * @param invoker
     * @return proxy
     */
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;
}
~~~

Dubbo中提供了两种方式实现ProxyFactory接口：一是JDK动态代理；二是JavassistProxyFactory，Javaassist是一个运行期字节码工具，它允许在运行时动态编译代码成class。Dubbo默认做法就是在初始化时生成一个接口的实现类的源代码，通过javaassist编译后加载，Consumer实际上就时调用的这个类。

##### 代理实现原理

代理有了，剩下的事情就按传输层要求的格式封装调用参数(Request/Response)。但Dubbo为了支持多种协议，保证扩展性，增加了其它处理。

重点的三个类：`Protocol`、`Invoker`、`Invocation`

- **Invoker** 是调用动作封装类，作用类似于Runnable相对于Thread。调用它的invoke()发起一次调用，这个调用有可能是远程或本地，对于调用方法来说是透明的。一般来说，每个远程服务接口都会对应一个Invoker实例。

~~~java
public interface Invoker<T> extends Node {
    /**
     * 获取Invoker调用的target接口类
     */
    Class<T> getInterface();
    /**
     * 发起一次调用
     */
    Result invoke(Invocation invocation) throws RpcException;
}
~~~

- **Invocation** invoke()操作的输入参数，就是把真实调用的method，输入参数等封装一下。其定义的部分方法如下：

~~~java
public interface Invocation {
    ...
    /**
     * 获取方法名
     */
    String getMethodName();
    /**
     * 获取接口名
     */
    String getServiceName();
    /**
     * 参数类型
     */
    Class<?>[] getParameterTypes();
   /**
     * 调用参数
     */
    Object[] getArguments();
     ...
}
~~~

- **Protocol** Dubbo对接口协议的抽象，主要是两个功能，对于Provider来说，通过Protocol把本地接口暴露成一个指定协议的Server；对于Consumer来说可以获取一个针对特定接口协议的Invoker。

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

看完上面的几个接口，把关系捋一下：

1. Consumer获取到接口的Proxy，调用方法时，Proxy根据远程服务的配置协议找对应的Protocol实现，默认是`DubboProtocol`。
2. 调用Protocol的refer方法获取到Invoker，如果使用DubboProtocol，那就是`DubboInvoker`。
3. 将Consummer元素的输入参数封装成一个Invocation(对应于远程调用就是`RpcInvocation`)，调用Invoker的invoke()方法。

<u>注意</u>：以上只是逻辑流程，不是正在代码调用流程，因为代码实现中对象都是提前初始化的，初始化的代码流程和逻辑流程不是完全一样。

##### 请求传输

Dubbo的传输层模块抽象层数较多，以下列举关键部分。

- **Client** 作为Consumer和服务提供端通信，针对不同协议都会提供Client，Dubbo将其抽象为ExchangeClient，用于发送Request并将收到的结果转化成Response。

  ~~~java
  public interface ExchangeClient extends Client, ExchangeChannel {}
  ~~~

  ExchangeClient接口是一个组合接口，组合了`Client`和`ExchangeChannel`。

  ~~~java
  public interface Client extends Endpoint, Channel, Resetable, IdleSensible {
      // 建立连接
      void reconnect() throws RemotingException;
      ...
  }
  
  public interface ExchangeChannel extends Channel {
      // 发送请求
      CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException;
      
      ExchangeHandler getExchangeHandler();//接收结果
      /**
       * 关闭Channel
       */
      @Override
      void close(int timeout);
  }
  ~~~

  Dubbo中接口的默认实现类分布式HeaderExchangeClient和HeaderExchangeHandler。

  Client仅仅属于传输层的一般，对于整个传输层的抽象则是Exchanger接口：

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

- **Transporter** 对于Client来说，发送和接收是Request和Response，它所依赖的底层数据的传输抽象为Traansporter。

  ~~~java
  @SPI("netty")
  public interface Transporter {
      /**
       * Bind a server.
       */
      @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
      RemotingServer bind(URL url, ChannelHandler handler) throws RemotingException;
      /**
       * Connect to a server.
       */
      @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
      Client connect(URL url, ChannelHandler handler) throws RemotingException;
  }
  ~~~

  Transproter层只管传数据，没有交互的概念，不管数据背后的意义。而Exchanger是应用传输层，有Request和Response的概念。Transproter默认实现类**NettyTransporter**。

- **编码** Transporter在收到数据后，需要协议来打成数据包，Dubbo负责编码的是Codec接口，其默认实现类是`ExchangeCodec` 

  ~~~java
  @SPI
  public interface Codec2 {
      @Adaptive({Constants.CODEC_KEY})
      void encode(Channel channel, ChannelBuffer buffer, Object message) throws IOException;
  
      @Adaptive({Constants.CODEC_KEY})
      Object decode(Channel channel, ChannelBuffer buffer) throws IOException;
      enum DecodeResult {
          NEED_MORE_INPUT, SKIP_SOME_INPUT
      }
  }
  ~~~

- **序列化** Dubbo对序列化的抽象接口是Serialization，默认实现类是`Hession2Serialization`。

  ~~~java
  @SPI("hessian2")
  public interface Serialization {
      /**
       * Get content type unique id
       */
      byte getContentTypeId();
      /**
       * content type
       */
      String getContentType();
      /**
       * 序列化
       */
      @Adaptive
      ObjectOutput serialize(URL url, OutputStream output) throws IOException;
      /**
       * 反序列化
       */
      @Adaptive
      ObjectInput deserialize(URL url, InputStream input) throws IOException;
  }
  ~~~

##### Consummer总结

![image](https://github.com/chenhua0427/java/tree/master/docs/images/dubbo-rpc1.jpg)

<u>Invoker</u>-调用动作封装类，提供invoke(Invocation invocation)方法，发起一次远程调用。Invocation 封装了真实调用的方法Method和参数。

<u>Protocol</u>-对接口协议(默认Dubbo:)的抽象，是Dubbo的核心模块(获取服务、暴露服务)，提供获取接口调用的Invoker方法`<T> Invoker<T> refer(Class<T> type, URL url);`。

<u>ExchangeClient</u>-获取连接、发送请求，`void reconnect()` 和`CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor)`

<u>Transporter</u>-传输数据，默认使用Netty。

<u>Codec</u>-根据协议打数据包。

<u>Serialization</u>-序列化。

