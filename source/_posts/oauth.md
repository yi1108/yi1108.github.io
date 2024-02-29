---
title: oauth
date: 2023-1-8 11:54:10
tags:
- 认证
- springsecurity
---



## 2

### 认证和授权的概念

​	认证即解决 我是谁 

​	授权即解决 我能做什么

​	

```plain
@EnableWebSecurity(debug = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //检查请求是否认证
        http.authorizeRequests(req -> req.antMatchers("/api/**").authenticated());
        //检查请求是否有权限
        http.authorizeRequests(req -> req.antMatchers("/api/**").hasRole("ADMIN"));
    }
}
```

### 过滤器和过滤器链

​		任何Spring Web 应用本质上都是一个servlet，Security Filter 在HTTP请求到达Controller之前会过滤每一个请求。

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742193200-1dd41ba7-526c-445b-ba97-7441e87a9c6d.png)

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742193043-e555c6c8-d67c-4a21-8444-e27927269e84.png)

使用过滤器链的好处： 职责单一、将负责逻辑简单化。

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742193135-87d01abf-934c-4492-ba5f-f4ac967cd088.png)

### HTTP 请求的结构

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742193179-7bea4e7e-55bc-4638-9b1a-eb244782070e.png)

```plain
POST localhost:8080/api/user/hello2?name=安慰
Authorization: Basic user 15afce2c-a151-4507-800a-9805ca227506
```

### HTTP 响应和 HTTP Basic Auth

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742193065-872dd716-bfc7-4f22-bac1-ffc68f58a9d9.png)

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742194191-a18d9557-36a9-4426-9969-bf5fce89a71d.png)

客户端请求服务端，服务端返回未授权，同时headle 里返回指定的认证方式，客户端收到响应后让用户填写信息发送服务端，服务端认证通过后返回通过状态码

### 安全配置

security 标准传统写法 and() 会返回一个HttpSecurity 继续配置

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742194475-d251ee1c-a9c9-46bf-8102-c618fc364c3d.png)

也可以使用函数式写法

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742194625-9544af7e-6fcf-48aa-b5ad-ec33e3f1c24c.png)

### 定制登录页

​		使用传统模板引擎 加入thymeleaf webjars 依赖 

```java
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.projectlombok:lombok'
    implementation('org.springframework.boot:spring-boot-starter-thymeleaf')
    implementation('org.webjars:bootstrap:4.5.3')
    implementation('org.webjars:webjars-locator-core')
}
```


​		配置SecurityConfig

```plain
@EnableWebSecurity(debug = true)
@Slf4j
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http
                //配置需要拦截的请求  req.anyRequest() 如何页面都需要认证
                .authorizeRequests(req -> req.antMatchers("/api/**").authenticated())
                
                //配置登录  使用全部过滤需要加.permitAll()
                .formLogin(from -> from.loginPage("/login"))
            
                .csrf(Customizer.withDefaults())
        ;

    }

    /**
     * 静态资源不拦截 不启动安全拦截
     *
     * @param web
     * @throws Exception
     */
    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring().antMatchers("/public/**", "/css/**", "/js/**", "/images/**")
                 //常用静态资源地址
                .requestMatchers(PathRequest.toStaticResources().atCommonLocations());
    }
}
```

​		配置mvcConfig

```plain
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/webjars/**")
                .addResourceLocations("/webjars/**")
                .resourceChain(false); // 不缓存
        registry.setOrder(1)    ;
    }

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry
                .addViewController("/login")//url
                .setViewName("login");//视图html名称
    }
}
```

即指定了自己的登录页面

### 登录成功及失败的处理

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742194905-79230537-d43d-4a8b-9ff7-138f8b31c108.png)

登录成功后返回json 由前端判断路由 

```plain
.formLogin(from -> from.loginPage("/login")
                        //传入AuthenticationSuccessHandler 类中的onAuthenticationSuccess方法，可以用函数式接口
                        //认证成功后的处理
                        .successHandler(getAuthenticationSuccessHandler())
                        //认证失败的处理器
                        .failureHandler(getAuthenticationFailureHandler())
                        .permitAll())
     
     
 private static AuthenticationSuccessHandler getAuthenticationSuccessHandler() {
        return (req, resp, auth) -> {
            resp.setStatus(HttpStatus.OK.value());
            resp.setContentType("application/json;charset=utf-8");
            resp.getWriter().write( new ObjectMapper().writeValueAsString(auth));
        };
    }

    private static AuthenticationFailureHandler getAuthenticationFailureHandler() {
        return (req, resp, e) -> {
            resp.setStatus(HttpStatus.UNAUTHORIZED.value());
            resp.setContentType("application/json;charset=utf-8");
            resp.getWriter().write(new ObjectMapper().writeValueAsString(e.getMessage()));
        };
    }
```

spring 自带的登录成功处理器 -传统项目用到 前后端分离的不适合

```plain
/**
     * Calls the parent class {@code handle()} method to forward or redirect to the target
     * URL, and then calls {@code clearAuthenticationAttributes()} to remove any leftover
     * session data.
     */
    public void onAuthenticationSuccess(HttpServletRequest request,
            HttpServletResponse response, Authentication authentication)
            throws IOException, ServletException {

        handle(request, response, authentication);
        //清理存在session 中的信息 删除在身份验证过程中可能存储在会话中的临时身份验证相关数据。
        clearAuthenticationAttributes(request);
    }


    /**
     * Invokes the configured {@code RedirectStrategy} with the URL returned by the
     * {@code determineTargetUrl} method.
     * <p>
     * The redirect will not be performed if the response has already been committed.
     */
    protected void handle(HttpServletRequest request, HttpServletResponse response,
            Authentication authentication) throws IOException, ServletException {
        //记录访问的目标url
        String targetUrl = determineTargetUrl(request, response, authentication);

        if (response.isCommitted()) {
            logger.debug("Response has already been committed. Unable to redirect to "
                    + targetUrl);
            return;
        }
        //认证通过后重定向到目标url
        redirectStrategy.sendRedirect(request, response, targetUrl);
    }
```

### 自定义 Filter

Spring 默认用来处理登录的过滤器 UsernamePasswordAuthenticationFilter

```plain
public Authentication attemptAuthentication(HttpServletRequest request,
            HttpServletResponse response) throws AuthenticationException {
        if (postOnly && !request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException(
                    "Authentication method not supported: " + request.getMethod());
        }

        String username = obtainUsername(request);
        String password = obtainPassword(request);

        if (username == null) {
            username = "";
        }

        if (password == null) {
            password = "";
        }

        username = username.trim();
        //构造一个安全对象
        UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
                username, password);

        // Allow subclasses to set the "details" property
        setDetails(request, authRequest);
        //认证处理最终机制 
        return this.getAuthenticationManager().authenticate(authRequest);
    }
```

自己新建一个模仿默认过滤器写一个RestAuthenticationFilter

```plain
//@RequiredArgsConstructor//将flnal 修饰的私有成员变量放到构造函数中
public class RestAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

    private  ObjectMapper objectMapper;

    public RestAuthenticationFilter(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {

        UsernamePasswordAuthenticationToken authRequest;

        try {
            ServletInputStream inputStream = request.getInputStream();
            JsonNode jsonNode = objectMapper.readTree(inputStream);
            String username = jsonNode.get("username").textValue();
            String password = jsonNode.get("password").textValue();
            authRequest= new UsernamePasswordAuthenticationToken(username, password);
            setDetails(request, authRequest);
        } catch (IOException e) {
            e.printStackTrace();
            throw new BadCredentialsException("Invalid username or password");
        }

        return this.getAuthenticationManager().authenticate(authRequest);
    }
}
```

