## Java基础

##### 反射

Class在jvm中是一个字节码文件。获取Class的三种方式：类名.class、对象.getClass()、Class.forName("全名称")。

Class包含：Field、Method、Constructor、Annotation

获取Field：`Field f = clazz.getDeclaredField("字段名")；`如果是私有类型字段，需要先设置允许访问，即`f.setAccessible(true)；`

获取Method：`Method m = clazz.getDeclaredMethod(""); method.invoke();` 

获取构造函数：

~~~java
Constructor c = clazz.getConstructor();
c.newInstance();
~~~

##### 代理

- JDK动态代理 利用反射机制生成要给实现代理接口的类。

  ~~~java
  public class ProxyFactor implements InvocationHandler{
      private Object target;
       public ProxyFactory(Object tar){
          this.target = tar;
      }
      public Object getProxyInstance(){
          // 通过Proxy返回代理
          return Proxy.newProxyInstance(this.target.getClassLodaer(),this.target.getClass().getInterfaces(),this);
      }
      @Override
      public Object invoke(Object obj, Method method, Object[] args, MethodProxy proxy){
          Object returnvalue = method.invoke(this.target, args);
          
          return returnvalue;
      }
  }
  ~~~

  

- cglib动态代理 将代理类的class文件加载，通过修改字节码生成子类的方式代理。

  ~~~java
  import org.springframework.cglib.proxy.MethodInterceptor;
  
  public class ProxyFactory implements MethodInterceptor{
      private Object target;
      public ProxyFactory(Object tar){
          this.target = tar;
      }
      
      public Object getProxyInstance(){
          Enhancer en = new Enhancer(); // 1.工具类
          en.setSuperclass(target.getClass()); // 2.设置父类
          en.setCallback(this); // 3.设置回调函数
          
          return en.create(); // 4.创建并返回子类代理对象
      }
      @Override
      public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy){
          Object returnvalue = method.invoke(this.target, args);
          
          return returnvalue;
      }
  }
  ~~~

##### lambda表达式

`Exception:Target type of a lambda conversion must be an interface`。lambda表达式只能用来简化只有一个public方法的接口实现，常和函数式接口配合使用。例：

~~~java
new Thread(()->System.out.println("")).start(); // 这里lambda表达式实现了Runnable接口。
~~~

- 语法
  - 一个参数不需要()，一个语句不需要{}，例如：`param -> param + "";`
  - 类型声明可选，例如：`(String hello, String name) -> hello + " " + name; // String不是必须的`
  - 如果主体只有一个表示的返回值，则不需要return关键字，例如：`()->{return "";} // 等价于 ()->"";`

- 方法引用
  - 对象::实例方法，例如：`System.out::println`
  - 类::静态方法，例如：`String::valueOf`
  - 类::实例方法，例如：`String::equals`

- 构造函数
  - 类::实例方法模型，例如：`User::new // 等价于() -> new User();`

##### 函数式接口

- JDK1.8新增接口

  - Supplier<R> R代表返回值类型，即 `() -> T;`

  - Consumer<T> T代表参数类型，即`T -> void;` //如果定义了多个consumer，则支持链式调用，共用一个参数按顺序从前到后各自执行，如果某个节点异常，则不再执行后面的节点：

    ```java
    Consumer<String> sc = str -> System.out.print(str);
    Consumer<String> sc1 = str -> System.out.print(str);
    sc.andThen(sc1).accept("hello"); // 如果sc报错，则不再执行sc1	
    ```

  - BiConsumer<T,R> TR都代表参数类型，功能同Consumer，即`(T,R) -> void;`

  - Function<T,R> T代表参数类型，R代表返回值类型，即`T -> R;`

    ```java
    Function<Integer, User> uf1 = id -> new User(id);
    Function<Integer, Integer> uf2 = id -> id + 1;
    uf1.andThen(uf2).apply(100); // 支持链式调用，uf1的执行结果返回给uf2作为参数继续执行
    ```

  - BiFunction<T,U,R> TU代表参数类型，R代表返回值类型，即(T,U) -> R;`

  - Predicate<T> T代表参数类型，返回boolean，即`T -> true/false`；

    ```java
    Predicate<String> re = param -> param.length()>1;
    re.and().or().negate().test(""); // 对应 && || !
    ```

  - BiPredicate<T,R> TR代表参数类型，返回boolean，即`(T,R) -> true/false`；

  - UnaryOperator<T> T代表参数和返回类型，功能和Function相同，即`T -> T;`

  - BinaryOperator<T,T> T代表参数和返回类型，功能和Function相同，即`(T,T) -> T;`

- JDK1.8之前接口

  - java.lang.Runnable<T>

  - java.util.Concurrent.Callable<T>

  - java.util.Comparator<T>

    ...

##### 事务

基于JDBC异常connecttion的@Transactional注解事务。其传播类型如下：

- <font color=" #1E90FF">REQUIRED</font> 当前存在事务则加入，否则创建事务
- <font color=" #1E90FF">REQUIRED_NEW</font> 新建事务，与其它事务独立
- <font color=" #1E90FF">NESTED</font> 嵌套事务。与其它兄弟事务独立，和父事务相互影响
- <font color=" #1E90FF">SUPPORTS</font> 当前存在事务则加入，否则以普通代码执行
- <font color=" #1E90FF">NOT_SUPPORTED</font> 非事务。不受方法事务影响，以普通代码执行
- <font color=" #1E90FF">MANDATORY</font> 使用当前事务，如果不存在则抛出异常
- <font color=" #1E90FF">NEVER</font> 如果当前存在事务，则抛出异常

##### 泛型

- <font color="#8B0000">类型擦除</font>：Java中的泛型实际是一种语法糖，编译之后类型都是Object。
- 应用：泛型类、泛型接口、泛型方法、泛型集合、泛型约束` extends`

##### 线程池

- 优点：
  - 降低资源消耗。重复利用线程，省去了线程创建销毁的性能消耗；
  - 提高响应速度。当任务创建时，不必等待线程创建，立即执行；
  - 便于统一管理。对线程统一管理，执行状态统一监控；

- 使用：

  ~~~java
  public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue,
  			  ThreadFactory threadFactory,
                            RejectedExecutionHandler handler);
  ~~~

  ~~~java
  executor.execute(IRunable task);//无返回值
  //返回一个Future
  Future<Integer> future = executor.submit(() -> {return 1 + 1;});
  Integer result = future.get();
  ~~~

  ~~~java
  //关闭线程池
  .shutdownNow();// 拒接收新提交的任务，同时立马关闭线程池，返回等待执行任务的列表
  .shutdown();// 拒接收新提交的任务，等待线程池里的任务执行完毕后关闭线程池
  //关闭状态
  .isShutdown()// 使用任一方法关闭都直接返回true
  .isTerminaed()// 线程池中所有任务都结束了才返回true
  ~~~

  <font color="#1E90FF">线程池数量</font>： IO密集型 - 2*CPU   计算密集型 CPU + 1

##### 多线程

      ~~~java
public interface Runnable{ public void run() {}}
public interface Callable{ public Object call() {}}
//通过 implements 以上两个接口实现多线程
      ~~~

##### 集合

##### 锁

##### IO