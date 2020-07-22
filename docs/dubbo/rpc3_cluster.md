## 白话Dubbo（三）：走向集群

#### 服务集群

一个系统从本地调用转成远程调用，原有有多种。最主要的两个：一是功能解耦；一是高可用。为了解决高可用问题，通常在服务部署的时候，会部署多套，以防止单个服务压力过大或网络分区等单点故障导致服务不可用，即集群。

##### 路由

当服务提供方部署成集群的时候，问题来了，原来只有一个机器提供服务的时候，调用方只需知道ip+prot就可以把请求发过去。现在提供方百年城了一个ip+prot的列表，每次调用前都要从中选择一个，显然这不是用户想要关心的，增加一个路由模块，集成到Consumer里便顺理成章。

##### 服务发现

对于路由模块来说，后端的机器配置是不断变化的，比如节点上下线，ip端口的变化等。因为服务节点不知道路由模块的存在，所以双方要有一个公共的地方，服务节点变化时更新数据，路由模块通过通知或轮询拉取变化。这个模块就是注册中心，常见的注册中心有<u>zookeeper</u>、Eureka等。

#### Dubbo集群解决方案

之前讲到，消费者端Proxy通过url获取到一个远程调用的Invoker；Dubbo要做的就是Invoker提供路由功能，通过注册中心来获取真实的url列表。Dubbo的逻辑是，如果url时具体服务节点的url，比如dubbo://127.0.0.1:12345，那就正常走建立连接，发起调用；如果url时注册中心的，Dubbo通过替换成机器的Invoker来发起调用。

##### Cluster

~~~java
@SPI(FailoverCluster.NAME)
public interface Cluster {
    /**
     * Merge 从directory获取的invoker列表
     */
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;
}
~~~

Cluster接口通过将从directory获取的Invoker列表合并成要给ClusterInvoker。Directory时Dubbo对象对服务列表提供者的抽象，显然注册中心就是一种DIrectory的实现，更准确说时一种Directory数据源。

- **Cluster实现** Dubbo提供了很多Cluster实现，默认是FailoverCluster：

  ```java
  public class FailoverCluster extends AbstractCluster {
      public final static String NAME = "failover";
      @Override
      public <T> AbstractClusterInvoker<T> doJoin(Directory<T> directory) throws RpcException {
          return new FailoverClusterInvoker<>(directory);
      }
  }
  ```

  可以看到这里的实现就是创建一个具体的ClusterInvoker，将directory传给他。所有的ClusterInvoker实现都继承了AbstractClusterInvoker。

- **ClusterInvoker实现** AbstractClusterInvoker实现了Invoker接口，所以对Consumer端的Proxy是透明的。

  ~~~java
  public abstract class AbstractClusterInvoker<T> implements Invoker<T> {
          ...
         //从directory获取invoker列表
          List<Invoker<T>> invokers = list(invocation);
         // 初始化负载均衡
          LoadBalance loadbalance = initLoadBalance(invokers, invocation);
          // 子类实现调用
          return doInvoke(invocation, invokers, loadbalance);
      }
  }
  ~~~

  这里面除了从directory获取invoker列表外，最主要的就是初始化LoadBalance，然后发起调用。

  Dubbo中默认的ClusterInvoker实现就是前面FailoverCluster返回的FailoverClusterInvoker，这个实现包含重试逻辑，即调用一个节点失败的情况下会重试其它的。

##### 负载均衡

当ClustorInvoker从directory获取到多个后端服务invoker后，选择调用哪个invoker是由负责均衡策略决定，即上面的LoadBanlance接口。

~~~java
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {
    /**
     * select one invoker in list.
     */
    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
}
~~~

接口比较简单，根据Invocation信息，从invokers中选择一个。这个方法由ClusterInvoker负责调用。

##### 注册中心

Dubbo为了对接不同的注册中心，抽象出了Registory接口，定义如下：

~~~java
public interface Registry extends Node, RegistryService {}
~~~

这个接口只是简单的组合了Node和RegistoryService，组合Node的原因是Dubbo配置中将注册中心也是用url的形式配置的，比如：zookeeper://127.0.0.1:2081。主要接口都定义在RegistoryServier中：

~~~java
public interface RegistryService {
    /**
     *  注册
     */
    void register(URL url);
    /**
     * 注销
     */
    void unregister(URL url);
    /**
     * 订阅
     */
    void subscribe(URL url, NotifyListener listener);
    /**
     * 取消订阅
     */
    void unsubscribe(URL url, NotifyListener listener);
    /**
     * 主动查询
     */
    List<URL> lookup(URL url);
}
~~~

注册中心包含两类接口：一类注册、注销主要给Provider使用；一类订阅、取消订阅主要给Consumer使用。Dubbo默认实现了主流注册中心的对接，比如zookeeper、eureka等。

- **RegistoryDirectory** 上面讲到Cluster接口，当需要要给ClusterInvoker的时候，需要提供Directory参数。RegistoryDirectory相当于Directory的注册中心实现，它包含了要给注册中心，当注册中心数据变化后，刷新自身Invoker缓存。

  ~~~java
  public class RegistryDirectory<T> extends AbstractDirectory<T> implements NotifyListener {
      ...
      // 初始化时注入
      private Registry registry;
      ...
  }
  ~~~

##### 路由策略

服务路由功能中，除了可以通过负责均衡策略来干涉具体叫用的服务之外，通常需要加一些更加个性化的设置。比如，部分新上线的功能指向让符合一定条件的Cousmer使用，就可以设置根据用户id或者标签来决定请求发送到后端的那个提供方。

Dubbo中针对路由策略的接口是Router：

~~~java
public interface Router extends Comparable<Router> {
    int DEFAULT_PRIORITY = Integer.MAX_VALUE;
    /**
     * Get the router url.
     */
    URL getUrl();
    /**
     * 对于传入的invokers做Rute规则匹配，返回匹配上的invoker列表
     */
    <T> List<Invoker<T>> route(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
    /**
     * 接收invoker list变化通知
     */
    default <T> void notify(List<Invoker<T>> invokers) {

    }
    /**
     * Router 的优先级
     */
    int getPriority();
    ... 
}
~~~

接口中最主要的就是route方法，在Cluster从Directory获取到Invoker列表后，首先查询可用的Rout规则，并逐个匹配，只有符合条件的invoker才会交给Loadbalance再选择。

Dubbo默认提供两种实现，一种是基于条件表达式的路由规则设置，一种是基于脚本的路由规则设置。

#### 总结

![image](https://github.com/walle710/java/blob/master/docs/image/dubbo-rpc3.jpg)

当服务编程一个集群之后，情况复杂了很多，要让用户无感知的调用集群，需要将集群调用做成抽象，并对接注册中心。这也是Dubbo再Proxy之后又抽象出Invoker的原因。针对集群调用，内部实现了不同的容错策略，同时围绕Invoker还扩容了其它功能。