然后在securityConfig中把自定义filter 加入过滤器链中

选择addFilterAt 替代默认过滤器

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742194966-b9675495-94ef-4299-922b-637c984eb4f9.png)

```plain
@Override
    protected void configure(HttpSecurity http) throws Exception {

        http
                //配置需要拦截的请求
                .authorizeRequests(req -> req
                          //不需要拦截的路径
                        .antMatchers("/authorize/**").permitAll()
                          //配置需要权限的路径
                        .antMatchers("/admin/**").hasRole("ADMIN")
                        .antMatchers("/api/**").hasRole("USER")
                        .anyRequest().authenticated())
				//将自定义过滤器加入
                .addFilterAt(restAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class)

            /*将原来的登录配置去掉*/
                //配置登录
//                .formLogin(from -> from.loginPage("/login")
//                        //传入AuthenticationSuccessHandler 类中的onAuthenticationSuccess方法，可以用函数式接口
//                        //认证成功后的处理
//                        .successHandler(getAuthenticationSuccessHandler())
//                        //认证失败的处理器
//                        .failureHandler(getAuthenticationFailureHandler())
//                        .permitAll())
//                .logout(logout -> logout.logoutUrl("/perform_logout")
//                        .logoutSuccessHandler((request, response, authentication) -> {
//                            response.setStatus(HttpStatus.OK.value());
//                            response.setContentType("application/json;charset=UTF-8");
//                            response.getWriter().write(new ObjectMapper().writeValueAsString("注销成功"));
//                        })
//                        .permitAll())
//
                .csrf(csrf->csrf.disable())
            ;


    }
/*配置自定义过滤器设置*/
private RestAuthenticationFilter restAuthenticationFilter () throws Exception {
        RestAuthenticationFilter filter = new RestAuthenticationFilter(objectMapper);
    	//设置认证成功的处理方法
        filter.setAuthenticationSuccessHandler(getAuthenticationSuccessHandler());
        filter.setAuthenticationFailureHandler(getAuthenticationFailureHandler());
        //设置AuthenticationManager 分类的方法
        filter.setAuthenticationManager(authenticationManager());
    	//设置请求路径
        filter.setFilterProcessesUrl("/authorize/login");
        return filter;
    }
```

思路：先去看spring默认的过滤器怎么写的，然后模仿他。 

## 3

### 3-1 密码进化史

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742195633-21d1853a-9ce1-48e2-a777-0989efba68bf.png)

哈希可用彩虹表 比对 得出原文

加盐：每次加密会随机生成一个盐值，存放在系统中，使用盐值+原文 哈希加密。 

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742195932-5c33af20-dd0c-4611-b7af-3d10de9502e4.png)

加大了密码复杂度后，响应时间会加长。

### 3-2 密码编码器

常用的有 BCryptPasswordEncoder 

![img](https://cdn.nlark.com/yuque/0/2023/png/22468540/1677742196825-d28399d1-4348-4ecf-ba69-d8d700a528fe.png)

使用多编码格式时，在存储密文时会在前面加上他的加密方式标识，设置时的key。{bcrypt}***

![img](https://cdn.nlark.com/yuque/0/2023/png/22468540/1677742196569-e72e680c-163e-4b9a-803b-5b59e6b888c7.png)

密码升级的思路： 用户登录时拿到 明文密码，认证通过后 使用新的加密方式得到密文 替换 原有密文

### 3-3 验证注解和自定义验证注解 JSR-380验证框架

implementation('org.springframework.boot:spring-boot-starter-validation')

![img](https://cdn.nlark.com/yuque/0/2023/png/22468540/1677742196798-b28dbb37-db60-44bb-abe4-00dba1d4fef0.png)

![img](https://cdn.nlark.com/yuque/0/2023/png/22468540/1677742197140-f1a900e4-b877-4e6f-8c13-ebb8a9b954e3.png)

implementation('org.springframework.boot:spring-boot-starter-validation')

### 3-4密码的验证规则和自定义注解和验证器

![img](https://cdn.nlark.com/yuque/0/2023/png/22468540/1677742197622-c53a2513-eb98-4302-8858-400e487fdd30.png)

##### 密码校验

创建一个校验注解

```plain
package com.li.springsecurity.annotation;

import com.li.springsecurity.validation.PasswordConstraintValidator;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import javax.validation.Constraint;
import javax.validation.Payload;

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = {PasswordConstraintValidator.class})
@Documented
public @interface ValidPassword {

    String message() default "Invalid password";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};


}
```

创建一个类实现密码验证ConstraintValidator接口 ：

1. 接口使用了泛型，需要指定两个参数，第一个自定义注解类，第二个为需要校验的数据类型。
2. 实现接口后要override两个方法，分别为initialize方法和isValid方法。其中initialize为初始化方法，可以在里面做一些初始化操作，isValid方法就是最终需要的校验方法。可以在该方法中实现具体的校验步骤。 本方法使用了passay 提供的校验方式。	

```plain
public class PasswordConstraintValidator implements ConstraintValidator<ValidPassword,String> {
    @Override
    public boolean isValid(String password, ConstraintValidatorContext constraintValidatorContext) {

        val validator = new PasswordValidator(Arrays.asList(
                new LengthRule(8, 30),
                //至少一个大写字母
                new CharacterRule(EnglishCharacterData.UpperCase, 1),
                //至少一个小写字母
                new CharacterRule(EnglishCharacterData.LowerCase, 1),
                //至少一个数字字符
                new CharacterRule(EnglishCharacterData.Special, 1),
                //至少一个特殊字符
                new CharacterRule(EnglishCharacterData.Digit, 1),
                new WhitespaceRule()
                ));
        //验证密码
        val result = validator.validate(new PasswordData(password));

        return result.isValid();
    }

    @Override
    public void initialize(ValidPassword constraintAnnotation) {
        ConstraintValidator.super.initialize(constraintAnnotation);
    }
}
```

​	使用时加上该注解即可

```plain
@NotNull
    @ValidPassword
    private String password;
```

##### 多字段联合校验

同样先创建一个注解 作用于类上

```plain
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.PARAMETER, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = {PasswordMatchValidator.class})
@Documented
public @interface PasswordMatch {

        String message() default "password not matched";

        Class<?>[] groups() default {};

        Class<? extends Payload>[] payload() default {};
}
```

创建一个验证类

​		这里需要实现的泛型写需要校验的类 

​		在isValid中进行多字段校验

```plain
public class PasswordMatchValidator  implements ConstraintValidator<PasswordMatch, UserDTO> {
    @Override
    public void initialize(PasswordMatch constraintAnnotation) {
        ConstraintValidator.super.initialize(constraintAnnotation);
    }

    @Override
    public boolean isValid(UserDTO userDTO, ConstraintValidatorContext constraintValidatorContext) {
        //两次密码是否一致
        val result = userDTO.getPassword().equals(userDTO.getMatchingPassword());
        return result;
    }
```

### 3-5 passay 异常消息的国际化

​	在之前的异常返回信息中都是英文 用passay的中英文字典 处理

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742197983-00f1da81-fa13-4760-9f18-bfd15f1fa5ae.png)

1. 在WebMvcConfig中配置messageResolver![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742198219-d0864737-e41e-488a-a480-e3d030d26dcd.png)
2. 在passay PasswordValidator() 构造器中引入 springMessageResolver消息解析器。
3. 将原有错误信息禁用，使用新的错误信息![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742198713-21d8c001-b3a9-4ca8-9031-13e90e0361c7.png)
4. 配置 添加中文和英文的国际化消息内容 -https://blog.csdn.net/weixin_38657051/article/details/115221338

1. 1. valid 消息国际化 在mvcconfig中再加一个 bean![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742198656-5ed2fa35-1465-41c0-a12b-daf8d8ee52ed.png)

​			



### 3-6 异常统一处理

主要用到了 problem 的功能 

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677742437422-05f36c6a-a3ea-433e-b15c-ca9fd542a58f.png)



1. 引入 problem 依赖

```xml
<dependency>
  <groupId>org.zalando</groupId>
  <artifactId>problem-spring-web</artifactId>
  <version>0.26.1</version>
</dependency>
```

1. 创建ExcepptionHandler

```java
@ControllerAdvice
    public class ExceptionHandler implements ProblemHandling {
    /**
     * 是否将堆栈中的错误信息返回
     */
        @Override
        public boolean isCausalChainsEnabled() {

            return true;
        }
    }
```

1. 创建SecurityExceptionHandler

```java
public class SecurityExceptionHandler implements SecurityAdviceTrait{

}
```

1. 配置异常替换

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677748479347-df0666f2-3cf8-4122-b3c2-ca411e8698a0.png)

https://blog.csdn.net/weixin_38657051/article/details/115221684

### 3-7 多种安全配置共存 

 既能实现form表单登录，同时也支持api的rest请求的登录。

1. 再写一个针对传统页面登录的配置

```java
@Configuration
@Slf4j
@Order(100)
public class LoginSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http

                .authorizeRequests(req -> req.anyRequest().authenticated())//授权请求控制
                .formLogin(form -> form.loginPage("/login")
                        .usernameParameter("username1")
                        .defaultSuccessUrl("/index")
                        .permitAll())
                .logout(logout -> logout.logoutUrl("/perform_logout"))
                .rememberMe(rememberMe-> rememberMe.tokenValiditySeconds(60*60));
    }

    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring().mvcMatchers("/webjars/**", "/public/**")
        //常用静态资源地址
                .requestMatchers(PathRequest.toStaticResources().atCommonLocations());
    }
}
```

写法和普通配置一致，然后在其中配置针对表单登录显示的处理，多配置类共存的情况下，需要设置@Order注解，用来顺序加载Bean。否则会冲突。

## 4 

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677754218900-5e5511f7-75a5-4227-b8a8-f70c5dac7c78.png)

### 4-1 核心组件 - SecurityContext SecurityContextHolder Authentication

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677754241989-c5dbab87-cba9-4246-b50f-6dd994630c69.png)



