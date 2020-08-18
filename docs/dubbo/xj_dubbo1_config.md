## 详解Dubbo（一）：消费端初始化

##### 配置类

Dubbo封装了一些列的Config类，继承于AbstractConfig。

常用的配置类如下：

- ApplicationConfig：配置整个应用的参数，比如应用名称、版本号。
- RegisterConfig：注册中心的配置参数，比如连接地址、用户名、密码等。
- PotocolConfig：针对某个协议的配置，比如Dubbo协议、可配置编码解码器、传输层选择(Netty、Http等)。
- ConsumerConfig：服务消费参数配置，对所有服务引用有效，比如调用超时、负载均衡类型、容错处理方式等。
- ServiceConfig：针对具体一个接口的配置，比如接口名、实现类等。
- ReferenceConfig：针对一个具体接口引用的配置，比如引用的接口名、url等。
- MethodConfig：针对方法的配置参数，比如超时、失败重试次数、Mock方法等。

##### URL

URL类完成了一次Dubbo调用的整个过程，Dubbo中每个可配置的单元都有唯一的URL，配置参数通过URL的param来携带。URL是对标志URL类的扩展，比如对于一个注册中心，他有一个url：`zookeeper://224.5.6.7:2181` ，代表这是一个zookeeper注册中心。再比如一个服务的url是：`dubbo://10.5.6.7:28080/org.apache.dubbo.demo.DemoService?version=1.0.0&timeout=3000` 代表这个服务是以dubbo协议暴露在28080端口，版本1.0.0，调用超时3秒。大部分时候我们不会直接接触到这些url的具体值，只要了解**在一次dubbo调用中，所有的配置参数都是通过url来传递**。

##### 服务引用

回到一次RPC的调用开始，首先看下dubbo官方Demo：

```java
@Component("demoServiceComponent")
public class DemoServiceComponent implements  DemoService{
    @Reference
    private DemoService demoService;

    @Override
    public String sayHello(String name) {
        return demoService.sayHello(name);
    }
}
```

@Reference功能类似于@Autowried，为了实现demoService.sayHello()这个调用，Dubbo需要做两件事情：

​	1.需要实现一个DemoService的实现类，这个实现类能够调用远程的DemoService接口。

​	2.在Spring Bean初始化的时候将这个实现注入到Bean中。

为了做到，Demo在启动类做了处理：

```java
public class Application {
    /**
     * In order to make sure multicast registry works, need to specify '-Djava.net.preferIPv4Stack=true' before
     * launch the application
     */
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ConsumerConfiguration.class);
        context.start();
        DemoService service = context.getBean("demoServiceComponent", DemoServiceComponent.class);
        String hello = service.sayHello("world");
        System.out.println("result :" + hello);
    }

    @Configuration
    @EnableDubbo(scanBasePackages = "org.apache.dubbo.demo.consumer.comp")
    @PropertySource("classpath:/spring/dubbo-consumer.properties")
    @ComponentScan(value = {"org.apache.dubbo.demo.consumer.comp"})
    static class ConsumerConfiguration {

    }
}
```

启动类里，在Spring的配置上 加了一个@EnableDubbo的注解，同时配置了Dubbo的扫描路径。在Spring初始化的时候，dubbo就会到这个路径下扫描所有的@Reference注解，将生成的代理对象注入进去。

##### @EnableDubbo注解原理

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

@EnableDubbo是一个派生注解，它组合了2个Dubbo相关的注解

- **@EnableDubboConfig** 这个注解的作用是初始化上面讲的Dubbo配置类，同时将这些配置类注册成Spring Bean。
- **@DubboComponentScan** 这个是Dubbo注解的核心，扫描路径下所有的类，如果字段上有@Referenece注解，则说明这是个远程引用，dubbo会尝试生成接口的远程代理类，然后注入到这个字段上；如果类上有@Service注解（注意是org.apache.dubbo.config.annotation.Service），则说明这是一个远程服务接口，dubbo会尝试将服务暴露出去。

##### @EnableDubboConfig注解原理

