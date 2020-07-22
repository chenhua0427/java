## 白话Dubbo（四）：微服务治理

#### 前言

前面几篇文章结合rpc框架的基础需求，讲解了Dubbo中对于这些需求的抽象。Dubbo现在能够被如此多的线上应用所采用，跟最近几年微服务的广泛推广有很大的关系。微服务绝不仅仅是把服务拆小，改成远程调用这么简单，必须有配套的服务治理的功能，比如监控、熔断限流等。这篇文章就来分解下Dubbo是怎样来支持这些功能的。

##### Filter详解

之前讲到Dubbo对注册中心的支持就是通过抽象出一个Directory接口实现。相对来说，注册中心的职责功能是比较明确的，主流的注册中心实现对外的接口相对同一，区别再内部实现上。但对于微服务相关的其它功能，因需求个性化太强，很难做成这样的抽象。Dubbo是通过Filter来实现对扩展功能的支持的。

##### Filter原理

Consumer端所有的远程调用通过Invoker发起，而Dubbo通过再Invoker上加上Filter链，再调用前后可以添加扩展逻辑。同样，再服务提供端Invoker上也有一样的逻辑。

~~~java
@SPI
public interface Filter {
    /**
     * Make sure call invoker.invoke() in your implementation.
     */
    Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException;
}
~~~

Filter接口在Invoker被调用前后被调用，在添加自己的逻辑后，通过条用Invoker参数的invok方法，来将请求继续传下去。

##### Filter和Invoker集成

Filter要对用户无感知，就要将Filter和Invoker集成到一起，伪装成要给Invoker，这一点是通过`ProtocolFilterWrapper`类实现，这个类是Protocol的实现类。

在Dubbo中，Consumer通过DubboProtocol来获取Invoker的引用。其实，在DubboProtocol的外层还有一个装饰类ProtocolFilterWrapper来将Filter集成进去。

~~~java
public class ProtocolFilterWrapper implements Protocol {
    private final Protocol protocol;
    public ProtocolFilterWrapper(Protocol protocol) {
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }
   @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        if (UrlUtils.isRegistry(invoker.getUrl())) {
            return protocol.export(invoker);
        }
        return protocol.export(buildInvokerChain(invoker, SERVICE_FILTER_KEY, CommonConstants.PROVIDER));
    }

    @Override
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        if (UrlUtils.isRegistry(url)) {
            return protocol.refer(type, url);
        }
        return buildInvokerChain(protocol.refer(type, url), REFERENCE_FILTER_KEY, CommonConstants.CONSUMER);
    }
}
~~~

从上面的代码看到，export和refer都是调用封装的Protocol的方法，对于Dubbo协议，这个protocol就是一个DubboProtocol的实例。这里用了buildInvokerChain()方法来将Filter和元素的Invoker做了绑定。export()方法中，只会绑定用于Provider端的Filter，refer()方法中只会绑定用于Consumer端的Filter。

##### Filter链

```java
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
        Invoker<T> last = invoker;
        //加载所有可用的Filter
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);

        if (!filters.isEmpty()) {
            //从后往前连接
            for (int i = filters.size() - 1; i >= 0; i--) {
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;
                //将filter装饰成一个invoker
                last = new Invoker<T>() {
                    ...
                    ...
                    @Override
                    public Result invoke(Invocation invocation) throws RpcException {
                        Result asyncResult;
                        try {
                            //调用filter的invoker方法，filter完成自己的逻辑后必须调用next.invoke()
                            asyncResult = filter.invoke(next, invocation);
                        } catch (Exception e) {
                            ...
                            ...
                            throw e;
                        } finally {

                        }
                        return asyncResult.whenCompleteWithContext((r, t) -> {
                              ...
                              ...
                        });
                    }
                    ...
                };
            }
        }

        return last;
    }
```

Dubbo直接将Filter封装成一个Invoker，然后将多个Inovker连接在一起。对于Consumer端的代理来说，从protocol获取到的一个Inovker，调用invok()方法，实际上是调用的这个链条的header，然后header再把请求传递下去。这样就把FIlter的整个逻辑对Proxy透明化。

![](https://github.com/chenhua0427/java/tree/master/docs/images/dubbo-rpc4.jpg)

##### Filter列表

Duboo支持用户自定义Filter。以下是一些Dubbo自身几个比较重要的Filter：

​	**AccessLogFilter**-访问日志记录，默认写到文件中

​	**ActiveLimitFilter**-Consumer端限流，限制并发请求数

​	**ExecuteLimtFilter**-Provider端限流

​	**MonitorFilter**-Dubbo监控，将收集到的指标数据发给MonitorService

​	**MetricsFilter**-对接ali开源的Metric监控，是现在比较主流的监控指标手机上报方式

​	**GenericFilter**-泛化调用支持，多用于调用方没有服务提供方API的情况，只需要使用map传递参数就可以调用。典型应用如一个支持dubbo的job调度中心，不需要把api的jar上传就可以调用dubbo接口。

​	**TokenFilter**-token校验，用于限制接口的访问权限，只允许携带合法token的consumer调用。