![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677916715450-31b9b39b-9ad6-47f9-ae83-a1b28e7f59ce.png)

securityContext 存储的 为 Authentication  接口

Principal ： 个人信息 Object 可以放任何信息 不一定是一个人 可以是程序

credentials ： 可以存储各种密码信息 如 人脸识别 数据信息

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677916878150-9dea5b6e-9e1e-424c-8e23-68093699e265.png)

包含各种token 

实践一下

写一个获取用户权限信息的接口

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677917508100-2df39164-0f07-48f5-94bd-b559e4b9ef33.png)

由于get不能携带body，开启httpBasic 使其能携带认证头

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677917584876-cb7cb84f-b73f-4b88-8d73-ac7f26201e8c.png)

测试一下

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677917623865-1b53a32d-992a-4c4e-bcad-bdf5c6263990.png)

在Spring中 被Spring管理的 可以直接注入用户信息

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677917692392-65f74dbc-3911-4881-b0b5-b9b98a9721b2.png)

同样拿到了用户信息

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677917733592-e339a4b0-cbc4-4587-832b-35d0d75da417.png)

## 

### 4-2 UserDetails、UserDetailsService和jdbcAuthentication

UserDetails 是通常意义上的用户

UserDetailsService 用于调取用户信息的服务 把用户信息提取出来 形成UserDetails

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677917930247-377a8eef-a8c6-4e12-bc26-3830dc8deb4d.png)

UserDetails 是一个接口 高可拓展性 想要定制化 可以实现这些接口

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677918172999-2fb2659e-da69-4782-8a45-6ac44300ed80.png)

真正的认证服务在 AuthenticationManager 中配置

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677918116548-fa3ba7b0-837d-4f8e-9c39-4fa090a597d9.png)

只有一个方法，加载用户信息



添加数据库依赖 配置

使用h2内存数据库 可以设置h2 模式mysql

```plain
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>

        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>


  datasource:
    driver-class-name: org.h2.Driver
    password: ""
    url: jdbc:h2:mem:test;MODE=MySQL;DATABASE_TO_LOWER=TRUE
    username: sa
  h2:
    console:
      enabled: true
      path: /h2-console
      settings:
        trace: false
        web-allow-others: false
```



添加内建数据库支持![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677921188864-8b86a934-9e58-421b-8dd0-cce32f619678.png)![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677921221948-9a4e0492-4ebd-4f3a-9633-055f4db93633.png)

配置好后生成了设置的两个用户

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677921320452-84f5d70e-a58a-4bc1-a78a-d2b220fac50e.png)

### 4-3 定制化数据库

设置表查询语句

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677922628420-e24bc806-5cb8-48ae-845c-bc714868fe4b.png)

### 4-4 深度定制化上 - 实现 UserDetails 和 GrantedAuthority

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1677923001070-9ed3854e-f24a-4b3f-9ee9-fbd240938514.png)

创建user类 和 role 类 实现UserDetails 并实现其方法

```plain
@Entity
@Table(name = "users")
public class User implements UserDetails, Serializable {



    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(length = 50,unique = true,nullable = false)
    private String username;

    @Column(name ="password_hash",length = 80,nullable = false)
    private String password;

    @Column(length = 255,unique = true,nullable = false)
    private String email;

    @Column(length = 50)
    private String realName;

    @Column(name = "enabled",nullable = false)
    private boolean enabled;

    @Column(name = "account_non_expired",nullable = false)
    private boolean accountNonExpired;

    @Column(name = "account_non_locked",nullable = false)
    private boolean accountNonLocked;

    @Column(name = "credentials_non_expired",nullable = false)
    private boolean credentialsNonExpired;


    @ManyToMany
    @Fetch(FetchMode.JOIN)
    @JoinTable(name = "user_role"
            ,joinColumns ={ @JoinColumn(name = "user_id",referencedColumnName = "id")}
            ,inverseJoinColumns ={ @JoinColumn(name = "role_id",referencedColumnName = "id")})
    private Set<Role> authorities;
```

Role 实现 GrantedAuthority 接口

```plain
@Data
@Entity
@Table(name = "roles")
public class Role implements GrantedAuthority , Serializable {


    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "role_name",length = 50,unique = true,nullable = false)
    private String authority;

    @ManyToMany(mappedBy = "authorities")
    private Collection<User> users;

    public Collection<User> getUsers() {
        return users;
    }

    public void setUsers(Collection<User> users) {
        this.users = users;
    }
}
```



在security config 中 配置自己的用户、权限 查找sql

```plain
@Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.jdbcAuthentication()
                //设置默认的schema
                .withDefaultSchema()
                //设置数据源
                .dataSource(dataSource)
                .usersByUsernameQuery("select username,password,enabled from users where username=?")
                .authoritiesByUsernameQuery("select username,authority from authorities where username=?")


                .withUser("user")
                .password("12345678")
                .roles("USER", "ADMIN")
                .and()
                .withUser("old_user")
                .password("123456")
                .roles("USER")
        ;
    }
```