这个注解的作用类似于SpringBoot中的外部化配置功能，用于读取环境变量或者配置文件中的配置来赋值给配置类，同时将配置类注册成Spring Bean。Dubbo支持配置来自于多个来源，这些配置来自于多个层级，可以根据优先级互相覆盖，具体可以查看官方文档的配置加载流程部分。

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@Import(DubboConfigConfigurationRegistrar.class)
public @interface EnableDubboConfig {
    /**
     * It indicates whether binding to multiple Spring Beans.
     *
     * @return the default value is <code>false</code>
     * @revised 2.5.9
     */
    boolean multiple() default true;
}
```

这个注解的主要配置工作通过DubboConfigConfigurationRegistrar类来完成。在这个Registrar类中，会将类的属性和配置绑定：

~~~java
public class DubboConfigConfigurationRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(         importingClassMetadata.getAnnotationAttributes(EnableDubboConfig.class.getName()));
        boolean multiple = attributes.getBoolean("multiple");
        // Single Config Bindings
        registerBeans(registry, DubboConfigConfiguration.Single.class);

        if (multiple) { // Since 2.6.6 https://github.com/apache/dubbo/issues/3193
            registerBeans(registry, DubboConfigConfiguration.Multiple.class);
        }
        // Since 2.7.6
        registerCommonBeans(registry);
    }
}
~~~

```java
public class DubboConfigConfiguration {

    /**
     * Single Dubbo {@link AbstractConfig Config} Bean Binding
     */
    @EnableConfigurationBeanBindings({
            @EnableConfigurationBeanBinding(prefix = "dubbo.application", type = ApplicationConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.module", type = ModuleConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.registry", type = RegistryConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.protocol", type = ProtocolConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.monitor", type = MonitorConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.provider", type = ProviderConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.consumer", type = ConsumerConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.config-center", type = ConfigCenterBean.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.metadata-report", type = MetadataReportConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.metrics", type = MetricsConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.ssl", type = SslConfig.class)
    })
    public static class Single {

    }

