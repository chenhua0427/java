##### 本质

Servlet是一个java接口，定义了一套处理网络请求的规范

~~~java
public interface Servlet {
    void init(ServletConfig var1) throws ServletException; // 初始化时要做什么
    ServletConfig getServletConfig();
    void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException; //接收到请求时要做什么
    String getServletInfo();
    void destroy(); //销毁时要做什么
}
~~~

HttpServlet实现了Servlet接口，并向子类公布了几个模板方法，交由子类实现

~~~java
public abstract class HttpServlet extends GenericServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp){// 直接返回异常}
    protected void doPost(HttpServletRequest req, HttpServletResponse resp){
        String protocol = req.getProtocol();
        String msg = lStrings.getString("http.method_post_not_supported");
        // 直接返回异常
        if (protocol.endsWith("1.1")) {
            resp.sendError(405, msg);
        } else {
            resp.sendError(400, msg);
        }
    }
    	...
    protected void doOptions(HttpServletRequest req, HttpServletResponse resp){// 自己实现了该方法}
}
~~~

##### 生命周期

Servlet是单例创建，随服务器停止而销毁；创建时机默认是第一次被访问时，可以在配置文件中设置为服务器启动时创建(Servlet在Web.xml中的配置)

~~~xml
<servlet>
        <servlet-name>default</servlet-name>
        <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
        <init-param> //Servlet初始化参数ServletConfig
            <param-name>debug</param-name>
            <param-value>0</param-value>
        </init-param>
        <init-param>
            <param-name>listings</param-name>
            <param-value>false</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup> //启动时创建
</servlet>
<servlet-mapping>
        <servlet-name>default</servlet-name>
        <url-pattern>/</url-pattern> //拦截的url，支持ant格式；表示：没有被其它Servlet匹配到的时候，匹配该Servlet，即最低优先级
</servlet-mapping>
~~~

##### 域对象

ServletContext 、HttpSession、ServletRequest都是Map结构支持getAttribute()/setAttribute()方法。

##### Servlet3.0特性

`@WebServlet(value="/hello")`通过注解的方式配置自定义Servlet

`@WebFilter(value="/*")`通过注解的方式配置自定义Filter

`@WebListener()`通过注解的方式配置自定义Listener

ServletContext增强，对象支持在运行时动态部署 Servlet、过滤器、监听器，以及为 Servlet 和过滤器增加URL 映射等

HttpServletRequest 对文件上传的支持

...

##### DispatcherServlet

SpringMVC框架核心，自定义Servlet，接收并处理请求，包含以下功能：

1. 调用HandlerMapping，找到具体的处理器对象即拦截器，即Controller和Method
2. 调用HandlerAdapter，通过反射的方式执行Controller.Method并返回ModelAndView对象
3. 调用ViewReslover(视图解析器)解析ModelAndView对象，并返回view
4. 根据View进行渲染视图(模型数据填充至视图)，返回请求

![dispatcherservlet](/docs/images/servlet-dispatcherservlet.png)