### 4-5  深度定制化下 - UserDetailsService 和 UserDetailsPasswordService

创建 UserDetailsServiceImpl 实现 UserDetaiilsService

```plain
@Service
@RequiredArgsConstructor
public class UserDetailsServiceImpl implements UserDetailsService {

    private final UserRepo userRepo;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return userRepo.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found"));
    }
}
```

创建UserDetailsPasswordServiceImpl 实现 UserDetailsPasswordService 

```plain
@Service
@RequiredArgsConstructor
public class UserDetailsPasswordServiceImpl implements UserDetailsPasswordService {

    private final UserRepo userRepo;

    //用于密码升级
    @Override
    public UserDetails updatePassword(UserDetails user, String newPassword) {
        return userRepo.findByUsername(user.getUsername())
                .map(u ->
                        (UserDetails)userRepo.save(u.withPassword(newPassword))
                )
                .orElse(user);
    }
}
```

dao 层代码

```plain
@Repository
public interface UserRepo extends JpaRepository<User,Long>{



    Optional<User> findByUsername(String username);

}

@Repository
public interface RoleRepo extends JpaRepository<Role, Long> {

}
```

用 Optional作为返回值 能够方便的判空并进行处理

实体类使用@ With 能够修改属性的值 并返回一个新的对象https://www.jianshu.com/p/6660142f70c7



### 4-6 环境和环境变量

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678089703311-510a41cd-e71c-421a-b672-daa620ff52ce.png)

环境变量可以把敏感信息 隔离出来，不写在yml中 开发与运维隔离

 不同环境可以设置不同的yml 并用  

```plain
spring:
	profiles:
  	active: dev
```

标注这个文件是哪个环境 

在application.yml 或者环境 中指定使用哪个配置

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678090833627-0446ffc0-b925-425b-9a82-c8839de8952d.png)



### 4-7 自动化测试

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678091807797-872b9d6e-f8f5-4bde-aacd-d65f3254c012.png)



```java
@SpringBootTest
    public class SecurityRestAPIIntTests {

        @Autowired
        private WebApplicationContext context;

        private MockMvc mockMvc;


        @BeforeEach
        public void setup() {
            mockMvc = MockMvcBuilders.webAppContextSetup(context)
                //应用安全配置
                .apply(springSecurity())
                .build();
        }

        //提供一个虚拟用户
        @WithMockUser(username = "user", password = "password", roles = "USER")
        @Test
        public void givenNoToken_whenGetSecureRequest_thenUnauthorized() throws Exception {
            mockMvc.perform(get("/api/gr"))
                .andExpect(status().isOk());
        }
    }
```



## 5 深入了解springSecurity 认证过程



![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678092005558-d24ea9f8-5f4d-4c15-87b9-1393f1d8ec28.png)

### 5-1 认证流程和源码解析

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678092039729-ffaf92b5-9f58-46cd-88ef-3c3e2b2bcfa2.png)

AuthenticationManager中可以有多个AuthenticationProvider ，Provider 是认证机制，支持多种方式 如 数据库密码严重 通过http去其他服务认证

只要一个 provider 认证成功 会将用户信息拿到 构建一个UserDetails



https://blog.51cto.com/u_15072920/4180591



密码升级

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678099079352-0d73e2de-f86a-44da-a3a3-b5bd44f3ea1d.png)



### 5-2 LDAP 配置和多 AuthenticationProvider 共存

日常接触场景不多

用户信息不止存储于sql数据库中  security认证流程很复杂 是为了适配多种认证方式

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678099341807-494acbce-9ff7-40c4-9399-f92319afd95d.png)



https://www.cnblogs.com/wilburxu/p/9174353.html 

ldap是树状结构



配置：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-ldap</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.ldap</groupId>
  <artifactId>spring-ldap-core</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.security</groupId>
  <artifactId>spring-security-ldap</artifactId>
</dependency>

<dependency>
  <groupId>com.unboundid</groupId>
  <artifactId>unboundid-ldapsdk</artifactId>
</dependency>
```



```yaml
spring:
	ldap:
		base: dc=imooc,dc=com
		embedded:
			base-dn: dc=imooc,dc=com
			ldif: classpath:test-ldap-server.ldif
			port: 8389
		urls: ldap://localhost:8389/
```



创建LDAPUser模型 实现UserDetails

```java
@Builder
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    @Entry(objectClasses = {"inetOrgPerson", "organizationalPerson", "person", "top"})
    public final class LDAPUser implements UserDetails {
        @Id
        @JsonIgnore
        private Name id;

        @Attribute(name = "uid")
        private String username;

        @Attribute(name = "userPassword")
        private String password;

        @Override
        public Collection<? extends GrantedAuthority> getAuthorities() {
            return Collections.singletonList(new SimpleGrantedAuthority("ROLE_USER"));
        }

        @Override
        public boolean isAccountNonExpired() {
            return true;
        }

        @Override
        public boolean isAccountNonLocked() {
            return true;
        }

        @Override
        public boolean isCredentialsNonExpired() {
            return true;
        }

        @Override
        public boolean isEnabled() {
            return true;
        }
    }
```



ORM层

```plain
package com.imooc.uaa.security.auth.ldap;

import org.springframework.data.ldap.repository.LdapRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface LDAPUserRepo extends LdapRepository<LDAPUser> {
    Optional<LDAPUser> findByUsername(String username);

    Optional<LDAPUser> findByUsernameAndPassword(String username, String password);

    List<LDAPUser> findByUsernameLikeIgnoreCase(String username);
}
```



Provider

```java
package com.imooc.uaa.security.auth.ldap;

import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.authentication.dao.AbstractUserDetailsAuthenticationProvider;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.core.userdetails.UserDetails;

import lombok.RequiredArgsConstructor;

/**

●  展示和 DaoAuthenticationProvider 一起工作的场景 
●  使用 UsernamePasswordAuthenticationToken
*/
@RequiredArgsConstructor
public class LDAPMultiAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {
private final LDAPUserRepo ldapUserRepo;
@Override  
protected void additionalAuthenticationChecks(UserDetails userDetails, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
}
@Override  
protected UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
return ldapUserRepo.findByUsernameAndPassword(username, authentication.getCredentials().toString())
.orElseThrow(() -> new BadCredentialsException("[LDAP] 用户名或密码错误"));
} 

}
```



将LDAP Provider 加入到Security 配置中

```java
@Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 配置 LdapAuthenticationProvider
        auth.authenticationProvider(new LDAPMultiAuthenticationProvider(ldapUserRepo));
        // 配置 DaoAuthenticationProvider
        auth
            .userDetailsService(userDetailsServiceImpl) // 配置 AuthenticationManager 使用 userService
            .passwordEncoder(passwordEncoder()) // 配置 AuthenticationManager 使用 userService
            .userDetailsPasswordManager(userDetailsPasswordServiceImpl); // 配置密码自动升级服务
    }
