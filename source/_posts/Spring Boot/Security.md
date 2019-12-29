---
title: Spring Boot Security

date: 2019-12-29

categories: 
- Spring Boot

tags:
- 教程
---
一个能够为基于Spring的企业应用系统提供声明式的安全訪问控制解决方式的安全框架（简单说是对访问权限进行控制嘛），应用的安全性包括用户认证（Authentication）和用户授权（Authorization）两个部分。spring security的主要核心功能为 认证和授权，所有的架构也是基于这两个核心功能去实现的。
<!--more -->

### 在spring boot项目中集成security

在pom文件中引入依赖
```

        <!--security-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-data</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!--jwt-->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>0.7.0</version>
        </dependency>
```

### 登录时认证

```
    public Object login(SignInForm form) {

        Authentication authentication;

//        try {
            authentication = authenticationManager.authenticate(new UsernamePasswordAuthenticationToken
                    (form.getUsername(), form.getPassword()));
//        } catch (Exception e) {
//            throw new CommonException(e.getMessage());
//        }

        return tokenProvider.createToken(authentication, form.getRememberMe());
    }
```

security使用Authentication对象来描述当前用户的信息， AuthenticationManager是security自带的一个类，使用@Autowired注解注入即可。

UsernamePasswordAuthenticationToken是security使用username和password生成的一个Authentication对象，拿到该对象后还要进行其它的操作如验证用户名密码是否正确，获取用户角色权限等。

```
@Component
public class UsernamePasswordAuthenticationProvider implements AuthenticationProvider {


    @Autowired
    private UserDetailsService securityUserDetailsService;


    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        renull
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return (UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication));
    }

}
```

可以用的扩展逻辑并不是直接用的继承UsernamePasswordAuthenticationToken，而是实现AuthenticationProvider接口，在supports()方法里表明当前类实现的是哪个AuthenticationToken类。 authenticate()方法里用来描述主要业务逻辑。

下面来看下authenticate()方法的业务逻辑

```
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    
        UsernamePasswordAuthenticationToken token = (UsernamePasswordAuthenticationToken) authentication;

        SecurityUserDetails userDetails = (SecurityUserDetails) securityUserDetailsService.loadUserByUsername((String) authentication.getPrincipal());

        if (!userDetails.getPassword().equals(authentication.getCredentials())) {
            // 自定义异常类，继承自RuntimeException
            throw new CommonException("密码错误");
        }

        JWTUser jwtUser = new JWTUser();
        jwtUser.setOpenId(userDetails.getOpenId());
        jwtUser.setUserId(userDetails.getUserId());
        jwtUser.setUsername(userDetails.getUsername());
        token.setDetails(jwtUser);

        return new UsernamePasswordAuthenticationToken(jwtUser, null, userDetails.getAuthorities());
    }
    
```

UserDetailsService是security获取用户信息的接口，需要我们自己去实现。

```
@Service
public class SecurityUserDetailsService implements UserDetailsService {

    @Autowired
    private UserDAO userDAO;

    @Autowired
    private UserRoleDAO userRoleDAO;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        User user = userDAO.findByUsername(username);
        if (user == null) {
            return null;
        }

        List<UserRole> userRoles = userRoleDAO.findAllByUserId(user.getId());
        return new SecurityUserDetails(user.getId(), user.getOpenId(), username, user.getPassword(), getUserAuthorities(userRoles));
    }

    private List<GrantedAuthority> getUserAuthorities(List<UserRole> userRoles) {
        List<GrantedAuthority> authorities = new ArrayList<>();
        for (UserRole userRole : userRoles) {
            authorities.add(new SimpleGrantedAuthority("ROLE_" + userRole.getRoleCode()));
        }
        return authorities;
    }
}
```


最后返回的SecurityUserDetails类继承了security的User类

