---
title: SpringSecurity认证功能源码解析
date: 2024-10-07 23:16:00 +0800
categories: [SpringSecurity6, Authentication]
tags: [SpringSecurity6]
---





本文基于**SpringSecurity6.3**版本进行源码剖析，配套使用**SpringBoot3.3**，**JDK17**



## 1、SpringSecurity基本原理

![image-20241006180505636](https://s2.loli.net/2024/10/06/vNeuOMXQHhwsqbE.png) 



在Servlet应用下，经常使用Filter过滤器进行请求预处理，同时Servlet容器在启动时，会将所有Filter过滤器预加载到容器中；

SpringSecurity的原理也是基于Filter过滤器，但是在Spring项目中，Bean对象是由Spring容器进行管理的，

Spring容器和Servlet容器并不相通，因此SpringSecurity使用了**DelegatingFilterProxy**来作为沟通二者之间的桥梁，即委托代理。



在实际开发中，可能会单独针对某个url访问地址进行过滤链（**SecurityFilterChain**）的配置，因此抽象出了**FilterChainPorxy**实例，

由FilterChainProxy实例决定执行哪个过滤器链。





## 2、认证流程中关键组件

- **SecurityContextHolder**：SpringSecurity中存储验证通过后的身份信息的地方
- **SecurityContext**：位于SecurityContextHolder中，包含了用户的身份验证信息（Authentication）
- **Authentication**：用于存储用户身份验证信息的实例，常作为AuthenticationManager的输入
- **AuthenticationManager**：定义SpringSecurity中执行身份验证流程的API
- **ProviderManager**：AuthenticationManager的默认实现
- **AuthenticationProvider**：ProviderManager使用它来实现认证流程
- **DaoAuthenticationProvider**：AuthenticationProvider的默认实现
- **UserDetailsService**：DaoAuthenticationProvider使用其验证用户信息
- **PasswordEncoder**：DaoAuthenticationProvider使用其验证用户密码





## 3、默认过滤器链

SpringSecurity默认有如下配置，在此配置下中会存在15个过滤器

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(Customizer.withDefaults())
            .authorizeHttpRequests(authorize -> authorize
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())
            .formLogin(Customizer.withDefaults());
        return http.build();
    }

}
```

在**DefaultSecurityFilterChain**类中debug如下

![image-20241006182545630](https://s2.loli.net/2024/10/06/TJgq2FbY7BVfDQZ.png)

15个过滤器的详细信息如图：

![image-20241006182211724](https://s2.loli.net/2024/10/06/OjR5hnVPCxLAyFt.png) 

其中，**UsernamePasswordAuthenticationFilter**过滤器最为重要，是默认情况下SpringSecurity基于内存实现

用户登录、登出的关键过滤器；流程如下图所示

![在这里插入图片描述](https://s2.loli.net/2024/10/06/hPRV5cBLbXxK6mj.png)



### 3.1、UsernamePasswordAuthenticationFilter执行流程

查看UsernamePasswordAuthenticationFilter类中代码，发现关键函数如下 
![image-20241006190739243](https://s2.loli.net/2024/10/06/V3iL6XefNKkcjJd.png) 

#### 3.1.1、封装Authentication实例

这里使用UsernamePasswordAuthenticationToken封装用户名、密码等信息；该类间接实现了**Authentication**接口

```java
UsernamePasswordAuthenticationToken authRequest = UsernamePasswordAuthenticationToken
    .unauthenticated(username,password);
