## 详解Dubbo（三）：消费端构造Invoker

#### 前言

构建Proxy需要传入Invoker参数。除基本方法外，其他的接口方法的调用最终都是调用invoker.invoke()方法。从RPC调用的整个流程来说，Invoker正好处在中间的位置，它的左边是用户的应用，调用的都是对象和方法；而它的右边是传输层，操作的都是Request和Response，所以Invoker是中间的桥梁。

##### Invoker结构

分为两部分：针对集群的ClusterInvoker和针对特定协议的Invoker。

##### Protocol和Invoker

消费端针对某个服务接口创建Invoker的时候，首先要获取URL。最简单的例子就是在`@Reference`注解上配置了url地址。最简单的URL如下：

```
dubbo://10.0.75.1:20880/org.apache.dubbo.demo.DemoService?&interface=org.apache.dubbo.demo.DemoService&methods=sayHello&side=provider&timestamp=1585553085050
```

当ReferenceBean拿到这个url后就回去找它对应的Protocol类，根据url的schema，dubbo可以找到DubboProtocol，然后调用其refer方法获取到Invoker，在抽象类`AbstractProtocol`里面。

```java
@Override
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    return new AsyncToSyncInvoker<>(protocolBindingRefer(type, url));
}
```

实际上调用的是子类的protocolBindingRefer()方法，这里外层封装的`AsyncToSyncInvoker`是一个装饰类，因为新版本dubbo把所有invoker调用都改成了异步返回，如果Consumer仍然希望同步调用，则用这个装饰类转换一下。下面看DubboProtocol的protocolBindingRefer()实现：

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

该方法直接创建了一个DubboInvoker，总共传入四个参数，第三个参数是构建传输层Client，前面讲过invoker链接了Proxy和传输层，当Invoker发起调用时，就需要这个ExchangeClient来发送请求和接受Response；第四个参数是invoker的缓存集合。

之前讲过Proxy最终是调用Invoker.invoke()发起远程调用，DubboInvoker.invoke()方法。

##### DubboInvoker

对invoke()方法的调用首先会进到`DubboInvoker`的父类`AbstractInvoker`中：

```java
@Override
    public Result invoke(Invocation inv) throws RpcException {
        // 判断invoker是否已经destroy了，是则打印警告，调用继续
        if (destroyed.get()) {
            logger.warn("Invoker for service " + this + " on consumer " + NetUtils.getLocalHost() + " is destroyed, "
                    + ", dubbo version is " + Version.getVersion() + ", this invoker should not be used any longer");
        }
        //追加RpcContext中的附加信息到Invocation中，比如链路追踪的Id等
        RpcInvocation invocation = (RpcInvocation) inv;
        invocation.setInvoker(this);
        if (CollectionUtils.isNotEmptyMap(attachment)) {
            invocation.addObjectAttachmentsIfAbsent(attachment);
        }
        Map<String, Object> contextAttachments = RpcContext.getContext().getObjectAttachments();
        if (CollectionUtils.isNotEmptyMap(contextAttachments)) {
            invocation.addObjectAttachments(contextAttachments);
        }
        //设置是同步还是异步调用
        invocation.setInvokeMode(RpcUtils.getInvokeMode(url, invocation));
        //如果是异步调用，给这次请求加一个唯一id
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

        AsyncRpcResult asyncResult;
        try {
            //调用子类的doInvoke()方法
            asyncResult = (AsyncRpcResult) doInvoke(invocation);
        } catch (InvocationTargetException e) { // biz exception
            //异常处理
            ...
        } catch (RpcException e) {
            //异常处理
            ...
        } catch (Throwable e) {
            //异常处理
            asyncResult = AsyncRpcResult.newDefaultAsyncResult(null, e, invocation);
        }
        RpcContext.getContext().setFuture(new FutureAdapter(asyncResult.getResponseFuture()));
        return asyncResult;
    }
```

