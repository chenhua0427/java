## Feign原理

#### 简述

Feign是一个声明式的Web Service客户端，它使得编写Web Service客户端更为容易。通过对使用@FeignClient注解的接口生成远程代理JDK Proxy，jdk动态代理需要实现InvocationHandler接口，Fegin提供了两种实现；Proxy调用`InvocationHandler.invoke()`方法执行远程调用，Fegin在此处做了扩展，支持不同方法使用不同方法处理器MethodHandler，InvocationHandler内部维护了一个`map<method,MethodHandler>`，InvocationHandler.invoke()动态调用MethodHandler的invoke方法完成实际的URL请求，然后返回解码后的响应结果；而`MethodHandler.invoke()`方法中执行URL请求的Client也做了多种扩展实现。

~~~
RequestTemplate中包含请求的所有信息，如请求参数，请求URL等。在MethodHandler中被创建，并根据它构建出request作为参数递给Client执行请求。
~~~

最终的调用链：<font color=" #1E90FF">Proxy -> InvocationHandler.invoke() -> MethodHandler.invoke() -> Client.execute()</font>

~~~java
/*OpenFeign*/在Feign的基础上增加了MVC的注解支持，包含了ribbon组件自带负载均衡功能。
~~~

##### Proxy

远程代理，基于JDK动态代理。

##### InvocationHandler

基于JDK的动态代理，需要实现该接口。Feign提供了两种实现<font color="#8B0000">`FeignInvocationHandler`</font>，如果结合了Hystrix则是<font color="#8B0000">`HystrixInvocationHandler`</font>

~~~java
static class FeignInvocationHandler implements InvocationHandler {
    // 保存了远程接口方法和本地方法处理类的映射
    private final Map<Method, MethodHandler> dispatch;

    FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
      this.target = checkNotNull(target, "target");
      this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      // 如果是Object上的方法hashcode、toString()、equals则调用本地实现
        ...
      // 根据方法名key获取到对应handler，并执行invoke()方法，完成实际的HTTP请求和结果的处理。
      return dispatch.get(method).invoke(args);
    }
  }
~~~

~~~java
final class HystrixInvocationHandler implements InvocationHandler {
     private final Map<Method, MethodHandler> dispatch;
     public Object invoke(final Object proxy, final Method method, final Object[] args)
      throws Throwable {

        HystrixCommand<Object> hystrixCommand =
        new HystrixCommand<Object>(setterMethodMap.get(method)) {
          @Override
          protected Object run() throws Exception {
            try {
              return HystrixInvocationHandler.this.dispatch.get(method).invoke(args);
            } catch (Exception e) {
              throw e;
            } catch (Throwable t) {
              throw (Error) t;
            }
          }

          @Override
          protected Object getFallback() {
            if (fallbackFactory == null) {
              return super.getFallback();
            }
            try {
              Object fallback = fallbackFactory.create(getExecutionException());
              Object result = fallbackMethodMap.get(method).invoke(fallback, args);
              if (isReturnsHystrixCommand(method)) {
                return ((HystrixCommand) result).execute();
              } else if (isReturnsObservable(method)) {
                // Create a cold Observable
                return ((Observable) result).toBlocking().first();
              } else if (isReturnsSingle(method)) {
                // Create a cold Observable as a Single
                return ((Single) result).toObservable().toBlocking().first();
              } else if (isReturnsCompletable(method)) {
                ((Completable) result).await();
                return null;
              } else if (isReturnsCompletableFuture(method)) {
                return ((Future) result).get();
              } else {
                return result;
              }
            } catch (IllegalAccessException e) {
              // shouldn't happen as method is public due to being an interface
              throw new AssertionError(e);
            } catch (InvocationTargetException | ExecutionException e) {
              // Exceptions on fallback are tossed by Hystrix
              throw new AssertionError(e.getCause());
            } catch (InterruptedException e) {
              // Exceptions on fallback are tossed by Hystrix
              Thread.currentThread().interrupt();
              throw new AssertionError(e.getCause());
            }
          }
        };

    if (Util.isDefault(method)) {
      return hystrixCommand.execute();
    } else if (isReturnsHystrixCommand(method)) {
      return hystrixCommand;
    } else if (isReturnsObservable(method)) {
      // Create a cold Observable
      return hystrixCommand.toObservable();
    } else if (isReturnsSingle(method)) {
      // Create a cold Observable as a Single
      return hystrixCommand.toObservable().toSingle();
    } else if (isReturnsCompletable(method)) {
      return hystrixCommand.toObservable().toCompletable();
    } else if (isReturnsCompletableFuture(method)) {
      return new ObservableCompletableFuture<>(hystrixCommand);
    }
    return hystrixCommand.execute();
  }
}
~~~