```



建立LADP 模型 ORM层 再配置一个LDAP的Provider 在Provider中配置认证用户的方式。将其加入到Security中。

ProviderManager会扫描到这个Provider 只要所有Provider中有一个认证通过 即认证通过。

相似的，可以用此方法配置其他数据源的认证方式。



### 5-3 JWT 的概念和创建以及解析



![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678264657607-8d192f1f-9a11-4d75-9289-d3ea30eaf683.png)

自包含：自己可以验证自己，不需要再去数据库查询 

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678264861186-35b48d04-df14-4d68-9c8a-542904112b91.png)

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678265023713-764f54f6-32bf-45b2-81fe-b16303937ae6.png)

jwt 的承载数据部分是公开的，不能放隐私信息

https://jwt.io/

```xml
<jjwt.version>0.11.1</jjwt.version>
<!-- JWT 依赖开始 -->
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt-api</artifactId>
  <version>${jjwt.version}</version>
</dependency>
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt-impl</artifactId>
  <version>${jjwt.version}</version>
  <scope>runtime</scope>
</dependency>
<dependency>
  <groupId>io.jsonwebtoken</groupId>
  <artifactId>jjwt-jackson</artifactId>
  <version>${jjwt.version}</version>
  <scope>runtime</scope>
</dependency>
```

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678265215954-cba9b251-fb6e-4952-a9aa-0ada8745bfe7.png)



### 5-4 访问令牌和刷新令牌

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678937160996-22bf27c5-1bd9-4967-9aaf-0861c7d7a0b5.png)

对于公开的令牌 缩短其有效期   还可以可以综合其方式 如高频访问的ip等手段来防止盗取

刷新令牌 是不能用于访问的，只有一个唯一的用途，获得访问令牌。 

前端存放的话 cookie最安全 直接由服务端设置 服务端设置时还可以把域名设置上，只允许这个域名访问。



![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678937782196-5a5ee984-e852-48a5-aed0-bcbea11f75c7.png)





#### 从yml中配置参数

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678938069310-3a7299b2-2beb-48d0-b4f0-da229b7234db.png)

创建一个Properties类 

prefix对应 yml

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678938123731-d206a177-224a-4285-a2c5-b1343f079f7e.png)

```java
public String createAccessToken(UserDetails userDetails) {
    return createJWTToken(userDetails, appProperties.getJwt().getAccessTokenExpireTime());
}

public String createRefreshToken(UserDetails userDetails) {
    return createJWTToken(userDetails, appProperties.getJwt().getRefreshTokenExpireTime(), refreshKey);
}

public String createJWTToken(UserDetails userDetails, long timeToExpire) {
    return createJWTToken(userDetails, timeToExpire, key);
}

/**
* 根据用户信息生成一个 JWT
*
* @param userDetails  用户信息
* @param timeToExpire 毫秒单位的失效时间
* @param signKey      签名使用的 key
* @return JWT
*/
public String createJWTToken(UserDetails userDetails, long timeToExpire, Key signKey) {
    return Jwts
        .builder()
        .setId("imooc")
        .setSubject(userDetails.getUsername())
        .claim("authorities",
               userDetails.getAuthorities().stream()
               .map(GrantedAuthority::getAuthority)
               .collect(Collectors.toList()))
        .setIssuedAt(new Date(System.currentTimeMillis()))
        .setExpiration(new Date(System.currentTimeMillis() + timeToExpire))
        .signWith(signKey, SignatureAlgorithm.HS512).compact();
}
```

### 5-5 创建JwtFilter

编写filter

```java
@Slf4j
@RequiredArgsConstructor
@Component
public class JwtFilter extends OncePerRequestFilter {

    private final AppProperties appProperties;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        //1. 检查 JWT Token 是否在 HTTP 报头中
        if(checkJwtToken(request)) {
            //2. 有的话，解析 JWT Token
            validateToken(request)
                    .filter(claims -> claims.get("authorities") != null)
                    .ifPresentOrElse(
                            this::setupSpringAuthentication, // 有值
                            SecurityContextHolder::clearContext //为空
                    );
        }
        //3. 无论如何都要继续过滤器链
        filterChain.doFilter(request, response);
    }

    private void setupSpringAuthentication(Claims claims) {
        val rawList = CollectionUtil.convertObjectToList(claims.get("authorities"));
        val authorities = rawList.stream()
                .map(String::valueOf)
                .map(SimpleGrantedAuthority::new)
                .collect(toList());
        val authentication = new UsernamePasswordAuthenticationToken(claims.getSubject(), null, authorities);
        //将对象设置进 Security 中
        SecurityContextHolder.getContext().setAuthentication(authentication);
    }

    private Optional<Claims> validateToken(HttpServletRequest req) {
        // 从 HTTP 报头中取出 JWT Token
        String jwtToken = req.getHeader(appProperties.getJwt().getHeader()).replace(appProperties.getJwt().getPrefix(), "");
        try {
            return Optional.of(Jwts.parserBuilder().setSigningKey(JwtUtil.key).build().parseClaimsJws(jwtToken).getBody());
        } catch (ExpiredJwtException | SignatureException | MalformedJwtException | UnsupportedJwtException | IllegalArgumentException e) {
            return Optional.empty();
        }
    }

    /**
     * 检查 JWT Token 是否在 HTTP 报头中
     *
     * @param req HTTP 请求
     * @return 是否有 JWT Token
     */
    private boolean checkJwtToken(HttpServletRequest req) {
        String authenticationHeader = req.getHeader(appProperties.getJwt().getHeader());
        return authenticationHeader != null && authenticationHeader.startsWith(appProperties.getJwt().getPrefix());
    }
}
```

在config中配置jwtFliter 将其加入到UsernamePasswordAu.. 前面

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678948485740-4274236e-015f-429a-9535-76e662fc867d.png)



### 5-6 实现登录接口和刷新令牌接口



重新生成token 利用了 token自包含特性

```java
@PostMapping("/token")
    public Auth login(@Valid @RequestBody LoginDto loginDTO) {
        return userService.login(loginDTO.getUsername(), loginDTO.getPassword());
    }

 /**
     * 用户登录
     *
     * @param username 用户名
     * @param password 密码
     * @return JWT
     */
    public Auth login(String username, String password) {
        return userRepo.findOptionalByUsername(username)
            .filter(user -> passwordEncoder.matches(password, user.getPassword()))
            .map(user -> new Auth(jwtUtil.createAccessToken(user), jwtUtil.createRefreshToken(user)))
            .orElseThrow(() -> new AccessDeniedException("用户名密码错误"));
    }


    @PostMapping("/token/refresh")
    public Auth refreshToken(@RequestHeader(name = "Authorization") String authorization, @RequestParam String refreshToken) {
        val PREFIX = "Bearer ";
        val accessToken = authorization.replace(PREFIX, "");
        // 1. 验证refreshToken是否有效
        // 2. 验证accessToken是否有效
        if (jwtUtil.validateRefreshToken(refreshToken) && jwtUtil.validateWithoutExpiration(accessToken)) {
            return new Auth(jwtUtil.buildAccessTokenWithRefreshToken(refreshToken), refreshToken);
        }
        throw new AccessDeniedException("Bad Credentials");
    }

    public boolean validateRefreshToken(String jwtToken) {
        return validateToken(jwtToken, refreshKey);
    }

public boolean validateWithoutExpiration(String jwtToken) {
        try {
            Jwts.parserBuilder().setSigningKey(JwtUtil.key).build().parseClaimsJws(jwtToken);
            return true;
        } catch (ExpiredJwtException | SignatureException | MalformedJwtException | UnsupportedJwtException | IllegalArgumentException e) {
            if (e instanceof ExpiredJwtException) {
                return true;
            }
        }
        return false;
    }
