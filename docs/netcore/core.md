### core框架本质

1. 从hello world说起

   ~~~c#
   public class Program
   {
       public static void Main()
       => new WebHostBuilder()
           .UseKestrel()
           .Configure(app => app.Run(context => context.Response.WriteAsync("Hello World!")))
           .Build()
           .Run();
   }
   ~~~

   这段代码涉及到两个核心对象<font color="#8B0000">`WebHost`</font>和<font color="#8B0000">`WebHostBuilder`</font>。我们可以将webhost理解为寄宿或者承载web应用的宿主，应用的启动可以通过启动作为宿主的WebHost来实现。至于WebHostBuilder，顾名思义，就是WebHost的构建者。

   .UseKestrel()旨在注册一个名为Kestrel的服务器

   .Configure()方法用于注册一个<font color="#8B0000">中间件</font>

   当调用Run方法启动webhost的时候，利用WebHostBuilder提供的服务器和中间件构建一个请求处理<font color="#8B0000">管道</font>。这个<font color="#8B0000">由一个服务器和多个中间件构成的管道就是ASP.NET Core框架的核心</font>

2. 自定义.net core框架

   ​    <font color="red">Pipeline =Server + Middlewares</font>，可以将多个Middlewares构成一个单一的<font color="red">HttpHandler</font>

   ​    七个核心对象：HttpContext、RequestDelegate、Middleware、ApplicationBuilder、Server、WebHost、WebHostBuilder

   1. <font color="#1E90FF">`HttpContext`</font>：共享上下文；

      ~~~c#
      public class HttpContext
      {           
          public  HttpRequest Request { get; }
          public  HttpResponse Response { get; }
      }
      public class HttpRequest
      {
          public  Uri Url { get; }
          public  NameValueCollection Headers { get; }
          public  Stream Body { get; }
      }
      public class HttpResponse
      {
          public  NameValueCollection Headers { get; }
          public  Stream Body { get; }
          public int StatusCode { get; set;}
      }
      ~~~

   2. <font color="#1E90FF">`RequestDelegate`</font>：HttpHandler可以表示为一个Func<HttpContext，Task>对象，即独立类型RequestDelegate。

   3. <font color="#1E90FF">`Middleware`</font>：中间件在ASP.NET Core被表示成一个 <font color="red">`Func<RequestDelegate, RequestDelegate>`</font>对象，也就是说它的输入和输出都是一个RequestDelegate。

   4. <font color="#1E90FF">`ApplicationBuilder`</font>：Application承担了处理请求的HttpHandler的所有职责。那么HttpHandler就等于Application，由于HttpHandler通过RequestDelegate表示，那么由ApplicationBuilder构建的Application就是一个RequestDelegate对象。

      ~~~c#
      public interface  IApplicationBuilder
      {
          //注册中间件
          IApplicationBuilder Use(Func<RequestDelegate, RequestDelegate> middleware);
          RequestDelegate Build();//将注册的中间件构建成一个RequestDelegate对象
      }
      
      ~~~

      ~~~c#
      public class ApplicationBuilder : IApplicationBuilder
      {
          private readonly List<Func<RequestDelegate, RequestDelegate>> _middlewares = new List<Func<RequestDelegate, RequestDelegate>>();
          public RequestDelegate Build()
          {
              _middlewares.Reverse();
              return httpContext =>
              {
                  // 定义一个RequestDelegate作为输入
                  RequestDelegate next = _ => { _.Response.StatusCode = 404; return Task.CompletedTask; };
                  foreach (var middleware in _middlewares)
                  {
                      next = middleware(next);// 执行所有注册中间件，并将请求向后分发
                  }
                  return next(httpContext);
              };
          }
      
          public IApplicationBuilder Use(Func<RequestDelegate, RequestDelegate> middleware)
          {
              _middlewares.Add(middleware);
              return this;
          }
      }
      ~~~

   5. <font color="#1E90FF">`Server`</font>：服务器在管道中的职责非常明确。启动后的服务器会绑定到指定的端口进行请求监听，一旦请求到达，服务器会创建代表上下文的HttpContext对象，并将上下文作为输入调用由所有注册中间件构建成的RequestDelegate对象。

      ~~~c#
      public interface IServer
      { 
          Task StartAsync(RequestDelegate handler);
      }
      ~~~

      ~~~c#
      public class HttpListenerFeature : IHttpRequestFeature, IHttpResponseFeature
      {
          private readonly HttpListenerContext _context;
          public HttpListenerFeature(HttpListenerContext context) => _context = context;
      
          Uri IHttpRequestFeature.Url => _context.Request.Url;
          NameValueCollection IHttpRequestFeature.Headers => _context.Request.Headers;
          NameValueCollection IHttpResponseFeature.Headers => _context.Response.Headers;
          Stream IHttpRequestFeature.Body => _context.Request.InputStream;
          Stream IHttpResponseFeature.Body => _context.Response.OutputStream;
          int IHttpResponseFeature.StatusCode { get => _context.Response.StatusCode; set => _context.Response.StatusCode = value; }
      }
      ~~~

      ~~~c#
      public class HttpListenerServer : IServer
      {
          private readonly HttpListener     _httpListener;
          private readonly string[]             _urls;
      
          public HttpListenerServer(params string[] urls)
          {
              _httpListener = new HttpListener();
              _urls = urls.Any()?urls: new string[] { "http://localhost:5000/"};
          }
      
          public async Task StartAsync(RequestDelegate handler)
          {
              Array.ForEach(_urls, url => _httpListener.Prefixes.Add(url));    
              _httpListener.Start();
              while (true)
              {
                  var listenerContext = await _httpListener.GetContextAsync(); 
                  var feature = new HttpListenerFeature(listenerContext);
                  var features = new FeatureCollection()
                      .Set<IHttpRequestFeature>(feature)
                      .Set<IHttpResponseFeature>(feature);
                  var httpContext = new HttpContext(features);
                  await handler(httpContext);
                  listenerContext.Response.Close();
              }
          }
      }
      ~~~

   6. <font color="#1E90FF">`WebHost`</font>：

      ~~~c#
      public interface IWebHost
      {
          Task StartAsync();
      }
      ~~~

      ~~~c#
      public class WebHost : IWebHost
      {
          private readonly IServer _server;
          private readonly RequestDelegate _handler; 
          public WebHost(IServer server, RequestDelegate handler)
          {
              _server = server;
              _handler = handler;
          } 
          public Task StartAsync() => _server.StartAsync(_handler);
      }
      ~~~

   7. <font color="#1E90FF">`WebHostBuilder`</font>：

      WebHostBuilder的使命非常明确：就是创建作为应用宿主的WebHost。由于在创建WebHost的时候需要提供注册的服务器和由所有注册中间件构建而成的RequestDelegate，所以在对应接口IWebHostBuilder中，我们为它定义了三个核心方法。

      ~~~c#
      public interface IWebHostBuilder
      {
          IWebHostBuilder UseServer(IServer server);
          // 中间件的注册是利用IApplicationBuilder对象来完成的
          IWebHostBuilder Configure(Action<IApplicationBuilder> configure);
          IWebHost Build();
      }
      ~~~

      ~~~c#
      public class WebHostBuilder : IWebHostBuilder
      {
          private IServer _server;
          private readonly List<Action<IApplicationBuilder>> _configures = new List<Action<IApplicationBuilder>>();   
      
          public IWebHostBuilder Configure(Action<IApplicationBuilder> configure)
          {
              _configures.Add(configure);
              return this;
          }
          public IWebHostBuilder UseServer(IServer server)
          {
              _server = server;
              return this;
          }   
      
          public IWebHost Build()
          {
              var builder = new ApplicationBuilder();
              foreach (var configure in _configures)
              {
                  configure(builder);
              }
              return new WebHost(_server, builder.Build());
          }
      }
      ~~~

   8. 最终使用

      ~~~c#
      public class Program
      {
          public static async Task Main()
          {
              await new WebHostBuilder()
                  .UseHttpListener()
                  .Configure(app => app
                      .Use(FooMiddleware)
                      .Use(BazMiddleware))
                  .Build()
                  .StartAsync();
          }
      
          public static RequestDelegate FooMiddleware(RequestDelegate next)
          => async context => {
              await context.Response.WriteAsync("Foo=>");
              await next(context);
          };
      
          public static RequestDelegate BazMiddleware(RequestDelegate next)
          => context => context.Response.WriteAsync("Baz");
      }
      ~~~

      ~~~c#
      public static partial class Extensions
      {
         public static IWebHostBuilder UseHttpListener(this IWebHostBuilder builder, params string[] urls)
          => builder.UseServer(new HttpListenerServer(urls));
      
          public static Task WriteAsync(this HttpResponse response, string contents)
          {
              var buffer = Encoding.UTF8.GetBytes(contents);
              return response.Body.WriteAsync(buffer, 0, buffer.Length);
           }
      }
      ~~~