```
@Getter
@Setter
public class SecurityUserDetails extends User {

    private Long userId;

    private String openId;

    public SecurityUserDetails(Long userId, String openId, String username, String password,  Collection<? extends GrantedAuthority> authorities) {
        super(username, password, true, false, false, false, authorities);
        this.userId = userId;
        this.openId = openId;
    }
}


// User类构造方法之一

    public User(String username, String password, boolean enabled, boolean accountNonExpired, boolean credentialsNonExpired, boolean accountNonLocked, Collection<? extends GrantedAuthority> authorities) {
        if (username != null && !"".equals(username) && password != null) {
            this.username = username;
            this.password = password;
            this.enabled = enabled;
            this.accountNonExpired = accountNonExpired;
            this.credentialsNonExpired = credentialsNonExpired;
            this.accountNonLocked = accountNonLocked;
            this.authorities = Collections.unmodifiableSet(sortAuthorities(authorities));
        } else {
            throw new IllegalArgumentException("Cannot pass null or empty values to constructor");
        }
    }

```

User类的构造方法有多个参数，我们主要关注的是username，password，authorities，  authorities参数用来存放登录用户的角色权限等信息，存放角色信息时要加上"ROLE_"开头让security知道这是一个角色，比如admin角色存入authorities是"ROLE_admin"

回过头看authenticate()方法的最后一行

```
return new UsernamePasswordAuthenticationToken(jwtUser, null, userDetails.getAuthorities());
```
返回了一个Authentication对象，先来看下该类的构造方法参数

```
    public UsernamePasswordAuthenticationToken(Object principal, Object credentials, Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.principal = principal;
        this.credentials = credentials;
        super.setAuthenticated(true);
    }
```
principal参数存放主要信息，这里使用JWTUser类封装了username，userId,openId等信息存入。credentials存放一些额外的信息。

### 生成token

获取到要用的Authentication对象后，用它来生成一个token，这里用的是jwt。

```
@Component
public class TokenProvider {


    // 配置文件中的签名，token时长等信息，可以直接在代理中写死（不建议）

    @Value("${jwt.secret_key}")
    private String secretKey;

    @Value("${jwt.token_validity}")
    private long tokenValidity;

    @Value("${jwt.token_validity_remember_me}")
    private long tokenValidityRememberMe;


    /**
     * 生成token
     */
    public String createToken(Authentication authentication, Boolean rememberMe) {
        long now = (new Date()).getTime();
        Date validity = rememberMe ? new Date(now + this.tokenValidityRememberMe * 1000) : new Date(now + this
                .tokenValidity * 1000);

        JWTUser authUser = (JWTUser) authentication.getPrincipal();
        Long userId = authUser.getUserId();
        String openId = authUser.getOpenId();
        String username = authUser.getUsername();

        return Jwts.builder().setSubject(authentication.getName())
                .claim("userId", userId.toString())
                .claim("openId", openId)
                .claim("username", username)
                .claim("roles", getRoleFromAuthorities(authentication.getAuthorities()))
                .signWith(SignatureAlgorithm.HS256, secretKey)
                .setExpiration(validity)
                .compact();
    }


    private String getRoleFromAuthorities(Collection<? extends GrantedAuthority> authorities) {
        List<String> roles = new ArrayList<>();
        for (GrantedAuthority authority : authorities) {
            if (authority instanceof SimpleGrantedAuthority) {
                roles.add(authority.getAuthority());
            }
        }
        return String.join(",", roles);
    }
}

```

createToken()方法就是用来生成token字符串返回给前端，之后请求其它接口时将token传入用来验证用户信息。比起存入session，jwt是一种以时间换空间的策略，又不会有存cookie跨域问题。

### token验证

以上讲的主要是登录时用security生成token，在那之后使用该token请求接口时又要怎么解析并获取用户信息。一般在项目里是用过滤器或者拦截器进行拦截解析的。