```

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678949472003-0ac40935-3d50-4bb3-8cb4-646915027c78.png)



### 5-8 完成注册接口

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678957006390-46758a7e-8c30-4c9d-a450-c8e3748ef85e.png)

```java
@PostMapping("/register")
    public void register(@Valid @RequestBody UserDto userDto, Locale locale) {
        // 1. 验证用户名是否存在
        if (userService.isUsernameExisted(userDto.getUsername())) {
            throw new DuplicateProblem("Exception.duplicate.username", messageSource, locale);
        }
        // 2. 验证邮箱是否存在
        if (userService.isEmailExisted(userDto.getEmail())) {
            throw new DuplicateProblem("Exception.duplicate.email", messageSource, locale);
        }
        // 3. 验证手机号是否存在
        if (userService.isMobileExisted(userDto.getMobile())) {
            throw new DuplicateProblem("Exception.duplicate.mobile", messageSource, locale);
        }
        val user = User.builder()
            .username(userDto.getUsername())
            .name(userDto.getName())
            .email(userDto.getEmail())
            .mobile(userDto.getMobile())
            .password(userDto.getPassword())
            .build();
        userService.register(user);
    }

	@Transactional
    public User register(User user) {
        return roleRepo.findOptionalByAuthority(ROLE_USER)
            .map(role -> {
                val userToSave = user
                    .withAuthorities(Set.of(role))
                    .withPassword(passwordEncoder.encode(user.getPassword()));
                return userRepo.save(userToSave);
            })
            .orElseThrow();
    }
```



##  6 

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678957818825-3ac7b223-1efe-4cee-ad26-7da34440848d.png)

### 6-1 多因子认证和TOTP

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678957853996-150c3013-ad2a-4745-951f-72922be7c4eb.png)

可以用多种因素来验证登录 如 指纹、位置。。



![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678958040489-da508328-5656-44e8-a207-8474d556f04d.png)

 ![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1678958212580-2c1aa523-f9cf-4a43-8e1a-9cfb1ebfe530.png)

redis 主要做缓存



本质就是生成一个在一定时间内不会变的密码，在有效期内输入验证正确即可通过，超过有效期 不能通过

```java
/**
 * 用于一次性验证码
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class TotpUtil {
    //密码有效期，单位秒
    private static final long TIME_STEP = 60 * 5L;
    //密码长度
    private static final int PASSWORD_LENGTH = 6;
    private KeyGenerator keyGenerator;
    //key 存储在用户中
    private TimeBasedOneTimePasswordGenerator totp;

    /*
     * 初始化代码块，Java 8 开始支持。这种初始化代码块的执行在构造函数之前
     * 准确说应该是 Java 编译器会把代码块拷贝到构造函数的最开始。
     */
    {
        try {
            totp = new TimeBasedOneTimePasswordGenerator(Duration.ofSeconds(TIME_STEP), PASSWORD_LENGTH);
            // 生成一个 key
            keyGenerator = KeyGenerator.getInstance(totp.getAlgorithm());
            // SHA-1 and SHA-256 需要 64 字节 (512 位) 的 key; SHA512 需要 128 字节 (1024 位) 的 key
            keyGenerator.init(512);
        } catch (NoSuchAlgorithmException e) {
            log.error("没有找到算法 {}", e.getLocalizedMessage());
        }
    }

    /**
     * @param time 用于生成 TOTP 的时间
     * @return 一次性验证码
     * @throws InvalidKeyException 非法 Key 抛出异常
     */
    public String createTotp(final Key key, final Instant time) throws InvalidKeyException {
        // 格式化字符串，前面补 0
        val format = "%0" + PASSWORD_LENGTH + "d";
        return String.format(format, totp.generateOneTimePassword(key, time));
    }

    public Optional<String> createTotp(final String strKey) {
        try {
            return Optional.of(createTotp(decodeKeyFromString(strKey), Instant.now()));
        } catch (InvalidKeyException e) {
            return Optional.empty();
        }
    }

    /**
     * 验证 TOTP
     *
     * @param code 要验证的 TOTP
     * @return 是否一致
     * @throws InvalidKeyException 非法 Key 抛出异常
     */
    public boolean validateTotp(final Key key, final String code) throws InvalidKeyException {
        val now = Instant.now();
        //再创建一个密码，验证是否一致 超过有效期，验证失败
        return createTotp(key, now).equals(code);
    }

    public Key generateKey() {
        return keyGenerator.generateKey();
    }

    public String encodeKeyToString(Key key) {
        return Base64.getEncoder().encodeToString(key.getEncoded());
    }

    public String encodeKeyToString() {
        return encodeKeyToString(generateKey());
    }

    public Key decodeKeyFromString(String strKey) {
        return new SecretKeySpec(Base64.getDecoder().decode(strKey), totp.getAlgorithm());
    }

    public long getTimeStepInLong() {
        return TIME_STEP;
    }

    public Duration getTimeStep() {
        return totp.getTimeStep();
    }
}
```

### 



### 6-2-3 实战发送TOTP

####   阿里云短信

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679541849224-ff70f740-8b55-453f-935e-aad8b49dbe3d.png)

key Id  Secret ：认证对



签名需要审核

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679541906023-4734bcf5-3cfb-436f-866f-82efbab29773.png)





#### LeanCloud

 和阿里云类似





#### 电子邮件

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679542015831-0f116dc1-5365-404e-8135-d8d433814640.png)

主要是两种 api smtp



因为有多种发消息的方式，可以先定义一个接口 来限制的方法

```java
public interface SmsService {
    void send(String mobile, String msg);
}
```

@ConditionalOnProperty 这个配置指 可以在yaml中配置使用 当yml中 mooc.sms-provider.name = ali 时会启用这个实现类 没有则不生效

```java
@Slf4j
    @RequiredArgsConstructor
    //这个配置指 可以在yaml中配置使用 当yml中 mooc.sms-provider.name = ali 时会启用这个实现类
    @ConditionalOnProperty(prefix = "mooc.sms-provider", name = "name", havingValue = "ali")
    @Service
    public class SmsServiceAliSmsImpl implements SmsService {

        private final IAcsClient client;
        private final AppProperties appProperties;

        @Override
        public void send(String mobile, String msg) {
            //就是构建一个请求 只不过帮我们屏蔽了很多认证的方法 
            val request = new CommonRequest();
            request.setSysMethod(MethodType.POST);
            request.setSysDomain(appProperties.getSmsProvider().getApiUrl());
            request.setSysAction("SendSms");
            request.setSysVersion("2017-05-25");
            request.putQueryParameter("RegionId", "cn-hangzhou");
            request.putQueryParameter("PhoneNumbers", mobile);
            request.putQueryParameter("SignName", "登录验证");
            request.putQueryParameter("TemplateCode", "SMS_1610048");
            request.putQueryParameter("TemplateParam", "{\"code\":\"" +
                                      msg +
                                      "\",\"product\":\"慕课网实战Spring Security\"}");
            try {
                val response = client.getCommonResponse(request);
                log.info("短信发送结果 {}", response.getData());
            } catch (ServerException e) {
                log.error("发送短信时产生服务端异常 {}", e.getLocalizedMessage());
            } catch (ClientException e) {
                log.error("发送短信时产生客户端异常 {}", e.getLocalizedMessage());
            }
        }
    }