3. 

##### 中间件

- Use、next：Use将多个请求委托链接在一起，next参数标识管道中的下一个委托；

- Run：定义管道中最后一个中间件；

- Map：创建管道分支

  ~~~c#
  public void Configure(IApplicationBuilder app)
      {
          app.Map("/map1", HandleMapTest1);
  
          app.Map("/map2", HandleMapTest2);
  
          app.Use(async (context, next) =>
          {
              // Do work that doesn't write to the Response.
              await next.Invoke();
              // Do logging or other work that doesn't write to the Response.
          });
          app.Run(async context =>
          {
              await context.Response.WriteAsync("Hello from non-Map delegate. <p>");
          });
      }
  ~~~

  ~~~c#
      // 常用中间件：
      app.UseExceptionHandler("/Error"); //异常
      app.UseHsts();//强制浏览器通过HTTPS进行所有通信，
      app.UseHttpsRedirection(); //HTTPS 重定向，即将HTTP请求重定向到HTTPS
      app.UseStaticFiles();//静态文件服务器
      app.UseCookiePolicy();//Cookie 策略实施
      app.UseAuthentication();//身份验证
      app.UseSession();//会话
      app.UseMvc();//MVC
      app.UseWebSockets();//启用websocket协议
      app.UseCors(); //配置跨域资源共享
  ~~~

