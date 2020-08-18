### Dubbo生产环境应用

------

##### 元数据中心

<font color="#8B0000">元数据</font>：描述数据的数据。在服务治理中，例如服务接口名，重试次数，版本号等等都可以理解为元数据。生产者端注册 30+ 参数，有接近一半是不需要作为注册中心进行传递；消费者端注册 25+ 参数，只有个别需要传递给注册中心。以前所有元数据都放在URL上进行相互传递，增加网络开销，影响性能。Dubbo提供了zookeeper，redis(推荐)的默认支持。

1. 引入，支持redis(推荐)、zookeeper：

   ~~~xml
   <dependency>
       <groupId>org.apache.dubbo</groupId>
       <artifactId>dubbo-metadata-report-redis|zookeeper</artifactId>
   </dependency>
   ~~~

2. 参数：

   ~~~properties
   dubbo.metadata-report.address=redis://127.0.0.1:2181
   dubbo.metadata-report.username=xxx        ##非必须
   dubbo.metadata-report.password=xxx        ##非必须
   dubbo.metadata-report.retry-times=30       ##非必须,default值100 失败重试次数
   dubbo.metadata-report.retry-period=5000    ##非必须,default值3000ms 重试周期
   dubbo.metadata-report.cycle-report=false   ##非必须,default值true 定时刷新
   ~~~

##### 配置中心

<font color="#8B0000">外部化配置</font>：可以简单理解为`dubbo.properties`的外部化存储，配置中心更适合将一些公共配置如注册中心、元数据中心配置等抽取以便做集中管理。

- 优先级：外部化配置默认较本地配置有更高的优先级，因此这里配置的内容会覆盖本地配置值。

- 作用域：外部化配置有全局和应用两个级别，全局配置是所有应用共享的，应用级配置是由每个应用自己维护且只对自身可见的。

- 使用，目前Nacos、Apollo、Zookeeper都支持作为配置中心，后两者支持作用域。

  ~~~properties
  dubbo.config-center.address=zookeeper://127.0.0.1:2181
  ~~~

##### 注册中心

<font color="#8B0000">服务发现</font>：Dubbo主要使用zookeeper、Nacos组件作为注册中心

- 使用

  1. 引入依赖

     ~~~xml
     <dependency>
         <groupId>org.apache.zookeeper</groupId>
         <artifactId>zookeeper</artifactId>
         <version>3.3.3</version>
     </dependency>
     <dependency>
         <groupId>com.netflix.curator</groupId>
         <artifactId>curator-framework</artifactId>
         <version>1.1.10</version>
     </dependency>
     ~~~

  2. 设置地址

     ~~~xml
     <!--集群注册中心-->
     <dubbo:registry protocol="zookeeper" simplified="true" address="10.20.153.10:2181,10.20.153.11:2181,10.20.153.12:2181" />
     <!--分组注册中心-->
     <dubbo:registry id="chinaRegistry" protocol="zookeeper" address="10.20.153.10:2181" group="china" />
     <dubbo:registry id="intlRegistry" protocol="zookeeper" address="10.20.153.10:2181" group="intl" />
     ~~~

- 注意：在设置注册中心地址时，需设置<font color="red"> simplified=“true”</font>（注册到注册中心的URL是否采用精简模式）。

##### 协议

- <font color="#8B0000">dubbo://</font> 缺省协议采用单一长连接和 NIO 异步通讯，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况。反之，不适合传送大数据量的服务，比如传文件，传视频等，除非请求量很低。
  - 特性：单连接、长连接、TCP、NIO异步传输、Hessian二进制序列化，适用<font color="#1E90FF">`参数数据包较小（建议小于100K），消费者比提供者个数多，单一消费者无法压满提供者，尽量不要用 dubbo 协议传输大文件或超大字符串。`</font>
  - 约束：参数及返回值需实现 Serializable 接口

- <font color="#8B0000">hessian://</font> 用于集成Hessian的服务，底层采用Http通讯，采用 Servlet 暴露服务，Dubbo 缺省内嵌 Jetty 作为服务器实现。

  - 特性：多链接、短连接、Http、同步传输、Hessian二进制序列化，适用<font color="#1E90FF">`页面传输，文件传输，或与原生hessian服务互操作。`</font>

  - 约束：参数及返回值需实现 Serializable 接口

    ~~~xml
    <dependency>
        <groupId>com.caucho</groupId>
        <artifactId>hessian</artifactId>
        <version>4.0.7</version>
    </dependency>
    ~~~

##### 序列化

Dubbo默认适用Hessian序列化，我们可以通过设置Kryo、FST序列化方式提升性能。

步骤：

1. 将需要被序列化的Bean注册到dubbo系统中

   ~~~java
   public class SerializationOptimizerImpl implements SerializationOptimizer {
       public Collection<Class> getSerializableClasses() {
           List<Class> classes = new LinkedList<Class>();
           classes.add(BidRequest.class);
           classes.add(BidResponse.class);
           classes.add(Device.class);
           classes.add(Geo.class);
           classes.add(Impression.class);
           classes.add(SeatBid.class);
           return classes;
       }
   }
   ~~~

2. 在协议上设置序列化方式

   ~~~xml
   <dubbo:protocol name="dubbo" serialization="kryo" optimizer="org.apache.dubbo.demo.SerializationOptimizerImpl"/>
   ~~~

   `由于注册被序列化的类仅仅是出于性能优化的目的，所以即使你忘记注册某些类也没有关系。事实上，即使不注册类，Kryo和FST的性能依然普遍优于hessian和dubbo序列化。`

