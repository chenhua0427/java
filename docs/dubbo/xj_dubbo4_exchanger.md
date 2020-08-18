## 详解Dubbo（四）：消费端请求发送Exchanger

#### 前言

之前讲了消费端代理的生成，最终到请求发送操作由Invoker来完成。Invoker同时集成了集群服务发现和路由功能，还集成了调用过程自定义扩展Filter。Invoker是业务对象的分水岭，请求到达Invoker之前，都是以业务接口和方法的方式调用，就是说调用方要拿到接口定义的API。Invoker之后就是Exchanger和Transproter，只存在Requset和Response了，在这里接口和方法变成了Requset的一个参数。这篇文章看下Dubbo如何初始化远程通信Client并发送和接收请求的。

#### Client初始化

Invoker调用的是Exchanger，Exchanger层针对消费端和服务端分别封装成ExchangClient和ExchangeServer。他们大部分接口都是一样的，只是对于Client来说，支持connect()操作来和提供端建立连接；Server端，需要通过bind操作来监听端口，来接收消费端的连接请求。其它的数据发送和接受对于两端来说一样。

##### Dubbo Client初始化

以Dubbo协议为例，在DubboProtocol的Invoker初始化操作：

```java
    public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
        ...
        // 创建Invoker
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
        invokers.add(invoker);
        return invoker;
    }
```

在构造Invoker的时候，需要传入一个Client的列表，之后Invoker通过client发送请求和接受返回结果。至于请求协议打包、连接建立、异步IO的结果接受等操作留给ExchangeClient来实现。下面看getClient实现：

```java
private ExchangeClient[] getClients(URL url) {
        boolean useShareConnect = false;
        //连接数配置
        int connections = url.getParameter(CONNECTIONS_KEY, 0);
        List<ReferenceCountExchangeClient> shareClients = null;
        // 如果没设置连接数，说明Consumer希望使用共享连接
        if (connections == 0) {
            useShareConnect = true;
            //共享连接数配置
            String shareConnectionsStr = url.getParameter(SHARE_CONNECTIONS_KEY, (String) null);
            connections = Integer.parseInt(StringUtils.isBlank(shareConnectionsStr) ? ConfigUtils.getProperty(SHARE_CONNECTIONS_KEY,
                    DEFAULT_SHARE_CONNECTIONS) : shareConnectionsStr);
            //获取共享Client
            shareClients = getSharedClient(url, connections);
        }
        //根据配置的连接个数初始化Client
        ExchangeClient[] clients = new ExchangeClient[connections];
        for (int i = 0; i < clients.length; i++) {
            if (useShareConnect) {
                //使用共享client
                clients[i] = shareClients.get(i);
            } else {
                //初始化Client
                clients[i] = initClient(url);
            }
        }
        return clients;
    }
```

上面的逻辑中涉及到共享Client的概念，因为Dubbo的服务是以接口为粒度的，每个Invoker对应了一个远程接口的调用封装。在实际应用这噢乖，一个应用会包含多个接口，如果对应同一个应用的的多个Inovker每个都初始化一个Client的话，万一接口过多，会造成每个Consumer和Provider之间建立多个Connection，而且连接数随着consumer的个数增加而成倍数增加。所以shareClient的意思就是对于同一个ip+port，所有Inovker共享client，有点类似于连接池的概念，达到节约资源的目的。共享client最终初始化client的方式和普通的是一样的，initClient()。

```java
private ExchangeClient initClient(URL url) {
        // client通信框架，默认netty
        String str = url.getParameter(CLIENT_KEY, url.getParameter(SERVER_KEY, DEFAULT_REMOTING_CLIENT));
        //使用Dubbo通信协议
        url = url.addParameter(CODEC_KEY, DubboCodec.NAME);
        // 设置发送心跳的间隔
        url = url.addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT));
        // 检查传输层是否支持该框架
        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported client type: " + str + "," +
                    " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
        }

        ExchangeClient client;
        try {
            // client延迟建立连接，即第一次调用时才connect，已经不推荐使用
            if (url.getParameter(LAZY_CONNECT_KEY, false)) {
                client = new LazyConnectExchangeClient(url, requestHandler);
            } else {
                //初始化并连接server
                client = Exchangers.connect(url, requestHandler);
            }
        } catch (RemotingException e) {
            throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
        }
        return client;
    }
```

上面的初始化过程除了初始化参数外，就是调用工具类`Exhangers`来初始化client。这里面除了url之外还有一个`ExchangHandler`参数，这个是用来处理服务端主动发送来的消息的，对于consumer发送请求的场景这里涉及不到。工具类`Exhangers.connect()`实现：

```java
public static ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
        return getExchanger(url).connect(url, handler);
    }
```