- 中间件应用

  ~~~c#
  //ConfigureServices中配置跨域
  services.AddCors(options =>{options.AddPolicy("自定义名称1",builder =>{ builder.WithOrigins("http://example.com","http://www.contoso.com");});});
  //使用：
  app.UseCors("自定义名称1"); //方式一：configure中使用
  [EnableCors("自定义名称1")];//方式二：在接口方法上注解
  //Hsts、HttpsRedirection
  使用反向代理配置部署的应用允许代理处理 (HTTPS) 的连接安全。 如果代理还处理 HTTPS 重定向，则无需使用 HTTPS 重定向中间件。 如果代理服务器还处理写入 HSTS 标头 (例如， IIS 10.0 (1709) 或更高版本) 中的本机 HSTS 支持，则该应用程序不要求 HSTS 中间件。
  //StaticFiles静态文件
  app.UseStaticFiles();//支持访问根目录下的静态文件：css/image/js等
  ~~~

  ~~~c#
  //自定义中间件
  //方式一：对IApplicationBuilder 扩展封装，直接通过
  app.UseExceptionHandlerMiddleware();
  public static IApplicationBuilder UseExceptionHandlerMiddleware(this IApplicationBuilder builder)
          {
              return builder.UseMiddleware<ExceptionHandlerMiddleware>();
          }
  //方式二：不封装
  app.UseMiddleware<Middleware1>();//调用
  class Middleware1{
      private readonly RequestDelegate _next;
          public Middleware1(RequestDelegate next)
          {
              _next = next;
          }
      // 中间件要执行的逻辑
          public async Task Invoke(HttpContext context)
          {
              Console.WriteLine("Request1");//请求进来时逻辑，为了验证顺序性，打印一句话
              await _next(context);//把context传进去执行下一个中间件
              await context.Response.WriteAsync("<p>Response1</p>");
          }
  }
  ~~~

##### 依赖注入

- 即通过interface解除强依赖，再通过构造函数传参的方式注入

  ~~~c#
  public void ConfigureServices(IServiceCollection services)
  {
      //暂时生存期服务是每次从服务容器进行请求时创建的。在处理请求的应用中，在请求结束时会释放临时服务。
      services.AddTransient<IOperationTransient, Operation>();
      // 每个客户端请求（连接）一次的方式创建，在请求结束时会释放有作用域的服务。
      services.AddScoped<IOperationScoped, Operation>();
      services.AddSingleton<IOperationSingleton, Operation>();
  }
  ~~~