# Security
##### Security

Security细分为**<font color="#8B0000">认证</font>**和**<font color="#8B0000">授权</font>**两块，认证判断你是谁，授权判断你能干什么。

- 小结：
  - spring security 的核心是基于filter
  - 入口filter是springSecurityFilterChain(它会被DelegatingFilterProxy委托来执行过滤任务)
  - springSecurityFilterChain实际上是FilterChainProxy(一个filter)
  - <font color="#8B0000">FilterChainProxy</font>里面有一个`List<SecurityFilterChain>`集合，doFilter中遍历集合直到返回true，则跳出遍历。

- 初始化:

  - <font color="#8B0000">`@EnableWebSecurity`</font>注解中包含<font color="#8B0000">`@Import({WebSecurityConfiguration.class）`</font>

  - <font color="#8B0000">`WebSecurityConfiguration`</font>类中方法`springSecurityFilterChain()`通过读取<font color="#8B0000">`WebSecurityConfigurerAdapter`</font>中的配置，最后交给`WebSecurity`类构建`FilterChainProxy`。

    ~~~java
    public final class WebSecurity extends
          AbstractConfiguredSecurityBuilder<Filter, WebSecurity> implements
          SecurityBuilder<Filter>, ApplicationContextAware {
        
        @Override
    	protected Filter performBuild() throws Exception {
    		int chainSize = ignoredRequests.size() + securityFilterChainBuilders.size();
            // 我们要找的 securityFilterChains
    		List<SecurityFilterChain> securityFilterChains = new ArrayList<SecurityFilterChain>(
    				chainSize);
    		for (RequestMatcher ignoredRequest : ignoredRequests) {
    			securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
    		}
    		for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
    			securityFilterChains.add(securityFilterChainBuilder.build());
    		}
            // 创建 FilterChainProxy  ，传入securityFilterChains
    		FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
    		if (httpFirewall != null) {
    			filterChainProxy.setFirewall(httpFirewall);
    		}
    		filterChainProxy.afterPropertiesSet();
    
    		Filter result = filterChainProxy;
    		postBuildAction.run();
    		return result;
    	}
    }
    ~~~

- 执行过滤：
  - `FilterChainProxy.doFilter()`方法中，通过遍历`List<SecurityFilterChain>`找出第一个match的`SecurityFilterChain`执行filter。

### 认证

几个关键对象<font color="#8B0000">Authentication</font>、<font color="#8B0000">UserDetail</font>、<font color="#8B0000">User</font>、<font color="#8B0000">UserPasswordAuthenticationToken</font>

几个关键接口AuthenticationManager、ProviderManager、AuthenticationProvider、UserDetailService、AuthenticationToken。

~~~java
// 定义认证规范
public interface AuthenticationManager {
   Authentication authenticate(Authentication authentication)
}
~~~

~~~java
// 认证提供接口，定义了认证方法和支持方法supports(),用于限定支持认证的Filter。
public interface AuthenticationProvider {
   Authentication authenticate(Authentication authentication);
   boolean supports(Class<?> authentication);
}
// supports实现示例，限定能够支持的filter，这里为UsernamePasswordAuthenticationToken
pulic boolean supports(Class<?> authentication){
   return (UsernamePasswordAuthenticationToken.class
.isAssignableFrom(authentication)); 
}
~~~

~~~java
// 官方默认提供的认证provider
public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider 
~~~

~~~java
// 实现了AuthenticationManager类，用于管理Privoders，并执行认证方法:遍历Privoders，执行support()==true的Provider，并返回结果Antuentication。
public class ProviderManager implements AuthenticationManager{
       private List<AuthenticationProvider> providers = Collections.emptyList();
       public Authentication authenticate(Authentication authentication){
           for (AuthenticationProvider provider : getProviders()) {
		        if (!provider.supports(toTest)) continue;
		        try {
			        result = provider.authenticate(authentication);
               }
       }
   }
~~~

~~~java
//定义了用户信息服务接口抽象，支持用户通过实现该接口自定义用户信息(包括角色等)获取方式：缓冲、数据库。
public interface UserDetailsService {
   UserDetails loadUserByUsername(String username);
}
// 自定义实现UserDetailService
public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
   // 根据用户名从DB获取用户角色信息，放入集合roles中
   ArrayList<GrantedAuthority> roles = new  ArrayList<GrantedAuthority>();
   roles.add(new SimpleGrantedAuthority(""));
   return User.withUsername("chenh").password("pwd").roles(roles).build();
}
~~~

