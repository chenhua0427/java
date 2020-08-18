##### 使用

- 核心库：不依赖任何框架/库，能够运行于 Java 7 及以上的版本的运行时环境，同时对 Dubbo / Spring Cloud 等框架也有较好的支持

  1. 应用程序引入依赖

     ~~~xml
     <dependency>
         <groupId>com.alibaba.csp</groupId>
         <artifactId>sentinel-core</artifactId>
         <version>1.7.2</version>
     </dependency>
     ~~~

  2. 定义资源

     ~~~java
     @SentinelResource("HelloWorld")
     public void helloWorld() {
         System.out.println("hello world");
     }
     ~~~

  3. 定义规则

     ```java
     private static void initFlowRules(){
         List<FlowRule> rules = new ArrayList<>();
         FlowRule rule = new FlowRule();
         rule.setResource("HelloWorld");
         rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
         // Set limit QPS to 20.
         rule.setCount(20);
         rules.add(rule);
         FlowRuleManager.loadRules(rules);
     }
     // 示例二
     public static void main(String[] args) {
         // 配置规则.
         initFlowRules();
     
         while (true) {
             // 1.5.0 版本开始可以直接利用 try-with-resources 特性，自动 exit entry
             try (Entry entry = SphU.entry("HelloWorld")) {
                 // 被保护的逻辑
                 System.out.println("hello world");
     		} catch (BlockException ex) {
                 // 处理被流控的逻辑
     	    	System.out.println("blocked!");
     		}
         }
     }
     ```

- 控制台(Dashboard)：负责管理推送规则、监控、集群限流分配管理、机器发现等

  1. 下载控制台jar包并在本地启动

  2. 客户端接入控制台

     1. 引入transport模块与dashboard通信

        ```xml
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-transport-simple-http</artifactId>
            <version>1.7.2</version>
        </dependency>
        ```

     2. 客户端启动时在jvm参数中指定控制台地址和端口

        ~~~yml
        -Dcsp.sentinel.api.port=port #客户端的prot用于上报相关信息（默认为 8719）
        -Dcsp.sentinel.dashboard.server=consoleIp:port #控制台的地址
        -Dproject.name= #应用名称，会在控制台天使
        # 一个provider的启动参数示例：
        -Djava.net.preferIPv4Stack=true -Dcsp.sentinel.api.port=8720 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=dubbo-provider-demo
        ~~~

        拦截日志统一记录在 `~/logs/csp/sentinel-block.log`中。

##### 控制

- 流量控制（FlowRule）

  同一个资源可以同时有多个限流规则，检测规则时会依次检查。

  | 属性            | 说明                                                         | 默认值                      |
  | --------------- | ------------------------------------------------------------ | --------------------------- |
  | resouce         | 资源名称，被控制的对象                                       |                             |
  | count           | 限流阈值                                                     |                             |
  | grade           | 限流类型：QPS模式/并发线程数模式                             | RuleConstant.FLOW_GRADE_QPS |
  | limitApp        | 针对的调用来源                                               | default，表示不限制         |
  | strategy        | 调用关系限流策略：直接、链路、关联                           | 根据资源本身（直接）        |
  | controlBehavior | 流控效果（直接拒绝/WarmUp/匀速+排队等待），不支持按调用关系限流 | 直接拒绝                    |
  | clusterMode     | 是否集群限流                                                 | 否                          |

- 熔断降级 (DegradeRule)

  同一个资源可以同时有多个降级规则

  | 属性                | 说明                                                         | 默认值      |
  | ------------------- | ------------------------------------------------------------ | ----------- |
  | resource            | 资源名称                                                     |             |
  | count               | 阈值。单位：ms、[0.0, 1.0]代表0% - 100%、次。                |             |
  | grade               | 熔断策略：秒级RT/秒级异常比例/分钟级异常数                   | 秒级平均 RT |
  | timeWindow          | 降级的时间，单位为秒                                         |             |
  | rtSlowRequestAmount | RT 模式下 1 秒内连续多少个请求的平均 RT 超出阈值方可触发熔断（1.7.0 引入） | 5           |
  | minRequestAmount    | 异常熔断的触发最小请求数，请求数小于该值时即使异常比率超出阈值也不会熔断（1.7.0 引入） | 5           |

  RuleConstant.DEGRADE_GRADE_RT：当资源的平均响应时间超过阈值(count ms)后，资源进入准降级状态。接下来如果持续进入5个请求，他们的RT都持续超过这个阈值，那么在timeWindow时间内，对这个资源的调用都会自动返回（自动熔断，抛出 `DegradeException`）。在下一个timeWindow到来时，会接着再放入5个请求，重复上面的判断。

  DEGRADE_GRADE_EXCEPTION_RATIO：当资源的每秒异常总数占通过量的比值超过阈值（`DegradeRule` 中的 `count`）之后，资源进入降级状态。

  DEGRADE_GRADE_EXCEPTION_COUNT：当资源近 1 分钟的异常数目超过阈值之后会进行熔断。

- 系统保护

- 访问控制

- 热点规则

##### 持久化

接入 Sentinel Dashboard 后，在页面上操作来更新规则，都无法避免一个问题，那就是服务重新后，规则就丢失了，因为默认情况下规则是保存在内存中的。sentinel中持久化的方式有两种，pull模式和push模式。

- pull模式是指站在第三方持久化系统(redis, nacos)的角度，他们去到sentinel中定时去拉去配置信息，可能会造成数据的不一致性。
- push模式是站在sentinel的角度，将其配置信息主动推送给第三方持久化系统，sentinel官方也推荐在线上使用该模式。