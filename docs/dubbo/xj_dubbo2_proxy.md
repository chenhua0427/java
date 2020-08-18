## 详解Dubbo（二）：消费端代理

##### 代理工厂

Dubbo中代理实例是通过代理工厂来获得的，其定义如下：

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

其中getProxy()方法是给Comsumer使用来获取代理实例的，它需要提供一个Invoker作为参数，而重载方法中generic参数是代表是否生成一个泛化接口的代理。最后一个getInvoker()是给Provider暴露服务时候使用。

##### 代理工厂实现

Dubbo提供了2中代理工厂的实现：一种是JDK自带动态代理；一种是使用javassist直接生成字节码实现(默认)。

- JDK动态代理

  ```java
  public class JdkProxyFactory extends AbstractProxyFactory {
  
      @Override
      @SuppressWarnings("unchecked")
      public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
          return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
      }
  }
  ```

  ~~~java
  public class InvokerInvocationHandler implements InvocationHandler {
      private final Invoker<?> invoker;
      private ConsumerModel consumerModel;
  
      public InvokerInvocationHandler(Invoker<?> handler) {
          this.invoker = handler;
          String serviceKey = invoker.getUrl().getServiceKey();
          if (serviceKey != null) {
              this.consumerModel = ApplicationModel.getConsumerModel(serviceKey);
          }
      }
       public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
          if (method.getDeclaringClass() == Object.class) {
              return method.invoke(invoker, args);
          }
          String methodName = method.getName();
          Class<?>[] parameterTypes = method.getParameterTypes();
          if (parameterTypes.length == 0) {
              if ("toString".equals(methodName)) {
                  return invoker.toString();
              } else if ("$destroy".equals(methodName)) {
                  invoker.destroy();
                  return null;
              } else if ("hashCode".equals(methodName)) {
                  return invoker.hashCode();
              }
          } else if (parameterTypes.length == 1 && "equals".equals(methodName)) {
              return invoker.equals(args[0]);
          }
          RpcInvocation rpcInvocation = new RpcInvocation(method, invoker.getInterface().getName(), args);
          String serviceKey = invoker.getUrl().getServiceKey();
          rpcInvocation.setTargetServiceUniqueName(serviceKey);
        
          if (consumerModel != null) {
              rpcInvocation.put(Constants.CONSUMER_MODEL, consumerModel);
              rpcInvocation.put(Constants.METHOD_MODEL, consumerModel.getMethodModel(method));
          }
  
          return invoker.invoke(rpcInvocation).recreate();
      }
  }
  ~~~

  逻辑：首先判断要调用的方法是否属于Object类，或者是调用的toString()、$destroy或hashCode方法，如果是的话，直接调用本地方法，不会发起远程调用。否则，将请求参数封装至RpcInvocation中，通过Invoker发出去。