```





```java
@Validated
    @Configuration
    @ConfigurationProperties(prefix = "mooc")
    public class AppProperties {

        @Getter
        @Setter
        @Valid
        private Jwt jwt = new Jwt();

        @Getter
        @Setter
        @Valid
        private SmsProvider smsProvider = new SmsProvider();

        @Getter
        @Setter
        @Valid
        private EmailProvider emailProvider = new EmailProvider();

        @Getter
        @Setter
        @Valid
        private LeanCloud leanCloud = new LeanCloud();

        @Getter
        @Setter
        @Valid
        private Ali ali = new Ali();

        @Getter
            @Setter
            public static class Jwt {

                private String header = "Authorization"; // HTTP 报头的认证字段的 key

                private String prefix = "Bearer "; // HTTP 报头的认证字段的值的前缀

                @Min(5000L)
                private long accessTokenExpireTime = 60 * 1000L; // Access Token 过期时间

                @Min(3600000L)
                private long refreshTokenExpireTime = 30 * 24 * 3600 * 1000L; // Refresh Token 过期时间

                private String key;

                private String refreshKey;
            }

        @Getter
            @Setter
            public static class LeanCloud {
                private String appId;
                private String appKey;
            }

        @Getter
            @Setter
            public static class Ali {
                private String apiKey;
                private String apiSecret;
            }

        @Getter
            @Setter
            public static class SmsProvider {
                private String name;
                private String apiUrl;
            }

        @Getter
            @Setter
            public static class EmailProvider {
                private String name;
                private String apiKey;
            }
    }
```



```java
@RequiredArgsConstructor
    @Slf4j
    @Service
    @ConditionalOnProperty(prefix = "mooc.sms-provider", name = "name", havingValue = "lean-cloud")
    public class SmsServiceLeanCloudSmsImpl implements SmsService {

        @Override
        public void send(String mobile, String msg) {
            val option = new AVSMSOption();
            option.setTtl(10);
            option.setApplicationName("慕课网实战Spring Security");
            option.setOperation("两步验证");
            option.setTemplateName("登录验证");
            option.setSignatureName("慕课网");
            option.setType(AVSMS.TYPE.TEXT_SMS);
            option.setEnvMap(Map.of("smsCode", msg));
            AVSMS.requestSMSCodeInBackground(mobile, option)
                .take(1)
                .subscribe(
                    (res) -> log.info("短信发送成功 {}", res),
                    (err) -> log.error("发送短信时产生服务端异常 {}", err.getLocalizedMessage())
                );
        }
    }
```



```java
@RequiredArgsConstructor
    @Configuration
    public class LeanCloudConfig {

        private final AppProperties appProperties;
        private final Environment env;

        @PostConstruct()//初始化前执行
        public void initialize() {
            if (env.acceptsProfiles(Profiles.of("prod"))) {
                AVOSCloud.setLogLevel(AVLogger.Level.ERROR);
            } else {
                AVOSCloud.setLogLevel(AVLogger.Level.DEBUG);
            }
            //初始化
            AVOSCloud.initialize(appProperties.getLeanCloud().getAppId(), appProperties.getLeanCloud().getAppKey());
        }
    }
```







```yaml
# 使用环境变量形式来配置 
ali:
  api-key: ${ALI_API_KEY}
  api-secret: ${ALI_API_SECRET}
```



### 6-4 邮件发送

```java
public interface EmailService {
    void send(String email, String msg);
}
```



```java
@Slf4j
    @ConditionalOnProperty(prefix = "mooc.email-provider", name = "name", havingValue = "api")
    @RequiredArgsConstructor
    @Service
    public class EmailServiceApiImpl implements EmailService {

        private final SendGrid sendGrid;

        @Override
        public void send(String email, String msg) {
            val from = new Email("service@imooc.com");
            val subject = "慕课网实战Spring Security 登录验证码";
            val to = new Email(email);
            val content = new Content("text/plain", "验证码为:" + msg);
            val mail = new Mail(from, subject, to, content);
            val request = new Request();
            try {
                request.setMethod(Method.POST);
                request.setEndpoint("mail/send");
                request.setBody(mail.build());
                Response response = sendGrid.api(request);
                if (response.getStatusCode() == 202) {
                    log.info("邮件发送成功");
                } else {
                    log.error(response.getBody());
                }
            } catch (IOException e) {
                log.error("请求发生异常 {}", e.getLocalizedMessage());
            }
        }
    }

@ConditionalOnProperty(prefix = "mooc.email-provider", name = "name", havingValue = "smtp")
    @RequiredArgsConstructor
    @Service
    public class EmailServiceSmtpImpl implements EmailService {

        private final JavaMailSender emailSender;

        @Override
        public void send(String email, String msg) {
            val message = new SimpleMailMessage();
            message.setTo(email);
            message.setFrom("service@imooc.com");
            message.setSubject("慕课网实战Spring Security 登录验证码");
            message.setText("验证码为:" + msg);
            emailSender.send(message);
        }
    }
```



```java
@RequiredArgsConstructor
    @Configuration
    public class EmailConfig {

        private final AppProperties appProperties;

        //如果有apikey才启动
        @ConditionalOnProperty(prefix = "mooc.email-provider", name = "api-key")
        @Bean
        public SendGrid sendGrid() {
            return new SendGrid(appProperties.getEmailProvider().getApiKey());
        }
    }
```



### 6-5 用户登录逻辑

##### 

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679542072438-2a8bc184-0fb8-49c3-ac4e-f43169e0afa8.png)

可以让用户选择是否需要二次认证，也可以管理员设置  当useringMfa 为true时 启用  



 

####  1.表加字段

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679553013669-96170b0b-9d47-4f76-b25b-2cc321412bfe.png)

#### 2. userService

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679553562354-61606e29-c0c5-460c-bd08-ab99662a63df.png)

创建用户时 创建一个mfa key 存入user表中

#### 3.  登录接口 返回mfa 标识和 用户信息key

```java
@PostMapping("/token")
    public ResponseEntity<?> login(@Valid @RequestBody LoginDto loginDTO) {
        return userService.findOptionalByUsernameAndPassword(loginDTO.getUsername(), loginDTO.getPassword())
            .map(user -> {
                userService.upgradePasswordEncodingIfNeeded(user, loginDTO.getPassword());
                if (!user.isEnabled()) {
                    throw new UserNotEnabledProblem();
                }
                if (!user.isAccountNonLocked()) {
                    throw new UserAccountLockedProblem();
                }
                if (!user.isAccountNonExpired()) {
                    throw new UserAccountExpiredProblem();
                }
                // 不使用多因子认证
                if (!user.isUsingMfa()) {
                    return ResponseEntity.ok().body(userService.login(user));
                }
                // 使用多因子认证
                //将用户信息缓存起来
                val mfaId = userCacheService.cacheUser(user);
                return ResponseEntity
                    .status(HttpStatus.UNAUTHORIZED)
                    //在请求头中加上 mfa标识 realm id 用于 二次认证时找到这个用户
                    .header("X-Authenticate", "mfa", "realm=" + mfaId)
                    .build();
            })
            .orElseThrow(BadCredentialProblem::new);
    }

//校验用户
 public Optional<User> findOptionalByUsernameAndPassword(String username, String password) {
        return findOptionalByUsername(username)
            .filter(user -> passwordEncoder.matches(password, user.getPassword()));
    }

//密码升级 
 public void upgradePasswordEncodingIfNeeded(User user, String rawPassword) {
        if (passwordEncoder.upgradeEncoding(user.getPassword())) {
            userRepo.save(user.withPassword(passwordEncoder.encode(rawPassword)));
        }
    }
	// 用户缓存
 private final TotpUtil totpUtil;

    public String cacheUser(User user) {
        val mfaId = cryptoUtil.randomAlphanumeric(12);
        log.debug("生成 mfaId: {}", mfaId);
        RMapCache<String, User> cache = redisson.getMapCache(Constants.CACHE_MFA);
        if (!cache.containsKey(mfaId)) {
            cache.put(mfaId, user, totpUtil.getTimeStepInLong(), TimeUnit.SECONDS);
        }
        return mfaId;
    }