具体实现就是根据url参数获取对应的`Exchanger`，然后调用它的connect方法。Dubbo默认实现类是`HeaderExchanger`。

```java
public class HeaderExchanger implements Exchanger {

    public static final String NAME = "header";

    @Override
    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }

    @Override
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

}
```

上面实现中，通过调用connect()方法获取到ExchangeClient的实现`HeaderExchangeClient`，这里会通过工具类`Transporters`获取到一个传输层的Client对象。获取对象时，对传入的handler，又做了两层封装，`DecoderHander`用来对收到的Response中数据部分解码成对象；而`HeaderExchangeHandler`作用是对于异步返回的Response找到当时的Reuqest，将结果放回给当初调用方。

##### HeaderExchangeClient初始化

```java
public HeaderExchangeClient(Client client, boolean startTimer) {
        Assert.notNull(client, "Client can't be null");
        this.client = client;
        this.channel = new HeaderExchangeChannel(client);

        if (startTimer) {
            URL url = client.getUrl();
            startReconnectTask(url);
            startHeartBeatTask(url);
        }
    }
```

由上面的Client的构造函数可以看出主要做了两件事：首先将传入的client实现封装了一层，client的方法比如request()都是直接调用channel的方法；其次启动了两个定时任务，一个是在和server端的connection断开重连，一个是定时心跳发送。下面看下`HeaderExchangeChannel`对client做了一层封装后，主要干了什么：

```java
@Override
    public void send(Object message, boolean sent) throws RemotingException {
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send message " + message + ", cause: The channel " + this + " is closed!");
        }
        if (message instanceof Request
                || message instanceof Response
                || message instanceof String) {
            channel.send(message, sent);
        } else {
            Request request = new Request();
            request.setVersion(Version.getProtocolVersion());
            request.setTwoWay(false);
            request.setData(message);
            channel.send(request, sent);
        }
    }
    @Override
    public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
        }
        // create request.
        Request req = new Request();
        req.setVersion(Version.getProtocolVersion());
        req.setTwoWay(true);
        req.setData(request);
        DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout, executor);
        try {
            channel.send(req);
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }
```

这个类在发送前，将调用参数封装成一个request，设置版本号和请求类型。如果调用的是request方法，急需要接收response，以异步请求的方式发出，返回给调用方（也就是Invoker）一个future。

#### 请求发送过程

现在理一下一次请求的发送过程。在系统初始化阶段，创建Invoker的时候会初始化一个和服务端交互的ExchangeCLient，对于Dubbo协议来说，就是`DubboInvoker`中包含一个或多个`HeaderExchangeClient`。当代理调用Invoker.invoke()方法发送请求时，invoker选择一个Client发送request。如果是OneWay的请求，调用Send方法，直接返回成功或失败的结果。如果是TwoWay的请求，Client会返回一个Future给invoker，然后client在收到response后，会将受到的结果set到Future中，调用方就可以拿到远程接口的返回值了。

#### 接受请求响应

在上面Invoker初始化Client的时候，需传入一个`ExchangeHandler`来接收异步响应回调。在`HeaderExchangeClient`初始化的时候，又在handler上面套了两层，所以最终的关系图大概是下面这样：![dubbo-exchanger](E:\note\docs\image\dubbo-exchanger.jpg)

在底层的`Teansproter`收到Server端的数据并处理后，会将数据给到`DecodeHandler`，这个handler判断返回的数据是否实现了`Decodeable`接口，是的话就嗲用decode()方法并把解码出的value设置到Response中。

```java
    public void received(Channel channel, Object message) throws RemotingException {
        if (message instanceof Decodeable) {
            decode(message);
        }
        //provider端解码Request
        if (message instanceof Request) {
            decode(((Request) message).getData());
        }
       //consumer端解码Response
        if (message instanceof Response) {
            decode(((Response) message).getResult());
        }
       //调用下一个handler
        handler.received(channel, message);
    }
```

第二个handler是`HeaderExchangeHandler`，如果Server返回的是请求响应，最终会到`handleResponse()`中：

```java
static void handleResponse(Channel channel, Response response) throws RemotingException {
        if (response != null && !response.isHeartbeat()) {
            DefaultFuture.received(channel, response);
        }
    }
```

对于TwoWay的请求，request发出后，调用方会得到一个Future，这个Future就是在这个Handler这里填充结果的。这样一次完整的请求就完成了。

#### 总结

Invoker将一次远程方法的调用封装成request后，通过ExchangeClient发送出去，并同传入的ExchangeHandler参数处理异步返回的Response并和之前的Request关联，返回给调用方。Dubbo这里对于Exchange和Transproter的划分使用了MEP设计模式（Message Exchange Pattern）。