`AbstractInvoker`最终调用了`DubboInvoker`的`doInvoke()`方法。

```java
@Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(PATH_KEY, getUrl().getPath());
        inv.setAttachment(VERSION_KEY, version);
        //获取Dubbo协议的exchangeClient
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodPositiveParameter(methodName, TIMEOUT_KEY, DEFAULT_TIMEOUT);
            //如果Oneway调用，即Consumer端不关心调用是否成功，则发送请求后直接返回结果。多用在日志发送这种可以容忍数据丢失的场景
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                return AsyncRpcResult.newDefaultAsyncResult(invocation);
            } else {
                //2.7之后所有调用都改成异步，将Future放入result中，如果Consumer调用是同步的，上面的Protocol的refer()会阻塞等待异步结果返回
                ExecutorService executor = getCallbackExecutor(getUrl(), inv);
                CompletableFuture<AppResponse> appResponseFuture =
                        currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);
                // save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
                FutureContext.getContext().setCompatibleFuture(appResponseFuture);
                AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
                result.setExecutor(executor);
                return result;
            }
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```

#### 注册中心的URL

注册中心url格式类似：`registry://localhost:2181?refer=version%3f1.0.0`，所以dubbo可以基于url找到对应的Protocol类为ERegistoryProtocol：

```java
@Override
    @SuppressWarnings("unchecked")
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        //1. 转换成具体注册中心实现的url
        url = getRegistryUrl(url);
        //2. 获取注册中心实现
        Registry registry = registryFactory.getRegistry(url);
        //3. 如果是获取RegistryService的代理，则直接获取本地暴露的invoker
        if (RegistryService.class.equals(type)) {
            return proxyFactory.getInvoker((T) registry, type, url);
        }

        //4. 判断url是否指定了分组信息
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(REFER_KEY));
        String group = qs.get(GROUP_KEY);
        if (group != null && group.length() > 0) {
            if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
                //指定了分组，则使用MergeableCluster
                return doRefer(getMergeableCluster(), registry, type, url);
            }
        }
        //5. 获取Cluster Invoker
        return doRefer(cluster, registry, type, url);
    }
```

1. 首先需要将url转换成真是注册中心的地址。dubbo是支持多注册中心的，二配置中获取的是一个通用的注册中心url，以registory://开头，这一步转成真正的注册中心url，例如：`zookeeper://127.0.0.1/org.apache.dubbo.registry.RegistryService?refer=interface%3Dorg.apache.dubbo.demo.DemoService`
2. 根据真实的url获取注册到注册中心的实现类，比如上面的url获取到的就是使用zookeeper注册中心，获取的就是`ZookeeperRegistry`
3. ...
4. dubbo支持将多个远程服务调用结果做合并，来作为最终处理结果，通过配置一个merger类实现。
5. 没有指定group的话，则使用默认的Cluster构造Invoker

上面方法的主要目的就是获取Registry的实现，然后调用其doRefer()方法：

```java
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        //1. 构建directory实例
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        // 2. 生成consumer URL
        Map<String, String> parameters = new HashMap<String, String>(directory.getConsumerUrl().getParameters());
        URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
        //3. 将consumer信息写入注册中心
        if (directory.isShouldRegister()) {
            directory.setRegisteredConsumerUrl(subscribeUrl);
            registry.register(directory.getRegisteredConsumerUrl());
        }
        //4. 构建RouteChain
        directory.buildRouterChain(subscribeUrl);
        //5. 订阅服务变化通知
        directory.subscribe(toSubscribeUrl(subscribeUrl));
        //6. 生成ClusterInvoker
        Invoker<T> invoker = cluster.join(directory);
        List<RegistryProtocolListener> listeners = findRegistryProtocolListeners(url);
        if (CollectionUtils.isEmpty(listeners)) {
            return invoker;
        }
        //7. 回调Listener
        RegistryInvokerWrapper<T> registryInvokerWrapper = new RegistryInvokerWrapper<>(directory, cluster, invoker, subscribeUrl);
        for (RegistryProtocolListener listener : listeners) {
            listener.onRefer(this, registryInvokerWrapper);
        }
        return registryInvokerWrapper;
    }
```

