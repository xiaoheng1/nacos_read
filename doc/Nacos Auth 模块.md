关于安全如何设计：

@Retention(RetentionPolicy.RUNTIME)
public @interface Secured {
    
    /**
     * The action type of the request.
     *
     * @return action type, default READ
     */
    ActionTypes action() default ActionTypes.READ;
    
    /**
     * The name of resource related to the request.
     *
     * @return resource name
     */
    String resource() default StringUtils.EMPTY;
    
    /**
     * Resource name parser. Should have lower priority than resource().
     *
     * @return class type of resource parser
     */
    Class<? extends ResourceParser> parser() default DefaultResourceParser.class;
}


关于权限使用：com.alibaba.nacos.core.auth.AuthFilter#doFilter

权限模块比较简单，依托于 Spring，定义 Secured 注解，然后在 Filter 中解析 Secured，完成权限校验工作.

Permission(resource, action) 对资源的操作.

Nacos 权限控制方案中分为两部分，都是通过 AuthManager 来完成的.

1.身份识别
2.权限识别

关于身份识别，采用的是 JWT Token，而权限识别，采用的是 RBAC 模型.

身份识别，参考 AuthManager.login


SecurityContextPersistenceFilter
|
UsernamePasswordAuthenticationFilter
|
AuthenticationManager
|
AuthenticationProvider
|
AbstractUserDetailsAuthenticationProvider
|
DaoAuthenticationProvider

用户需要实现 UserDetailsService 接口.


JWT token.


JWT & Spring-Secutiry 整合：com.alibaba.nacos.console.security.nacos.NacosAuthConfig

参考：https://www.cnblogs.com/zzjlxy-225223/p/11290900.html


参考：
1.https://blog.csdn.net/u010889990/article/details/105154123/
2https://wanglinyong.github.io/2018/07/10/Spring-Security%E7%99%BB%E5%BD%95%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E5%8E%9F%E7%90%86/