```

#### 3.1.2、调用authenticate函数进行认证

这里调用了**this.getAuthenticationManager()**获取AuthenticationManager实例

```java
return this.getAuthenticationManager().authenticate(authRequest);
```

#### 3.1.3、溯源this.getAuthenticationManager()

进入函数内部可发现，函数位于其父类**AbstractAuthenticationProcessingFilter**类中，代码如下

![image-20241006191526768](https://s2.loli.net/2024/10/06/mKGLqfsA48jIdOZ.png) 

在setter函数中进行debug操作，并在启动项目后成功进入debug；
这里可以看到是SpringSecurity中默认的**ProviderManager** 实例

![image-20241006191701763](https://s2.loli.net/2024/10/06/cJgXSZwPxNAa7d9.png) 

#### 3.1.4、溯源ProviderManager

通过查看线程执行流程可以得知：执行setter函数前执行了该**configure**函数

![image-20241006192040538](https://s2.loli.net/2024/10/06/8EmL4j3G1nudqz2.png) 

进入对应类中函数查看代码如下，并在调用setter函数时进行debug操作；

其中this.authFilter即为UsernamePasswordAuthenticationFilter实例
（这里如果继续深究可知：在其子类**FormLoginConfigurer** 的构造函数中调用了父类构造函数，
并传递了一个UsernamePasswordAuthenticationFilter实例，因此this.authFilter为对应实例）

![image-20241006192510543](https://s2.loli.net/2024/10/06/vsy2ni1wlfHU47S.png) 

debug进入http.getSharedObject(AuthenticationManager.class)函数中，查看代码如下；

可得知从hashMap集合**sharedObject** 中取出ProviderManager实例

![image-20241006192816574](https://s2.loli.net/2024/10/06/EPnvpazVXsrSIgK.png) 

到此不再继续溯源，SpringSecurity在启动时自动注册ProviderManager的bean对象，并将其存入**sharedObjects** 该map集合中





### 3.2、ProviderManager执行流程

查看ProviderManager中authenticate函数代码如下,其中关键代码如下

```java
result = provider.authenticate(authentication);
```

![image-20241006193325478](https://s2.loli.net/2024/10/06/oWfNVYHsc9nwTbm.png) 

#### 3.2.1、溯源provider实例

在默认表单下发起登录请求，debug代码如下，可以发现provider实例为**DaoAuthenticationProvider** 类型

![image-20241006193827148](https://s2.loli.net/2024/10/06/7mnug83NclWAJ6h.png) 

在构造函数中debug代码如下，发现在ProviderManager实例化时进行赋值操作

![image-20241006194153739](https://s2.loli.net/2024/10/06/RQ15HeMtN2vyYTa.png) 

到此不再溯源，之后仍是SpringSecurity默认操作





### 3.3、DaoAuthenticationProvider执行流程

查看DaoAuthenticationProvider中authenticate函数可知，该函数继承于**AbstractUserDetailsAuthenticationProvider**，

函数代码如下，其中关键代码为

```java
user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
```

![image-20241006194726732](https://s2.loli.net/2024/10/06/87TLK5xzXRpvfSg.png) 

#### 3.3.1、溯源UserDetailService实例

定位到DaoAuthenticationProvider类中该函数位置，其代码如下，关键代码为

```java
UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
```

![image-20241007004809758](https://s2.loli.net/2024/10/07/UYpI4MWPHdma6uZ.png)  

在默认表单下发起登录请求，debug代码如下，可以看到获取的实际UserDetailsService实例是**InMemoryUserDetailsManager** 类型

由该类型的实例调用**loadUserByUsername** 函数，在内存中获取用户信息

![image-20241007005404651](https://s2.loli.net/2024/10/07/LztVl8Amr2MJxRO.png) 





### 3.4、InMemoryUserDetailsManager执行流程

进入**loadUserByUsername**函数中，发现代码如下；

在本函数中，通过用户名在内存中查询出对应的用户信息，并封装进**UserDetails** 实例返回

![image-20241007005659350](https://s2.loli.net/2024/10/07/IhijZwkBnUPeu5L.png) 



### 3.5、用户密码校验流程（DaoAuthenticationProvider）

上文提到，DaoAuthenticationProvider实例调用内部authenticate函数进行认证流程，其中关键代码为

```java
user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
```

该函数返回一个**UserDetails** 类型的实例，其中包含了查询到的用户信息；
随后将查询到的用户信息与用户输入的信息作比对，代码如下，其中关键代码为

```java
// user为查询到的UserDetails类型的实例，内部包含了用户信息
// authentication则是用户输入信息封装成的实例
additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
```

![image-20241007174753714](https://s2.loli.net/2024/10/07/dGOnRVqLXDQvcPB.png) 



#### 3.5.1、用户信息校验逻辑

进入内部对应函数中，查看代码如下，其中关键代码为

```java
if (!this.passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
			this.logger.debug("Failed to authenticate since password does not match stored value");
			throw new BadCredentialsException(this.messages
				.getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
}
```

![image-20241007175927667](https://s2.loli.net/2024/10/07/XrJsBxqMNgOLVK6.png) 

可以发现，通过调用**passwordEncoder** 实例的**matches**函数将查询到的用户密码和输入的用户密码作比对；

若二者不同，则抛出异常。（在此不过多追溯passwordEncoder实例的具体类型和赋值流程）





## 4、实际开发中过滤器链



在实际的**前后端分离**开发中，存在一些必要的自定义配置如下

```java
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
             // 关闭http基础认证、csrf、默认登录登出表单、session管理
                .httpBasic(HttpBasicConfigurer::disable)
                .csrf(CsrfConfigurer::disable)
                .formLogin(FormLoginConfigurer::disable)
                .logout(LogoutConfigurer::disable)
                .sessionManagement(SessionManagementConfigurer::disable)
            // 除options请求、登录等请求之外，所有请求都需要认证通过
                .authorizeHttpRequests(authorize -> authorize
                        .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                        .requestMatchers(
                                "/auth/**").permitAll()
                        .anyRequest().authenticated()
                );

        return http.build();
    }