    /**
     * Multiple Dubbo {@link AbstractConfig Config} Bean Binding
     */
    @EnableConfigurationBeanBindings({
            @EnableConfigurationBeanBinding(prefix = "dubbo.applications", type = ApplicationConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.modules", type = ModuleConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.registries", type = RegistryConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.protocols", type = ProtocolConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.monitors", type = MonitorConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.providers", type = ProviderConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.consumers", type = ConsumerConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.config-centers", type = ConfigCenterBean.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.metadata-reports", type = MetadataReportConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.metricses", type = MetricsConfig.class, multiple = true)
    })
    public static class Multiple {

    }
}
```

从上面的配置可以看出dubbo.application开通的属性会绑定到ApplicationConfig配置类中。配置类有了，就可以使用浙西配置初始化Dubbo的Reference对象了。

##### @DubboComponentScan 注解原理

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(DubboComponentScanRegistrar.class)
public @interface DubboComponentScan {
     ...
}
```

```java
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        //解析需要扫描的路径
        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);
        //将带@Service注解的类暴露成服务接口
        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);

        // @since 2.7.6 Register the common beans
        registerCommonBeans(registry);
    }
    ....
}
public interface DubboBeanUtils {
    static void registerCommonBeans(BeanDefinitionRegistry registry) {

        // Since 2.5.7 Register @Reference Annotation Bean Processor as an infrastructure Bean
        registerInfrastructureBean(registry, ReferenceAnnotationBeanPostProcessor.BEAN_NAME,
                ReferenceAnnotationBeanPostProcessor.class);
DubboConfigAliasPostProcessor.BEAN_NAME,
                DubboConfigAliasPostProcessor.class);

        // Since 2.7.5 Register DubboLifecycleComponentApplicationListener as an infrastructure Bean
        registerInfrastructureBean(registry, DubboLifecycleComponentApplicationListener.BEAN_NAME,
                DubboLifecycleComponentApplicationListener.class);

        // Since 2.7.4 Register DubboBootstrapApplicationListener as an infrastructure Bean
        registerInfrastructureBean(registry, DubboBootstrapApplicationListener.BEAN_NAME,
                DubboBootstrapApplicationListener.class);

        // Since 2.7.6 Register DubboConfigDefaultPropertyValueBeanPostProcessor as an infrastructure Bean
        registerInfrastructureBean(registry, DubboConfigDefaultPropertyValueBeanPostProcessor.BEAN_NAME,
                DubboConfigDefaultPropertyValueBeanPostProcessor.class);
    }
}
```

registerCommonBeans()方法中，注册了如下几个BeanPostProcessor：

- ReferenceAnnotationBeanPostProcessor：处理@Reference注解。
- DubboConfigAliasPostProcessor：对所有注册的Config Bean如果id属性不为空，则用id作为bean name的别名。
- DubboLifecycleComponentApplicationListener：初始化实现了LifeCycle接口的Bean，比如Dubbo的Environment，监听Spring Context事件。

- DubboBootstrapApplicationListener：在Spring Contex状态变化时初始化或销毁DubboBootStrap。
- DubboConfigDefaultPropertyValueBeanPostProcessor：修改Config Bean的bean Name为 Config类或id的name属性。

##### @Reference注解处理

```java
public class ReferenceAnnotationBeanPostProcessor extends AbstractAnnotationBeanPostProcessor implements
        ApplicationContextAware, ApplicationListener {
    @Override
    protected Object doGetInjectedBean(AnnotationAttributes attributes, Object bean, String beanName, Class<?> injectedType,
                                       InjectionMetadata.InjectedElement injectedElement) throws Exception {
        /**
         * 获取引用的Service的Bean Name
         */
        String referencedBeanName = buildReferencedBeanName(attributes, injectedType);

        /**
         * 获取Reference Bean Name
         */
        String referenceBeanName = getReferenceBeanName(attributes, injectedType);
        // 构造一个ReferenceBean实例
        ReferenceBean referenceBean = buildReferenceBeanIfAbsent(referenceBeanName, attributes, injectedType);
       // 判断引用的接口是否是这个进程暴露的
        boolean localServiceBean = isLocalServiceBean(referencedBeanName, referenceBean, attributes);
        //把上一步构造的ReferenceBean实例注册成Spring Bean
        registerReferenceBean(referencedBeanName, referenceBean, attributes, injectedType);

        cacheInjectedReferenceBean(referenceBean, injectedElement);
        // 初始化Proxy
        return getOrCreateProxy(referencedBeanName, referenceBeanName, referenceBean, injectedType);
    }
}
```

这个类继承了`AbstractAnnotationBeanPostProcessor`，来自于alibaba-spring。作用是扫描calsspath所有类中，包含某个注解的Filed，找到后就会回调doGetInjectedBean()方法，获取到这个Field所应该注入的对象实例，然后注入到Field中。

在上面的代码中，首先根据注解和Field信息，构建Bean Name。然后根据@Reference注解的属性配置新建一个ReferenceBean类的实例，将这个实例注册成一个Spring Bean。

```
这一步其实有一种情况并不会注册ReferenceBean，就是添加@Reference注解的字段有本地实现。比如，服务提供方往外暴露的服务本地也有引用，这种情况如果用户不是用的@Autowird而是用的@Reference注解，Dubbo并不会使用远程调用，仍然会注入本地的Bean实现。
```

上面注册的ReferenceBean实现了Spring的FactoryBean接口，所以Spring在注入的时候，其实是调用它的getObject()方法。而这个方法返回的就是上面最后一步构建的Proxy。构建Proxy的代码如下：

```java
private Object getOrCreateProxy(String referencedBeanName, String referenceBeanName, ReferenceBean referenceBean, Class<?> serviceInterfaceType) {
        //如果@Reference引用的对象有本地的Bean实现，则直接使用java的动态代理返回本地bean的代理
        if (existsServiceBean(referencedBeanName)) { // If the local @Service Bean exists, build a proxy of ReferenceBean
            return newProxyInstance(getClassLoader(), new Class[]{serviceInterfaceType},
                    newReferencedBeanInvocationHandler(referencedBeanName));
        } else { 
            exportServiceBeanIfNecessary(referencedBeanName);
            //调用referenceBean的方法获取远程访问的代理类
            return referenceBean.get();
        }
    }
```

##### 生成远程代理

代码中每个@Reference注解注入的都是referenceBean返回的一个代理，下面看refenceBean.get()方法的实现：

```java
public synchronized T get() {
        if (destroyed) {
            throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
        }
        if (ref == null) {
            init();
        }
        return ref;
    }
```

get() 方法中只是检查一下ref有没有值，有说明之前已经初始化过了，直接返回，没有则初始化。

```java
 public synchronized void init() {
        if (initialized) {
            return;
        }
        //1.初始化DubboBootstrap
        if (bootstrap == null) {
            bootstrap = DubboBootstrap.getInstance();
            bootstrap.init();
        }
        //2. 检查和更新Config
        checkAndUpdateSubConfigs();
        //3. 如果配置了Local和Stub则做检查
        checkStubAndLocal(interfaceClass);
        //4. 如果配置了mock则做检查
        ConfigValidationUtils.checkMock(interfaceClass, this);

        Map<String, String> map = new HashMap<String, String>();
        map.put(SIDE_KEY, CONSUMER_SIDE);

        ReferenceConfigBase.appendRuntimeParameters(map);
        // 5. 将所有配置参数做一次merge
        ...
        ...
        //6. 根据merge好的参数构建proxy
        ref = createProxy(map);

        serviceMetadata.setTarget(ref);
        serviceMetadata.addAttribute(PROXY_CLASS_REF, ref);
        ConsumerModel consumerModel = repository.lookupReferredService(serviceMetadata.getServiceKey());
        consumerModel.setProxyObject(ref);
        consumerModel.init(attributes);

        initialized = true;

        // 触发一个Reference初始化完成的event
        dispatch(new ReferenceConfigInitializedEvent(this, invoker));
    }
```

整个初始化过程逻辑比较复杂，但主要就是参数检查和合并(因为Dubbo中同一个参数可以在不同级别上都配置，这里需要根据优先级进行覆盖合并)。

##### 合并Invoker

```java
private T createProxy(Map<String, String> map) {
        //1. 判断是否是进程内调用
        if (shouldJvmRefer(map)) {
            URL url = new URL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
            invoker = REF_PROTOCOL.refer(interfaceClass, url);
            if (logger.isInfoEnabled()) {
                logger.info("Using injvm service " + interfaceClass.getName());
            }
        } else {
            urls.clear();
            //2. 判断用户是否配置了对端地址
            if (url != null && url.length() > 0) { 
                String[] us = SEMICOLON_SPLIT_PATTERN.split(url);
                if (us != null && us.length > 0) {
                    for (String u : us) {
                        URL url = URL.valueOf(u);
                        if (StringUtils.isEmpty(url.getPath())) {
                            url = url.setPath(interfaceName);
                        }
                       //3. 地址是否是注册中心的地址
                        if (UrlUtils.isRegistry(url)) {
                            urls.add(url.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        } else {
                            urls.add(ClusterUtils.mergeUrl(url, map));
                        }
                    }
                }
            } else { 
                if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())) {
                    // 4. 检查是否有注册中心配置，没有会抛出异常
                    checkRegistry();
                    // 5. 获取所有配置中心的url
                    List<URL> us = ConfigValidationUtils.loadRegistries(this, false);
                    if (CollectionUtils.isNotEmpty(us)) {
                        for (URL u : us) {
                            URL monitorUrl = ConfigValidationUtils.loadMonitor(this, u);
                            if (monitorUrl != null) {
                                map.put(MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                            }
                            // 6. 是注册中心地址，则转换url 
                            urls.add(u.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        }
                    }
                    if (urls.isEmpty()) {
                        throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                    }
                }
            }

            if (urls.size() == 1) {
                // 7. 如果只获取到一个url，则直接通过Protocol获取Invoker
                invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
            } else {
                // 8. 如果是多个地址，则生成多个invoker
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                for (URL url : urls) {
                    invokers.add(REF_PROTOCOL.refer(interfaceClass, url));
                    if (UrlUtils.isRegistry(url)) {
                        // 9. 声明了多个注册中心的地址，默认采用最后一个
                        registryURL = url; 
                    }
                }
                if (registryURL != null) {
                    // 10. 如果多个地址中有注册中心的地址，则使用ZoneAwareCluster
                    URL u = registryURL.addParameterIfAbsent(CLUSTER_KEY, ZoneAwareCluster.NAME);
                    invoker = CLUSTER.join(new StaticDirectory(u, invokers));
                } else { 
                    // 11. 所有的地址都是直连地址，直接用这些地址生成ClusterInvoker
                    invoker = CLUSTER.join(new StaticDirectory(invokers));
                }
            }
        }
        //检查服务是否在线
        if (shouldCheck() && !invoker.isAvailable()) {
            throw new IllegalStateException("Failed to check the status of the service "
                    + interfaceName
                    + ". No provider available for the service "
                    + (group == null ? "" : group + "/")
                    + interfaceName +
                    (version == null ? "" : ":" + version)
                    + " from the url "
                    + invoker.getUrl()
                    + " to the consumer "
                    + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
        }
        if (logger.isInfoEnabled()) {
            logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
        }
        /**
         * @since 2.7.0
         * ServiceData Store
         */
        String metadata = map.get(METADATA_KEY);
        WritableMetadataService metadataService = WritableMetadataService.getExtension(metadata == null ? DEFAULT_METADATA_STORAGE_TYPE : metadata);
        if (metadataService != null) {
            URL consumerURL = new URL(CONSUMER_PROTOCOL, map.remove(REGISTER_IP_KEY), 0, map.get(INTERFACE_KEY), map);
            metadataService.publishServiceDefinition(consumerURL);
        }
        // 使用得到的Invoker生成接口代理
        return (T) PROXY_FACTORY.getProxy(invoker);
    }
```

具体步骤：

1. 判断是否是进程内调用，比如配置的@Reference(url="[injvm://localhost:28080](https://links.jianshu.com/go?to=injvm%3A%2F%2Flocalhost%3A28080)")，这种方式会直接走进程内调用
2. 判断用户是否在@Reference注解上直接配置了url参数，如果用户配置了则会忽略注册中心的配置
3. 如果配置的是注册中心的地址，比如@Reference(url="")，在会将配置参数加在refer参数中，比如[registry://localhost:28080?refer=version%3f1.0.0](https://links.jianshu.com/go?to=registry%3A%2F%2Flocalhost%3A28080%3Frefer%3Dversion%3f1.0.0)
4. 如果没有配置url，则检查是否有注册中心，没有的话会抛出异常
5. 获取所有注册中心的url
6. 注册中心url添加refer参数，同第3步
7. 如果最终只有一个url，则使用Protocol的refer方法获取到invoker
8. 如果最终由多个url，就生成多个Invoker。
9. 这里还会判断这些url里面有没有注册中心的地址
10. 如果有注册中心，就会在最后一个注册中心url中加一个cluster=zone-aware的参数
11. 最终会将多个invoker合并成一个ClusterInvoker

```
这里多提一句cluster=zone-aware参数的作用，当dubbo发现一个ReferenceBean关联到多个注册中心的时候，在发送请求时需要一个优先策略。加了这个参数就表明，优先选择同一个可用区的的注册中心。
```

##### 创建代理

获取到Invoker后，就可以使用Invoker生成代理了，最后一步是调用ProxyFactory.getProxy()获取到接口代理。而生成的代理类通过传入的Invoker参数来发起请求和处理Response。

#### 注解方式初始化总结

Dubbo + Sping的组合式当前最流行的使用方式，应用通过@EnableDubbo注解来激活Dubbo。Dubbo首先读取外部配置生成配置类并将配置类注册为Srping Bean，然后扫描classpath下所有类，找到添加@Reference注解的字段，从而得出有哪些接口需要生成远程代理。对每个接口的每个配置都会注册一个ReferenceBean，同时调用它的get()方法根据配置生成URL，进而创建Proxy。生成的Proxy通过Spring注入到业务类中。

##### 其他初始化方式

Spring xml配置