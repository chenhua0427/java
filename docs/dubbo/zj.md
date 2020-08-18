## Dubbo关键模块总结

##### URL

url类贯穿了一次Dubbo调用的整个过程，Dubbo中每个可配置的单元都有唯一的url，配置参数通过url携带。

例如，注册中心：`zookeeper://224.5.6.7:2181`；一个服务`dubbo://10.0.75.1:20880/org.apache.dubbo.demo.DemoService?&interface=org.apache.dubbo.demo.DemoService&methods=sayHello&side=provider&timestamp=1585553085050`

##### @EnableDubbo注解

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@EnableDubboConfig
@DubboComponentScan
public @interface EnableDubbo {
   ...
}
```

##### Proxy

通过@Reference注解注入，referenceBean.get()中的init()子方法createProxy()生成远程代理。Dubbo提供两种方式生成代理：JDK代理在`InvokerInvocationHandler`类中实现；javassist代理在`JavassistProxyFactory`类中实现。由于代理类时通过Invoker来发送和接收请求，所以生成代理之前要先初始化Inovker：`ProxyFactory.getProxy(invoker)`。

```java
@SPI("javassist")
public interface ProxyFactory {
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;

    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException;

    @Adaptive({PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;
}
```

##### Invoker

初始化过程：当ReferenceBean拿到url后根据协议找到对应的Protocol，然后调用`Protocol.refer()`方法生成。如果url是注册中心地址则对应RegistryProtocol，如果rul是某个服务的直接地址则找到的是DubboProtocol，由抽象类`AbstractProtocol`实现：

```java
@Override
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        return new AsyncToSyncInvoker<>(protocolBindingRefer(type, url));
    }
```

```java
    @Override
    public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
        optimizeSerialization(url);

        // 创建Dubbo Invoker
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
        invokers.add(invoker);

        return invoker;
    }
```

ClusterInvoker是Dubbo支持集群调用的核心实现，包括负载均衡、路由、容错等，Inovker还包含过滤Filter；invoker构建完成之后不会直接返回给Proxy，还需要和Filter集成最终返回一个由Filter构成的Inovker链，返回给Porxy的是过滤链的头部Header Invoker。它的初始化依赖url参数，首先需要获取url：

1. 判断是否是进程内调用，比如配置的@Reference(url="injvm:localhost:28080")，这种方式会直接走进程内调用。
2. 判断用户是否在@Reference配置了url参数，如果是则忽略注册中心的配置。
3. 如果配置的是注册中心的地址@Reference(url="registry://...")，则会将配置参数加在refer参数中，比如registry://localhost:28080?refer=version%3f1.0.0
4. 如果没有配置url，则检查是否有注册中心，没有的化抛出异常。
5. 获取注册中心的url，并添加refer参数。
6. 如果最终只有一个url，则使用Protocol的refer方法获取到Invoker。
7. 如果最终有多个url，则生成多个Invoker。
8. 最终会将多个Invoker合并成要给ClusterInvoker。

##### Exchanger

invoker生成的方法中包含一个`getClients(url)`参数。因为Invoker承上启下，所以它需要调用`ExchangeClient`**协议打包**、**连接建立**、**异步io的结果接收**。

Dubbo的服务是以接口为粒度的，每个Invoker对应了一个远程接口的调用封装。在实际应用中，一个应用会包含多个接口，如果对应同一个应用的多个Invoker每个都初始化一个Client的话，万一接口过多，会造成每个Consumer和Provider之间建立多个Connection，而且连接数随着consumer的个数增加而成倍数的增加。所以shareClient的意思就是对于同一个ip+port，所有invoker共享client，有点类似于连接池的概念，达到节约资源的目的。

```java
public HeaderExchangeClient(Client client, boolean startTimer) {
        Assert.notNull(client, "Client can't be null");
        this.client = client;
        this.channel = new HeaderExchangeChannel(client);

        if (startTimer) {
            URL url = client.getUrl();
            startReconnectTask(url); //断后重连
            startHeartBeatTask(url); //心跳检测
        }
    }
```

##### 异步请求的发送过程

在系统初始化阶段，创建Invoker的时候会初始化一个和服务端交互的`ExchangeClient`，对于Dubbo协议来说，就是`DubboInvoker`中包含一个或者多个`HeaderExchangeClient`。当代理第哦啊用invoke方法发送请求时，invoker选择一个Client发送request(通过`ExchangeChannel`发送出去，同时注册一个`ChannelHandler`来异步接收Response)。如果时OneWay请求，调用send方法，直接返回成功或失败的结果。如果是TwoWay的请求，Client会返回一个Future给invoker，然后client在收到response后，会将收到的结果set到Future中，调用方法就可以拿到远程接口的返回值了。

##### Transporter

ExchangeClient初始化建立connection实际是通过`Transporters.connect()`方法来连接的Server端。Transporters是一个工具类，最终是使用的具体Transporter实现类的`connect()`方法。对于Dubbo协议来说，默认的就是**NettyTransporter** 

### 顶层接口总结

**ProxyFactory**-构建Proxy。`JavassistProxyFactory`类(默认)、`JDKProxyFactory`类。

```java
public class JavassistProxyFactory extends AbstractProxyFactory {
    @Override
    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
public class JdkProxyFactory extends AbstractProxyFactory {
    @Override
    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
    }
```

**Protocol**-远程调用层，封装RPC调用，构建invoker、暴露服务。DubboProtocol类、RegistryProtocol类，根据url判断使用哪个。

~~~java
public interface Protocol extends org.apache.dubbo.rpc.Protocol {
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException; // 暴露服务
    <T> Invoker<T> refer(Class<T> aClass, URL url) throws RpcException; // 构建invoker
}
~~~

**Invoker**-执行远程方法调用。DubboInvoker类。`DubboInvoker.doInovke(final Invocation invocation)`

**Invocation**-请求方法参数的封装。

**Exchanger**-信息交互层，协议打包、连接建立、接收异步io结果。针对服务端和消费端分别封装成`ExchangeClient`、`ExchangeServer`。Dubbo中默认实现类是`HeaderExchanger`：

```java
public class HeaderExchanger implements Exchanger {
    public static final String NAME = "header";
    @Override // 获取一个连接Client
    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }
    @Override // 绑定监听一个端口
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }
}
```

**Transporter**-网络传输层，抽象mina、netty。以 `Message` 为中心，扩展接口为 `Channel`, `Transporter`, `Client`, `Server`, `Codec`。

**Registry**-注册中心层，封装服务地址的注册与发现，以URL为中心。

**Config**-配置层，对外配置接口，以ServiceConfig、ReferenceConfig为中心。

**Serialize**-数据序列化层，扩展接口为 `Serialization`, `ObjectInput`, `ObjectOutput`, `ThreadPool`