```

在此配置下，会有8个过滤器存在，可以看到最为关键的**UsernamePasswordAuthenticationFilter** 过滤器并不存在；
这正是因为在自定义配置中关闭了默认登录、登出表单页

![image-20241006182058411](https://s2.loli.net/2024/10/06/a4dHPU5ngV97pSx.png) 

在这种情况下，默认存在的可供访问的**/login、 /logout** 请求也会失效，所以需要自行开发接口，完善登录逻辑；




### 4.1、自定义认证流程

#### 4.1.1、自定义AuthenticationManager

在**UsernamePasswordAuthenticationFilter** 缺失的情况下，默认的认证流程同样不会再执行，
这是因为自定义配置SecurityFilterChain后，所剩余的8个过滤器并无执行认证流程的功能；

要解决此问题，可以通过自行获取**AuthenticationManager** 类型的实例并调用**authenticate** 函数进行认证流程，
在获取前，需要向IOC容器中注入对应类型的实例，一般为ProviderManager类型，示例代码如下

```java
    @Bean
    public AuthenticationManager authenticationManager(
        UserDetailsService userDetailsService, 
        PasswordEncoder passwordEncoder) {
        // 实例化
        DaoAuthenticationProvider authenticationProvider = new DaoAuthenticationProvider();
        // 设置需要调用的实例
        authenticationProvider.setUserDetailsService(userDetailsService);
        authenticationProvider.setPasswordEncoder(passwordEncoder);
		// 封装进ProviderManager实例中，并设置 认证后不删除凭证(密码)信息
        ProviderManager providerManager = new ProviderManager(authenticationProvider);
        providerManager.setEraseCredentialsAfterAuthentication(false);

        return providerManager;
        
    }
```

#### 4.1.2、自定义UserDetailsService

默认情况下，UserDetailsService类型的实例为**InMemoryUserDetailsManager**类型，基于内存对用户信息进行存取；

在实际开发中，用户信息总是存储在数据库中，而且默认认证流程多数情况下不符合业务需求，因此需要自定义UserDetailsService
的实现类并重写其函数，示例代码如下：

```java
@Component
@Log4j2
public class UserDetailsServiceImpl implements UserDetailsService {

    @Resource
    private AuthMapper authMapper;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 调用Mapper层接口，获取用户信息
        Wrapper<SysUser> wrapper = new LambdaQueryWrapper<SysUser>()
                .eq(SysUser::getUserName, username)
                .eq(SysUser::getDelFlag, 0);
        SysUser sysUser = authMapper.selectOne(wrapper);
        // 对用户信息作校验
        if (Objects.isNull(sysUser)) {
            log.error("不存在该用户");
            throw new TextAlertException("不存在该用户");
        }else if (Objects.equals(sysUser.getStatus(), 1)) {
            log.error("用户已停用");
            throw new TextAlertException("用户已停用");
        }
        // 返回UserDetails类型的实例（自定义其实现类）
        return new LoginUser()
                .setUsername(sysUser.getUserName())
                .setPassword(sysUser.getPassword());
    }

}
```

#### 4.1.3、自定义PasswordEncoder

在DaoAuthenticationProvider中执行认证流程时，会将获取的用户信息与输入的用户信息通过**passwordEncoder** 实例进行比对，

在实际开发中，经常使用bcrypt加密算法加密用户密码，因此需要提供一个**BCryptPasswordEncoder** 类型的实例交由IOC容器进行管理
示例代码如下

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

#### 4.1.4、调用authenticate函数，发起认证流程

由于过滤器链中缺失了UsernamePasswordAuthenticationFilter，因此我们需要获取AuthenticationManager的实例（ProviderManager）并主动调用**autenticate** 函数发起认证流程；

由于之前已经配置过一系列的**AuthenticationManager** 、 **UserDetailsService** 、 **PasswordEncoder** 等示例，因此会按照指定的流程继续进行认证操作，示例代码如下

```java
    @Resource
    private AuthenticationManager authenticationManager;

    @Override
    public void login(LoginUser loginUser) {
        // 封装用户信息为authentication实例
        UsernamePasswordAuthenticationToken authenticationToken =
                new UsernamePasswordAuthenticationToken(loginUser.getUsername(), loginUser.getPassword());
        // 调用函数执行认证流程
        Authentication authentication = authenticationManager.authenticate(authenticationToken);
        if (Objects.isNull(authentication)) {
            throw new TextAlertException("用户认证失败");
        }
        // TODO 登录成功-生成token

    }