```



#### 4. 选择方式 发送校验码

```java
//生成一个totp
@PutMapping("/totp")
    public void sendTotp(@Valid @RequestBody SendTotpDto sendTotpDto) {
        userCacheService.retrieveUser(sendTotpDto.getMfaId())
            //把流 拍扁 返回opt..<>
            //pair 一次返回两个字 减少包装对象
            .flatMap(user -> userService.createTotp(user).map(code -> Pair.of(user, code)))
            .ifPresentOrElse(pair -> {
                log.debug("totp: {}", pair.getSecond());
                if (sendTotpDto.getMfaType() == MfaType.SMS) {
                    smsService.send(pair.getFirst().getMobile(), pair.getSecond());
                } else {
                    emailService.send(pair.getFirst().getEmail(), pair.getSecond());
                }
            }, () -> {
                throw new InvalidTotpProblem();
            });
    }
//验证totp
@PostMapping("/totp")
    public Auth verifyTotp(@Valid @RequestBody TotpVerificationDto totpVerificationDto) {
        return userCacheService.verifyTotp(totpVerificationDto.getMfaId(), totpVerificationDto.getCode())
            .map(User::getUsername)
            .flatMap(userService::findOptionalByUsername)
            .map(userService::loginWithTotp)
            .orElseThrow(InvalidTotpProblem::new);
    }

    public Optional<User> verifyTotp(String mfaId, String code) {
        log.debug("输入参数 mfaId: {}, code: {}", mfaId, code);
        RMapCache<String, User> cache = redisson.getMapCache(Constants.CACHE_MFA);
        if (!cache.containsKey(mfaId) || cache.get(mfaId) == null) {
            return Optional.empty();
        }
        val cachedUser = cache.get(mfaId);
        log.debug("找到用户 {}", cachedUser);
        try {
            val isValid = totpUtil.validateTotp(totpUtil.decodeKeyFromString(cachedUser.getMfaKey()), code);
            log.debug("code {} 的验证结果为 {}", code, isValid);
            if (!isValid) {
                return Optional.empty();
            }
            cache.remove(mfaId);
            log.debug("移除 mfaId: {}", mfaId);
            return Optional.of(cachedUser);
        } catch (InvalidKeyException e) {
            log.error("Key is invalid {}", e.getLocalizedMessage());
        }
        return Optional.empty();
    }
```



### 6-8 前端集成



```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <artifactId>uaa-ui</artifactId>

  <parent>
    <groupId>com.imooc</groupId>
    <artifactId>mono-uaa</artifactId>
    <version>0.0.1-SNAPSHOT</version>
  </parent>

  <build>
    <plugins>
      <plugin>
        <groupId>com.github.eirslett</groupId>
        <artifactId>frontend-maven-plugin</artifactId>
        <version>${frontend-maven-plugin.version}</version>
        <executions>
          <!-- 安装 node.js和npm -->
          <execution>
            <id>install node and npm</id>
            <goals>
              <goal>install-node-and-npm</goal>
            </goals>
            <configuration>
              <nodeVersion>v12.12.0</nodeVersion>
            </configuration>
          </execution>
          <!-- 安装项目依赖 -->
          <execution>
            <id>npm install</id>
            <goals>
              <goal>npm</goal>
            </goals>
            <!-- 可选步骤，因为默认的阶段就是"generate-resources" -->
            <phase>generate-resources</phase>
            <configuration>
              <arguments>install</arguments>
            </configuration>
          </execution>
          <!-- 编译构建前端文件 -->
          <execution>
            <id>npm run build</id>
            <goals>
              <goal>npm</goal>
            </goals>
            <configuration>
              <arguments>run build</arguments>
            </configuration>
          </execution>
          <!-- 运行单元测试 -->
          <!-- <execution>
          <id>npm run test:unit</id>
          <goals>
          <goal>npm</goal>
        </goals>
          <phase>test</phase>
          <configuration>
          <arguments>run test:unit</arguments>
        </configuration>
        </execution>
          -->
          <!-- 运行集成测试 -->
          <!--
          <execution>
          <id>npm run test:e2e</id>
          <goals>
          <goal>npm</goal>
        </goals>
          <phase>test</phase>
          <configuration>
          <arguments>run test:e2e</arguments>
        </configuration>
        </execution>
          -->
        </executions>
        </plugin>
        </plugins>
        </build>
        </project>
```

### 6-9 跨域处理

两种方法 

1. 避免跨域

1. 1. 前后端放一起。 适用于传统单体项目
   2. 前端处理： 设置代理  进行转发 端口映射 

1. 后端开启支持

1. 1. mvc中配置![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679567835466-59611f87-e330-42de-84e8-9aa48dea3b82.png)
   2. security 中配置![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679567878817-4725fc1d-84aa-4b10-8213-b1f860042fe1.png)	

```java
@Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        // 允许跨域访问的主机
        if (environment.acceptsProfiles(Profiles.of("dev"))) {
            configuration.setAllowedOrigins(Collections.singletonList("http://localhost:4001"));
        } else {
            configuration.setAllowedOrigins(Collections.singletonList("https://uaa.imooc.com"));
        }
        //允许的请求方式
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(Collections.singletonList("*"));
        //开放返回响应请求头 如 用于 totp时返回的请求头
        configuration.addExposedHeader("X-Authenticate");
        // 设置 配置bean
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        //把这个配置应用于那些url上  所有的
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }

//在安全配置中加入
```

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679568291240-fc296223-a67f-47a8-90c8-3701b03fab57.png)



## 7



![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679646629292-8f6ba469-b269-4bae-a668-a248533f8288.png)



![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679646764321-62e9cdbe-ff91-41ee-9f8a-7d7dd10179e1.png)

### 7-1 授权的概念和安全表达式的作用

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679647196090-2189f4d2-0f28-44b6-8f87-aa99234d16c7.png)



![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679647241095-0b0414c7-c7ac-48f1-b55d-140882ba9087.png)



### 7-3 方法级注释

先在类上开启方法级安全配置

执行方法之前授权  @PreAuthorize

执行方法之前过滤  @PreFilter

执行方法之前授权  @PostAuthorize

执行方法之前过滤  @PostFilter



![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679648081616-47fe3969-dc78-4d95-9ebe-76c92daf2e6e.png)



#### web 请求是成功的 url层



![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679648098008-cf1513b5-e7af-4497-a96d-adbd22568d9a.png)

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679648243203-8a86819a-c641-446e-bfec-ec5bda797313.png)

#### 方法后的安全注解： 

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679648356240-05dc064e-4cd5-45b6-a60a-6f5904818031.png)

```java
@PostAuthorize("authentication.name.equals(returnObject.username)")
@GetMapping("/users/by-email/{email}")
public User getUserByEmail(@PathVariable  String email) {
    return userService.findOptionalByEmail(email).orElseThrow();
}
```





###  7-3 RBAC 和 角色分级

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679648891446-37497042-7a62-4a77-9aa1-0b842c1e532f.png)

角色是权限的集合

 

角色分层体系 一个角色可以包含多个子角色

![img](https://biji-1307941976.cos.ap-guangzhou.myqcloud.com/1679649296804-7d320b7b-af38-420b-82e7-9c7d60f5e8f5.png)