```
public class JWTFilter extends GenericFilterBean {


    private TokenProvider tokenProvider;

    public JWTFilter(TokenProvider tokenProvider) {
        this.tokenProvider = tokenProvider;
    }


    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain)
            throws IOException, ServletException {

        HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
        String token = getTokenForHeader(httpServletRequest);
        if (StringUtils.hasText(token)) {
            if (tokenProvider.validateToken(token)) {
                Authentication authentication = tokenProvider.getAuthentication(token);
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        }

        filterChain.doFilter(servletRequest, servletResponse);

    }

    private String getTokenForHeader(HttpServletRequest request) {
        //获取header里的token
        String token = request.getHeader(“Authorization”);
        if (StringUtils.hasText(token)) {
            return token;
        }
        return null;
    }
}




@Component
public class TokenProvider {

    /**
     * 获取
     */
    public Authentication getAuthentication(String token) {
        Claims claims = Jwts.parser()
                .setSigningKey(secretKey)
                .parseClaimsJws(token)
                .getBody();

        JWTUser jwtUser = null;

        String openId = claims.get("openId", String.class);
        Long userId = Long.valueOf(claims.get("userId").toString());
        String username = claims.get("username", String.class);
        jwtUser = new JWTUser(userId, username, openId);

        return new UsernamePasswordAuthenticationToken(jwtUser, null, getAuthoritiesFromRole(claims.get("roles", String.class)));
    }


    /**
     * token验证
     */
    public boolean validateToken(String authToken) {
        try {
            Jwts.parser().setSigningKey(secretKey).parseClaimsJws(authToken);
        } catch (Exception e) {
            return false;
        }
        return true;
    }


    private List<SimpleGrantedAuthority> getAuthoritiesFromRole(String roles) {
        List<SimpleGrantedAuthority> authorities = new ArrayList<>();
        String[] roleArray = roles.split(",");
        for (String role : roleArray) {
            authorities.add(new SimpleGrantedAuthority(role));
        }
        return authorities;
    }
}

```


在Filter中，我们在请求头中获取用户token，先token合法性进行解析，解析成功后，像登录时生成一个Authentication对象存储。

doFIlter()方法里

```
SecurityContextHolder.getContext().setAuthentication(authentication);
```

之后要全局获取Authentication可以自己进行一层封装

```
public final class SecurityUtils {

    public static JWTUser getCurrentUser() {

        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication != null) {
            if (authentication.getPrincipal() instanceof JWTUser) {
                return (JWTUser) authentication.getPrincipal();
            }
        }

        return null;
    }
}
```


### 过滤器配置


#### 过滤器顺序
```
public class JWTConfigurer extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {


    private TokenProvider tokenProvider;

    public JWTConfigurer(TokenProvider tokenProvider) {
        this.tokenProvider = tokenProvider;
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {
        JWTFilter jwtFilter = new JWTFilter(tokenProvider);
        http.addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
    }
}

```


#### 过滤路径
```
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    private final TokenProvider tokenProvider;

    public SecurityConfig(TokenProvider tokenProvider) {
        this.tokenProvider = tokenProvider;
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .csrf()
                .disable()
                .headers()
                .frameOptions()
                .disable()
                .and()
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests()
                //  yml配置的context-path: /base  不能加上
                .antMatchers("/test/**").permitAll()
                .antMatchers("/user/signIn").permitAll()
                .antMatchers("/**").authenticated()
                .and()
                .exceptionHandling()
                .accessDeniedHandler(accessDeniedHandler())
                .authenticationEntryPoint(authenticationEntryPoint())
                .and()
                .apply(securityConfigurerAdapter())
        ;

    }


    private JWTConfigurer securityConfigurerAdapter() {
        return new JWTConfigurer(tokenProvider);
    }

    @Bean
    public AccessDeniedHandler accessDeniedHandler() {
        return new AuthAccessDeniedHandler();
    }

    @Bean
    public AuthenticationEntryPoint authenticationEntryPoint() {
        return new LoginAuthenticationEntryPoint();
    }

}

```


在Filter中如果解析token失败会造成请求500错误，无法正确返回我们要的信息， AuthAccessDeniedHandler类和LoginAuthenticationEntryPoint是针对其中两种异常返回自定义的异常信息


```
public class AuthAccessDeniedHandler implements AccessDeniedHandler {


    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       AccessDeniedException e) throws IOException, ServletException {

        response.setContentType("application/json");
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        CommonResult result = new CommonResult("Prohibition of access");
        result.setCode("403");
        PrintWriter out = response.getWriter();
        out.print(JSON.toJSONString(result));
        out.flush();
        out.close();
    }
}
```

### 角色注解拦截

在controller层的方法里加上@PreAuthorize进行权限拦截

```
    @PreAuthorize("hasRole('admin')")
    @GetMapping("/userInfo")
    public CommonResult getUserInfo() {
        return CommonResultTemplate.execute(() -> userManager.getUserInfo());
    }
```
@PreAuthorize("hasRole('admin')")表示admin角色才能访问该接口，@PreAuthorize注解支出SPEL表达式