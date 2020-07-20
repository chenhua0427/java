# 认证
## AuthenticationManager
> 定义认证规范
    public interface AuthenticationManager {{
        Authentication authenticate(Authentication authentication)
    }
## AuthenticationProvider
> 认证提供接口，定义了认证方法和支持方法supports(),用于限定支持认证的Filter。
    public interface AuthenticationProvider {
        Authentication authenticate(Authentication authentication);
        boolean supports(Class<?> authentication);
    }
    // supports实现示例
    pulic boolean supports(Class<?> authentication){
        return (UsernamePasswordAuthenticationToken.class
    .isAssignableFrom(authentication)); 
    }
> DaoAuthenticationProvider 是官方提供的默认Provider
## ProviderManager
> 实现了AuthenticationManager类，用于管理Privoders，并执行认证方法:遍历Privoders，执行support()==true的Provider，并返回结果Antuentication。
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
## UserDetailService
> 定义了用户信息服务接口抽象，支持用户通过实现该接口自定义用户信息(包括角色等)获取方式：缓冲、数据库。
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
## AbstractUserDetailsAuthenticationProvider
> 默认Provider抽象类，实现了默认认证方法authenticate。
    public Authentication authenticate(Authentication authentication){
        // 1.retrieveUser获取用户信息，子类调用UserDetailService.loadUserByUsername获取用户信息
        // 2.additionalAuthenticationChecks匹配密码
        // 以上2个方法均由子类DaoAuthenticationProvider实现(模板方法设计模式)
    }
## SecurityContextHolder
> 在当前上下文中保存了认证信息Authentication，通过SecurityContextHolder.getContext().setAuthentication/getAuthentication设置和获取
## 几个关键的对象
>Authentication、UserDetail、User、UserPasswordAuthenticationToken
## 认证调用过程
> UserPasswordAuthenticationFilter通过UserPasswordAuthenticationToken将请求信息封装为Authentication，调用authenticate()最终由Provider实现类DaoAuthenticationProvider执行：  
>>1.调用UserDetailService.loadUserByUsername获取用户密码权限信息；  
>>2.比较Authentication中的前端输入密码和查询出的后端密码是否一致；  
>>3.认证通过后，将用户权限信息、认证结果更新到Authentication中，并存入上下文SecurityContextHolder。
## 自定义token认证