##### MethodHandler

MethodHandler接口只有一个方法invoke()，主要职责是完成实际远程URL请求，然后返回解码后的响应结果。默认实现类<font color="#8B0000">`SynchronousMethodHandler`</font>

~~~java
final class SynchronousMethodHandler implements MethodHandler {
    @Override
  public Object invoke(Object[] argv) throws Throwable {
      // 1.生成远程URL请求实例 request；
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Options options = findOptions(argv);
    while (true) {
      try {
        return executeAndDecode(template, options);  //远程调用，并将返回结果解码
      } catch (RetryableException e) {
      }
    }
  }
  // 执行请求，然后解码结果
  Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
    Request request = targetRequest(template);

    Response response;
    try {
      response = client.execute(request, options); // 执行http调用
      response = response.toBuilder()
          .request(request)
          .requestTemplate(template)
          .build();
    } 
    if (decoder != null)
      return decoder.decode(response, metadata.returnType());
  }
}
~~~

##### Client

客户端组件是Feign中一个非常重要的组件，负责端到端的执行URL请求。其核心逻辑是发送request请求到服务器，并接收response响应后并进行解码。

~~~java
public interface Client {
    Response execute(Request request, Options options) throws IOException;
}
~~~

根据内部完成HTTP请求的组件和技术不同，有以下几个实现：

- Client.Default类：默认的客户端实现类，内部使用<font color="#8B0000">`HttpURLConnnection`</font>完成URL请求处理；<font color="red">性能低</font>
- ApacheHttpClient类：内部使用<font color="#8B0000">`Apache httpclient `</font>开源组件；<font color="red">带有连接池功能，具备优秀的HTTP连接复用能力。</font>使用需通过Maven依赖。
- <font color=" #1E90FF">OkHttpClient</font>类：内部使用OkHttp3开源组件；支持SPDY协议（<font color="red">SPDY是Google开发的基于TCP的传输层协议</font>，用以最小化网络延迟，提升网络速度，优化用户的网络使用体验。）
- <font color=" #1E90FF">LoadBalancerFeignClient</font>类：内部使用Ribben负载均衡技术完成URL请求处理；简单的使用了delegate包装代理模式：Ribben组件计算出合适的server后，内部包装delegate代理客户端完成http请求；所封装的delegate客户端代理实例的类型可以自由使用上面三种URL请求处理类；即<font color="red">封装Ribben组件实现负载均衡功能，并调用高性能的URL请求处理类。</font>

### 使用

##### 在consumer端：

1. 在Maven中引入Feign：`spring-cloud-starter-openfeign`，并在启动类上注解`@EnableFeignClients`

   ```xml
    <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
   ```

2. 定义接口，里面包含了需要远程调用的方法；对接口和方法添加注解(支持MVC的注解)：

   ~~~java
   @FeignClient("stores")
   public interface StoreClient {
       @RequestMapping(method = RequestMethod.GET, value = "/stores")
       List<Store> getStores();
   
       @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
       Store update(@PathVariable("storeId") Long storeId, Store store);
   }
   ~~~

   

3. 在应用类上，添加接口属性，并使用@Autowired注入

   ~~~java
   public class Demo{
       @Autowired
       private StoreClient storeClient;
       
       private void Test(){
           List<Store> store = storeClient.getStores();
       }
   }
   ~~~