```





## 5、Token认证过滤器

在实际的开发中，用户信息认证通过后将会生成一个token信息，传递给前端；

自此前端在每次发起请求时都会携带该token，后端则会对该token进行校验：
	若信息无误且未到认证过期时间则放行请求
	若信息有误或已到认证过期时间则直接返回结果，不再执行后续流程

要实现此功能，即需要自定义一个token认证过滤器，在每次请求时对请求进行拦截，并校验其中的token信息，
由于在过滤器链中，真正的认证流程是由**UsernamePasswordAuthenticationFilter** 内部开始的，因此
把自定义的token认证过滤器放在该过滤器前即可。



### 5.1、自定义token认证过滤器

自定义token认证过滤器，示例代码如下：

可以看到，该过滤器继承了**OncePerRequestFilter** 这个过滤器的基类，确保对每个请求只进行一次拦截，这里不对其实现原理作具体阐述。

```java
@Component
@Log4j2
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Resource
    private RedisCache redisCache;

    @Override
    protected void doFilterInternal(HttpServletRequest request, @NonNull HttpServletResponse response, @NonNull FilterChain filterChain) throws ServletException, IOException {
        // 获取请求头
        String authorization = request.getHeader("Authorization");
        if (Objects.isNull(authorization)) {
            filterChain.doFilter(request, response);
            return;
        }
        // 解析
        String token = authorization.substring(6).trim();
        String userId;
        try {
            Claims claims = JwtUtil.parseClaims(token);
            userId = claims.getSubject();
        }catch (Exception e) {
            log.error("token解析失败");
            request.setAttribute("filter.error", "token解析失败");
            // 将异常分发到对应控制器，交由全局异常处理器返回
            request.getRequestDispatcher("/error/throw").forward(request, response);
            return;
        }
        // 校验身份信息
        LoginUser cacheObject = redisCache.getCacheObject("login-user:" + userId);
        if (Objects.isNull(cacheObject)) {
            log.error("用户尚未登录");
            request.setAttribute("filter.error", "用户尚未登录");
            // 将异常分发到对应控制器，交由全局异常处理器返回
            request.getRequestDispatcher("/error/throw").forward(request, response);
            return;
        }
        // 校验成功，放行请求
        filterChain.doFilter(request, response);
    }
}
```



### 5.2、设置过滤器执行优先级

由于用户token信息的校验流程始终应该是位于**UsernamePasswordAuthenticationFilter** 认证过滤器之前执行，因此需要将该过滤器的优先级设置于认证过滤器之前，修改过滤器链代码如下：

这里有个疑问：上文中我们看到在实际开发的过滤器链中，并不存在UsernamePasswordAuthenticationFilter过滤器，那么在此处设置过滤器链的位置是否能生效？

答案：可以，因为在SpringSecurity中，项目启动时会将所需的过滤器链优先级提前设置好（优先级以**Integer** 类型的整数来判定），因此即使UsernamePasswordAuthenticationFilter过滤器实际不存在于过滤器链中，但是仍可以获取其优先级，并将JWT认证过滤器置于更高一级的执行位置。

```java
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        // 设置认证相关，在此省略不写
        ...
        // 设置过滤器执行优先级
        http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
```