~~~java
// 默认Provider抽象类，实现了默认认证方法authenticate。
public abstract class AbstractUserDetailsAuthenticationProvider implements
		AuthenticationProvider, InitializingBean, MessageSourceAware {
    public Authentication authenticate(Authentication authentication){
       // 1.retrieveUser获取用户信息，子类调用UserDetailService.loadUserByUsername获取用户信息
       // 2.additionalAuthenticationChecks匹配密码
       // 以上2个方法均由子类DaoAuthenticationProvider实现(模板方法设计模式)
    }
}
~~~

- ##### 认证调用过程

  <font color="#8B0000">`UserPasswordAuthenticationFilter`</font>通过<font color="#8B0000">`UserPasswordAuthenticationToken`</font>将请求信息封装为<font color="#8B0000">`Authentication`</font>，调用<font color="#8B0000">`AuthenticationManager.authenticate()`</font>最终由Provider实现类DaoAuthenticationProvider执行：

  1. 调用<font color="#8B0000">`UserDetailService.loadUserByUsername`</font>获取用户密码权限信息。 
  2. 比较`Authentication`中的前端输入密码和查询出的后端密码是否一致。
  3. 认证通过后，将用户权限信息、认证结果更新到Authentication中，并存入上下文SecurityContext(保存在SecurityContextHolder中)。

### 授权

几个关键的类和接口FilterSecurityInterceptor、SecurityMetadataSource、AccessDecisionManager

- ##### 授权调用过程

  <font color="#8B0000">`FilterSecurityInterceptor.invoke`</font>方法内通过调用父类的beforeInvocation()方法实现授权验证：

  ~~~java
  public abstract class AbstractSecurityInterceptor implements InitializingBean,
  		ApplicationEventPublisherAware, MessageSourceAware {
      protected InterceptorStatusToken beforeInvocation(Object object) {
          ...
          Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource()
  				.getAttributes(object); // 获取用户请求的url-权限关系集合map
          ...
          // 获取用户在认证阶段放入Authentication的拥有的权限
          Authentication authenticated = authenticateIfRequired();
          try {
              // 验证权限，未通过则报AccessDeniedException异常
  			this.accessDecisionManager.decide(authenticated, object, attributes);
  		}
  		catch (AccessDeniedException accessDeniedException) {
  			publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated,accessDeniedException)); //并通知执行验证失败相关handler。
  			throw accessDeniedException; // 抛出异常
  		}
          ...
  }
  ~~~

  要实现动态url权限验证，可以通过自定义MetadataSource返回`Collection<ConfigAttribute> `集合。

  通过实现接口：

  ```java
  public interface FilterInvocationSecurityMetadataSource extends SecurityMetadataSource {
  }
  ```

  实现父接口的方法

  ~~~java
  public interface SecurityMetadataSource extends AopInfrastructureBean {
      // 从db获取url的权限集合
      Collection<ConfigAttribute> getAttributes(Object object);
      boolean supports(Class<?> clazz);
  }
  ~~~

  系统默认实现类

  ```java
  public class DefaultFilterInvocationSecurityMetadataSource implements
  		FilterInvocationSecurityMetadataSource {
      public Collection<ConfigAttribute> getAttributes(Object object) {
  		final HttpServletRequest request = ((FilterInvocation) object).getRequest();
  		for (Map.Entry<RequestMatcher, Collection<ConfigAttribute>> entry : requestMap
  				.entrySet()) {
  			if (entry.getKey().matches(request)) {
  				return entry.getValue();
  			}
  		}
  		return null;
  	}
      public boolean supports(Class<?> clazz) {
  		return FilterInvocation.class.isAssignableFrom(clazz);
  	}
  }
  ```