上面的doRefer()方法中，首先为服务生成`RegistryDirectory`实例，该类的作用是关联Directory和Registry接口；随后，Consumer会将自己也注册到注册中心，所以可以通过注册中心的数据看到某个Provider都被谁消费，也可以看到某个Consumer都调用了哪些服务。

第5步中，订阅注册中心数据变化，在provider变化时，可以收到通知。

第6步中，生成最终ClusterInvoker，Dubbo默认配置中，这里的Cluster是`FailoverCluster`，join方法返回`FailoverClusterInvoker`。

##### ClusterInvoker实现

`ClusterInvoker`是Dubbo支持集群调用的核心实现，包括负载均衡、特殊路由、容错处理等。默认实现类`FailoverClusterInvoker`支持用户配置重试次数，可以在一个节点失败重试其它节点。
 AbstractClusterInvoker：

```java
@Override
    public Result invoke(final Invocation invocation) throws RpcException {
        //判断Invoker是否已经destroy，是则抛出异常
        checkWhetherDestroyed();
        // 将attachments加到Invocation中
        Map<String, Object> contextAttachments = RpcContext.getContext().getObjectAttachments();
        if (contextAttachments != null && contextAttachments.size() != 0) {
            ((RpcInvocation) invocation).addObjectAttachments(contextAttachments);
        }
       // 获取可用invoker列表
        List<Invoker<T>> invokers = list(invocation);
       //根据配置获取指定的负载均衡实现
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        return doInvoke(invocation, invokers, loadbalance);
    }
```

`ClusterInvoker`的invoke()方法首先调用list()方法获取所有可用invoker列表，这里的是直接调用的Directory的list方法，Directory缓存了从注册中心获取的provider url列表，会将每个url生成invoker。
 在获取到一组invoker后需要从其中选择一个发起调用，这时候就需要用到负载均衡，最终根据获取的invoker列表和负载均衡器调用子类的具体实现。
 FailoverClusterInvoker：

```java
@Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        List<Invoker<T>> copyInvokers = invokers;
        checkInvokers(copyInvokers, invocation);
        String methodName = RpcUtils.getMethodName(invocation);
        // 获取重试次数，最低可配置在方法粒度
        int len = getUrl().getMethodParameter(methodName, RETRIES_KEY, DEFAULT_RETRIES) + 1;
        if (len <= 0) {
            len = 1;
        }
        // retry loop.
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {
            //Reselect before retry to avoid a change of candidate `invokers`.
            //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
            if (i > 0) {
                checkWhetherDestroyed();
                copyInvokers = list(invocation);
                // check again
                checkInvokers(copyInvokers, invocation);
            }
            // 使用负载均衡最终选择一个invoker
            Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
            invoked.add(invoker);
            RpcContext.getContext().setInvokers((List) invoked);
            try {
                Result result = invoker.invoke(invocation);
                if (le != null && logger.isWarnEnabled()) {
                    logger.warn(...);
                }
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
        throw new RpcException(...);
    }
```

上面的逻辑主要是两点：一根筋配置的重试次数来决定是否重试；二根据负载均衡实现从注册中心返回的可用服务中选择其中一个，然后发起调用。当重试结束还未成功，则抛出异常。

#### 构造带Filter的Invoker

上面讲了两种Invoker的获取和invoke的工作原理，其实Dubbo中上面得到的Invoker不会直接返回给Proxy，而是需要和Filter集成最终返回Invoker链。

### 总结

消费端的Proxy通过Invoker发起调用，Invoker对Proxy屏蔽了集群和服务治理等一系列逻辑，同时从Invoker层开始，提供了对多协议的哦支持。从Invoker再往后走，将不存在接口和方法的概念，而是传输层。