##### 回声测试

检测服务是否可用，回声测试按照正常请求流程执行，能够测试整个调用是否通畅，可用于监控。所有服务自动实现 `EchoService` 接口，只需将任意服务引用强制转型为 `EchoService`，即可使用。

`<dubbo:reference id="memberService" interface="com.xxx.MemberService" />`

~~~java
// 远程服务引用，强制转型为EchoService
EchoService echoService = (EchoService) ctx.getBean("memberService");
// 回声测试可用性
String status = echoService.$echo("OK"); 
assert(status.equals("OK"));
~~~

##### 多协议

Dubbo 允许配置多协议，在不同服务上支持不同协议或者同一服务上同时支持多种协议。不同服务在性能上适用不同协议进行传输，比如大数据用短连接协议，小数据大并发用长连接协议。

~~~xml
<dubbo:application name="world"  />
<!-- 多协议配置 -->
<dubbo:protocol name="dubbo" port="20880" />
<dubbo:protocol name="hessian" port="8080" />
<!-- 多协议配置 方式二直接在服务上配置支持多个协议-->
<dubbo:service id="helloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" protocol="dubbo,hessian" />
~~~

##### 多注册中心

Dubbo 支持同一服务向多注册中心同时注册，或者不同服务分别注册到不同的注册中心上去，甚至可以同时引用注册在不同注册中心上的同名服务。

~~~xml
<!-- 多注册中心配置 -->
    <dubbo:registry id="hangzhouRegistry" address="10.20.141.150:9090" />
    <dubbo:registry id="qingdaoRegistry" address="10.20.141.151:9010" default="false" />
    <!-- 向多个注册中心注册 -->
    <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" registry="hangzhouRegistry,qingdaoRegistry" />
~~~

##### 服务分组

当一个接口有多种实现时，可以用group区分。

~~~xml
<dubbo:service group="feedback" interface="com.xxx.IndexService" />
<dubbo:service group="member" interface="com.xxx.IndexService" />
<!--当消费端不设置group值时，代表调用任意组的服务-->
<dubbo:reference id="barService" interface="com.foo.BarService" group="*" />
~~~

##### 多版本

当一个接口实现，出现不兼容升级时，可以用版本号过渡，版本号不同的服务相互间不引用。迁移步骤：

1. 在低压力时间段，先升级一半提供者为新版本
2. 再将所有消费者升级为新版本
3. 然后将剩下的一半提供者升级为新版本

~~~xml
<dubbo:service interface="com.foo.BarService" version="1.0.0" />
<!--当消费端version等于*时，代表不区分版本-->
<dubbo:reference id="barService" interface="com.foo.BarService" version="1.0.0" />
~~~

##### 令牌验证

通过令牌验证在注册中心控制权限，以决定要不要下发令牌给消费者，可以防止消费者绕过注册中心访问提供者，另外通过注册中心可灵活改变授权方式，而不需修改或升级提供者。支持固定密码或true。

~~~xml
<!--随机token令牌，使用UUID生成。全局模式-->
<dubbo:provider interface="com.foo.BarService" token="true" />
<!--固定token令牌，相当于密码。服务级别-->
<dubbo:service interface="com.foo.BarService" token="123456" />
~~~

##### 容错负载

- 提供多个相同服务：服务名称一致，URL不同(IP或端口)。

- 负载均衡策略：<font color="#1E90FF">`Random`</font> <font color="#1E90FF">`RoundRobin`</font> <font color="#1E90FF">`LeastActive`</font> <font color="#1E90FF">`ConsistentHash`</font>；配置：loadbalance="roundrobin" weight=”100“。

- 容错，Dubbo提供了多种集群容错模式，cluster=“failover”：

  - <font color="#1E90FF">`FailoverClusterInvoker`</font>(默认)：失败重试，调用服务失败后自动切换到其它服务提供者服务器进行重试。默认重试次数retries="2" 。

  - <font color="#1E90FF">`FailfastClusterInvoker`</font>：快速失败，调用失败立即抛出异常RpcException。

  - <font color="#1E90FF">`FailsafeClusterInvoker`</font>：失败安全，直接忽略异常，并记录错误日志

    ```java
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
            try {
                checkInvokers(invokers, invocation);
                Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
                return invoker.invoke(invocation);
            } catch (Throwable e) {
                logger.error("Failsafe ignore exception: " + e.getMessage(), e);
                return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); // ignore
            }
        }
    ```

  - <font color="#1E90FF">`FailbackClusterInvoker`</font>：失败自动恢复，记录失败请求，并按照策略后期再重试。这种模式通常用于消息通知操作。

    ~~~java
    protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
            Invoker<T> invoker = null;
            try {
                checkInvokers(invokers, invocation);
                invoker = select(loadbalance, invocation, invokers, null);
                return invoker.invoke(invocation);
            } catch (Throwable e) {
                logger.error("Failback to invoke method " + invocation.getMethodName() + ", wait for retry in background. Ignored exception: "
                        + e.getMessage() + ", ", e);
                // 记录失败请求，并按照策略后期再重试
                addFailed(loadbalance, invocation, invokers, invoker);
                return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); // ignore
            }
        }
    ~~~

  - <font color="#1E90FF">`ForkingClusterInvoker`</font>：并行调用，同时调用多个服务，只要一个成功即返回。通常用于实时性要求较高，不惜浪费资源的场景。 设置最大并行数forks="2"

  - <font color="#1E90FF">`BroadcastClusterInvoker`</font>：广播模式，逐个调用，任意一台报错则报错。

##### 熔断降级