- Javassist动态代理

  javassist是一个字节码生成工具，它可以在程序运行期间动态编译源代码生成class字节码。

  Dubbo加载这些类的字节码，然后发起调用。简单说，Dubbo为每个远程接口都生成一份源码并编译，在请求时直接调用和泽泻编译好的类的方法就行了，这样显然比JDK动态代理快。

  ```java
  public class JavassistProxyFactory extends AbstractProxyFactory {
  
      @Override
      @SuppressWarnings("unchecked")
      public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
          return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
      }
  }
  ```

  代理工厂实现类里面直接调用的Proxy.getProxy()获取一个proxy的class，然后通过传入InvokerInvocationHandler参数调用newInstance()方法。这个参数跟JDK动态代理回调的handler是同一个。所以这里的核心就是Proxy的class是怎么产生和加载的。这个Proxy是Dubbo自定义的类：

  ```java
  public static Proxy getProxy(ClassLoader cl, Class<?>... ics) {
          ...
          ...
          //参数校验的逻辑省略
          StringBuilder sb = new StringBuilder();
          for (int i = 0; i < ics.length; i++) {
              String itf = ics[i].getName();
              ...
              ...
              sb.append(itf).append(';');
          }
  
          // key是所有要实现的接口用分号连在一起
          String key = sb.toString();
          
          // 查看缓存中是否已经生成过这个代理，如果生成过直接返回，如果生成中则等待
          final Map<String, Object> cache;
          synchronized (PROXY_CACHE_MAP) {
              cache = PROXY_CACHE_MAP.computeIfAbsent(cl, k -> new HashMap<>());
          }
  
          Proxy proxy = null;
          synchronized (cache) {
              do {
                  Object value = cache.get(key);
                  if (value instanceof Reference<?>) {
                      proxy = (Proxy) ((Reference<?>) value).get();
                      if (proxy != null) {
                          return proxy;
                      }
                  }
  
                  if (value == PENDING_GENERATION_MARKER) {
                      try {
                          cache.wait();
                      } catch (InterruptedException e) {
                      }
                  } else {
                      cache.put(key, PENDING_GENERATION_MARKER);
                      break;
                  }
              }
              while (true);
          }
  
          long id = PROXY_CLASS_COUNTER.getAndIncrement();
          String pkg = null;
          ClassGenerator ccp = null, ccm = null;
          try {
              // Java类源代码生成器，封装了javaassist
              ccp = ClassGenerator.newInstance(cl);
  
              Set<String> worked = new HashSet<>();
              List<Method> methods = new ArrayList<>();
  
              for (int i = 0; i < ics.length; i++) {
                  if (!Modifier.isPublic(ics[i].getModifiers())) {
                      String npkg = ics[i].getPackage().getName();
                      if (pkg == null) {
                          pkg = npkg;
                      } else {
                          if (!pkg.equals(npkg)) {
                              throw new IllegalArgumentException("non-public interfaces from different packages");
                          }
                      }
                  }
                  //添加类要实现的接口
                  ccp.addInterface(ics[i]);
                  //添加类要实现的方法
                  for (Method method : ics[i].getMethods()) {
                      String desc = ReflectUtils.getDesc(method);
                      if (worked.contains(desc) || Modifier.isStatic(method.getModifiers())) {
                          continue;
                      }
                      if (ics[i].isInterface() && Modifier.isStatic(method.getModifiers())) {
                          continue;
                      }
                      worked.add(desc);
  
                      int ix = methods.size();
                      Class<?> rt = method.getReturnType();
                      Class<?>[] pts = method.getParameterTypes();
  
                      StringBuilder code = new StringBuilder("Object[] args = new Object[").append(pts.length).append("];");
                      for (int j = 0; j < pts.length; j++) {
                          code.append(" args[").append(j).append("] = ($w)$").append(j + 1).append(";");
                      }
                      code.append(" Object ret = handler.invoke(this, methods[").append(ix).append("], args);");
                      if (!Void.TYPE.equals(rt)) {
                          code.append(" return ").append(asArgument(rt, "ret")).append(";");
                      }
  
                      methods.add(method);
                      ccp.addMethod(method.getName(), method.getModifiers(), rt, pts, method.getExceptionTypes(), code.toString());
                  }
              }
  
              if (pkg == null) {
                  pkg = PACKAGE_NAME;
              }
  
              // 添加构造函数和成员变量
              String pcn = pkg + ".proxy" + id;
              ccp.setClassName(pcn);
              ccp.addField("public static java.lang.reflect.Method[] methods;");
              ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
              ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{InvocationHandler.class}, new Class<?>[0], "handler=$1;");
              ccp.addDefaultConstructor();
              Class<?> clazz = ccp.toClass();
              clazz.getField("methods").set(null, methods.toArray(new Method[0]));
  
              // 创建Proxy的子类，覆盖newInstance()方法
              String fcn = Proxy.class.getName() + id;
              ccm = ClassGenerator.newInstance(cl);
              ccm.setClassName(fcn);
              ccm.addDefaultConstructor();
              ccm.setSuperClass(Proxy.class);
              ccm.addMethod("public Object newInstance(" + InvocationHandler.class.getName() + " h){ return new " + pcn + "($1); }");
              Class<?> pc = ccm.toClass();
              proxy = (Proxy) pc.newInstance();
          } catch (RuntimeException e) {
              throw e;
          } catch (Exception e) {
              throw new RuntimeException(e.getMessage(), e);
          } finally {
              // release ClassGenerator
              if (ccp != null) {
                  ccp.release();
              }
              if (ccm != null) {
                  ccm.release();
              }
              synchronized (cache) {
                  if (proxy == null) {
                      cache.remove(key);
                  } else {
                      cache.put(key, new WeakReference<Proxy>(proxy));
                  }
                  cache.notifyAll();
              }
          }
          return proxy;
      }
  ```

  下面看Dubbo的Demo中的远程接口，通过javassist生成的代理类的源代码是什么样子的。源接口：

  ```java
  public interface DemoService {
      String sayHello(String name);
  
      default CompletableFuture<String> sayHelloAsync(String name) {
          return CompletableFuture.completedFuture(sayHello(name));
      }
  }
  ```

  生成的实现类源代码

  ```java
  public class org.apache.dubbo.common.bytecode.Proxy0 extends org.apache.dubbo.common.bytecode.Proxy {
      public Object newInstance(java.lang.reflect.InvocationHandler h){ 
          return new org.apache.dubbo.common.bytecode.proxy0(h); 
      }
  }
  
  public class org.apache.dubbo.common.bytecode.proxy0 implements Destroyable, EchoService, DemoService{
      public static java.lang.reflect.Method[] methods;
      private java.lang.reflect.InvocationHandler handler;
      public proxy0(java.lang.reflect.InvocationHandler arg0){
          handler = arg0;
      }
  
      public void $destroy(){
          Object[] args = new Object[0]; 
          handler.invoke(this, methods[0], args);
      }
      
      public String sayHello(String arg0){
          Object[] args = new Object[1]; 
          args[0] = arg0; 
          return handler.invoke(this, methods[1], args); 
      }
      
      public CompletableFuture sayHelloAsync(String arg0){
          Object[] args = new Object[1]; 
          args[0] = arg0; 
          return handler.invoke(this, methods[2], args); 
      }
      
      public Object $echo(Object arg0){
          Object[] args = new Object[1]; 
          args[0] = ($w)$1; 
          return handler.invoke(this, methods[3], args); 
      }
  }
  ```

  从生成的源码看到javassist代理工厂实际上就是生成一个封装类，所有的方法调用都是通过调用handler来实现的，这种方式是Dubbo默认生成代理的方式。