##### 常用注解：

- @FeignClient属性

  - name feign客户端名称，支持占位符${}。

  - value服务名称。

  - url全路径地址或hostname，http或https可选。

  - decode404 设置响应状态码404时，是否抛出FeignExceptions异常。

  - configuration 自定义当前feign client的配置，设置自定义配置类。

    ```java
    @FeignClient(configuration = FeignConfiguration.class)
    ```

  - fallback 熔断机制，调用失败时走的回退方法(抛出异常、返回默认数据等)；依赖@EnableHystrix。

  - path 自动给所有方法derequestMapping加上前缀。

  - primary

- @SpringQueryMap 注解方法参数，将pojo或map参数注释为查询参数映射。

  ~~~java
  @FeignClient("demo")
  public class DemoTemplate {
      @GetMapping(path = "/demo")
      String demoEndpoint(@SpringQueryMap Params params);
  }
  ~~~

- @GetMapping、@RequestMapping 设置请求路径及参数

- @PathVariable 将请求URL中的模板变量映射到功能处理方法的参数上，url中{}内名称必须和参数名称一致。

##### 配置

Feign提供了两大类配置属性来配置上述三种HTTP客户端，<font color="#8B0000">`feign.client.*`</font>和<font color="#8B0000">`feign.httpclient.*`</font>，前者支持按实例进行配置，后者全局共享一套配置，包含线程池配置，但只影响HttpClient和OkHttp，不影响HttpURLConnection。

~~~
所谓按实例进行配置，就是指每个FeignClient实例都可以通过feign.client.<feignClientName>.*来单独进行配置，注意首字母小写。而feign.client.default.*表示默认配置。
~~~

- feign.client.config：；配置类。
- feign.client.default-config：default；。
- feign.compression.request.enabled：false；配置请求GZIP压缩。
- feign.compression.request.mime-types：[text/xml, application/xml, application/json]；
- feign.compression.request.min-request-size：2048；压缩数据大小的下限。
- feign.compression.response.enabled：false；配置响应GZIP压缩。
- feign.compression.response.useGzipDecoder：false；启用默认的gzip解码器。
- feign.httpclient.connection-timeout：2000；连接超时时间。
- feign.httpclient.connection-timer-repeat：3000；
- feign.httpclient.disable-ssl-validation：false；
- feign.httpclient.max-connections：200；线程池最大连接数（全局）。
- feign.httpclient.max-connections-per-route：50；线程池最大连接数（单个HOST）。
- feign.httpclient.time-to-live：900；线程存活时间(单位：秒)。
- feign.httpclient.time-to-live-unit：
- feign.hystrix.enabled：false；开启(true)/关闭(false)Hystrix功能。
- feign.okhttp.enabled：false；开启(true)/关闭(false) OK HTTP请求方式。
- feign.httpclient.enabled：true；开启(true)/关闭(false) Apache HTTP请求方式。
- feign.<font color=" #1E90FF">sentinel</font>.enabled:false;激活sentinel

##### 日志

```java
/**
 * Feign配置
 * 该配置放到SpringBoot可以扫描到的路径下
 */
@Configuration
public class FeignConfig {

    @Bean
    Logger.Level feignLevel() {
        return Logger.Level.FULL;
    }
}
```

Logger.Level枚举

- NONE：不输出日志
- BASIC：输出请求方法、url、响应状态码、执行时间
- HEADERS：基本信息以及请求和响应头
- FULL：全部输出。

##### 修改openfeign的负载均衡策略

openfeign默认使用轮询的负载均衡算法，通过在配置文件中修改：

~~~yml
SPRINGCLOUD-PROVIDER-DEPT: #服务名
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
~~~

常用的负载均衡算法：

- RoundRobinRule 轮询
- RandomRule 随机
- RetryRule 轮询，失败后重试
- WeightedResponseTimeRule 响应权重(响应越快，权重越高)
- BestAvailableRule 过滤故障，选择并发最小的服务
- ...