#### 本地存根

有时候我们想在调用Proxy前后做一些事情，这个时候就会用到Dubbo的Stub：

```
改成远程服务调用后，客户端通常只剩下接口，而实现全在服务器端，但提供方有些时候想在客户端也执行部分逻辑，比如：做 ThreadLocal 缓存，提前验证参数，调用失败后伪造容错数据等等，此时就需要在 API 中带上 Stub，客户端生成 Proxy 实例，会把 Proxy 通过构造函数传给 Stub，然后把 Stub 暴露给用户，Stub 可以决定
要不要去调 Proxy。
```

例子：

```java
public class BarServiceStub implements BarService {
    private final BarService barService;
    
    // 构造函数传入真正的远程代理对象
    public BarServiceStub(BarService barService){
        this.barService = barService;
    }
 
    public String sayHello(String name) {
        // 此代码在客户端执行, 你可以在客户端做ThreadLocal本地缓存，或预先验证参数是否合法，等等
        try {
            return barService.sayHello(name);
        } catch (Exception e) {
            // 你可以容错，可以做任何AOP拦截事项
            return "容错数据";
        }
    }
}
```

Stub本质上是一个静态代理(拥有被代理类实例及方法的调用)。在dubbo调用到达获取Proxy这一步的时候，不是直接返回Proxy，而是把proxy实例给到用户自定义的Stub类。所以Dubbo要求Stub必须要有一个入参是对应接口的构造函数。这样最终ProxyFactory返回的是用户自定义的Stub，用户在Stub中决定在调用Proxy的前后做一些自定义操作。用户通过@Reference注解的Stub熟悉来设置Stub类的名字。