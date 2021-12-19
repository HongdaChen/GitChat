### 前言

对于任何一家公司来说，安全就是底线，是重中之重，所以一般项目都会进行非常严格的认证和授权，保护项目安全，阻挡恶意请求，尽管如此，仍然会时常爆出某公司因数据泄漏造成重大损失的新闻，所以安全验证就变得越来越重要，是每个公司都非常重视的模块。

Java 领域常用的安全框架有 Shiro 和 Spring Security。Shiro 因其轻量级的特性得到了业内公司的广泛使用；相比于
Shiro，Spring Security 是一个相对重量级的安全框架，功能比 Shiro 更加强大，权限控制更加细化，它还有另外一个优势就是 Spring
Security 也是 Spring 技术栈的一个模块，可以和 Spring 应用直接整合使用，同时 Spring Boot
也提供了自动化配置，做到了开箱即用，综上所述，Spring Security 是 Java 领域安全框架的最优选择之一，本节课我们就来学习如何使用
Spring Boot 整合 Spring Security。

1\. 创建 Maven 工程，pom.xml 中添加相关依赖，spring-boot-starter-security 为 Spring Security
相关依赖。

    
    
    <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>2.1.5.RELEASE</version>
    </parent>
    
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
      </dependency>
    
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
      </dependency>
    
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
      </dependency>
    </dependencies>
    

2\. 创建 Handler。

    
    
    @Controller
    public class SecurityHandler {
    
        @GetMapping("/index")
        public String index(){
            return "index";
        }
    }
    

3\. 创建 HTML。

    
    
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Title</title>
    </head>
    <body>
        <p>index</p>
        <form method="post" action="/logout">
            <input type="submit" value="退出"/>
        </form>
    </body>
    </html>
    

4\. 创建 application.yml。

    
    
    spring:
      thymeleaf:
        prefix: classpath:/templates/     #模版路径
        suffix: .html                     #模版后缀
        servlet:
          content-type: text/html         #设置Content-type
        encoding: UTF-8                   #编码方式
        mode: HTML5                       #校验 HTML5 格式
    

5\. 创建启动类 Application。

    
    
    package com.southwind;
    
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    
    @SpringBootApplication
    public class Application {
        public static void main(String[] args) {
            SpringApplication.run(Application.class,args);
        }
    }
    

6\. 启动 Application，打开浏览器直接访问 http://localhost:8080/index，会自动跳转到如下页面。

![](https://images.gitbook.cn/9adfd920-c760-11e9-99c1-c37abd23c4b1)

这是因为 Spring Security 给所有的请求都添加了自动验证规则，必须是登录用户才可以访问，否则直接跳转到 Spring Security
默认的登录页面，Spring Security 默认的用户名是 user，那么密码是多少呢？密码是 Spring Security
随机的一段字符串，并在控制台进行了输出，可以直接看到，如下所示。

![](https://images.gitbook.cn/cd105be0-c760-11e9-a5ba-f1eeeb548c06)

所以我们使用 "user" 和 "2889394e-7968-46fe-9c3f-c81dcfefa63a" 即可完成登录，如下所示。

![](https://images.gitbook.cn/d9ba64d0-c760-11e9-99c1-c37abd23c4b1)

![](https://images.gitbook.cn/e3d5d3a0-c760-11e9-9ae4-c3d609c8bfbd)

在实际的生成环境中，用户名和密码当然是需要我们自己来定义的，设置自定义用户名密码非常简单，只需要在 application.yml
中添加配置即可，如下所示。

    
    
    spring:
      thymeleaf:
        prefix: classpath:/templates/     #模版路径
        suffix: .html                     #模版后缀
        servlet:
          content-type: text/html         #设置Content-type
        encoding: UTF-8                   #编码方式
        mode: HTML5                       #校验HTML5格式
      security:
        user:
          name: admin                     #用户名
          password: 123123                #密码
    

重启 Application，使用 "user" 和 "123123" 即可完成登录，如下所示。

![](https://images.gitbook.cn/f2f26330-c760-11e9-99c1-c37abd23c4b1)

![](https://images.gitbook.cn/ffb2b340-c760-11e9-99c1-c37abd23c4b1)

#### 权限管理

实际开发中，我们一般不会直接将权限和用户绑定，而是将权限赋给角色，在给用户添加角色，比如我们现在定义两个 HTML 资源：index.html 和
admin.html，同时定义两个角色 ADMIN 和 USER，ADMIN 拥有访问 index.html 和 admin.html 的权限，USER
只有访问 index.html 的权限，如何用 Security 来实现这一功能呢？

7\. 创建 SecurityConfig 类，继承 WebSecurityConfigurerAdapter。

    
    
    package com.southwind.config;
    
    import org.springframework.context.annotation.Configuration;
    import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
    import org.springframework.security.config.annotation.web.builders.HttpSecurity;
    import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
    import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
    
    @Configuration
    @EnableWebSecurity
    public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
        @Override
        protected void configure(AuthenticationManagerBuilder auth) throws Exception {
            auth.inMemoryAuthentication().passwordEncoder(new MyPasswordEncoder())
                    .withUser("user").password(new MyPasswordEncoder()
                    .encode("000")).roles("USER")
                    .and()
                    .withUser("admin").password(new MyPasswordEncoder()
                    .encode("123")).roles("ADMIN","USER");
        }
    
        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http.authorizeRequests()
                    .antMatchers("/admin").hasRole("ADMIN")
                    .antMatchers("/index").access("hasRole('ADMIN') or hasRole('USER')")
                    .anyRequest().authenticated()
                    .and()
                    .formLogin()
                    .loginPage("/login")
                    .permitAll()
                    .and()
                    .logout()
                    .permitAll()
                    .and()
                    .csrf()
                    .disable();
        }
    }
    

重写 configure(AuthenticationManagerBuilder auth) 和 configure(HttpSecurity
http)，在 configure(AuthenticationManagerBuilder auth) 中添加用户和角色，这里需要用到自定义的
MyPasswordEncoder ，因为这里依赖的是 Spring Security 5.X 版本，需要提供一个 PasswordEncorder
的实例，否则会抛出异常，如下所示。

![WX20190611-145151@2x](https://images.gitbook.cn/26f200a0-c761-11e9-99c1-c37abd23c4b1)

MyPasswordEncoder

    
    
    package com.southwind.config;
    
    import org.springframework.security.crypto.password.PasswordEncoder;
    
    public class MyPasswordEncoder implements PasswordEncoder {
        @Override
        public String encode(CharSequence charSequence) {
            return charSequence.toString();
        }
    
        @Override
        public boolean matches(CharSequence charSequence, String s) {
            return s.equals(charSequence.toString());
        }
    }
    

再回到 SecurityConfig 类的 configure(AuthenticationManagerBuilder auth) 的方法中，代码如下。

    
    
    auth.inMemoryAuthentication().passwordEncoder(new MyPasswordEncoder())
                    .withUser("user").password(new MyPasswordEncoder()
                    .encode("000")).roles("USER")
                    .and()
                    .withUser("admin").password(new MyPasswordEncoder()
                    .encode("123")).roles("ADMIN","USER");
    

这段代码表示设置了两个用户：

  * 用户名：user；密码：000；角色：USER
  * 用户名：admin；密码：123；角色：ADMIN、USER

不同的用户通过 and() 方法连接，非常简单，这里不再赘述。用户设置完成之后，接下来就是给不同的角色赋予不同的权限，具体的操作在
configure(HttpSecurity http) 方法中完成设置，代码如下。

    
    
    http.authorizeRequests()
                    .antMatchers("/admin").hasRole("ADMIN")
                    .antMatchers("/index").access("hasRole('ADMIN') or hasRole('USER')")
                    .anyRequest().authenticated()
                    .and()
                    .formLogin()
                    .loginPage("/login")
                    .permitAll()
                    .and()
                    .logout()
                    .permitAll()
                    .and()
                    .csrf()
                    .disable();
    

  * .antMatchers("/admin").hasRole("ADMIN") 表示访问 /admin 需要具备 ADMIN 角色。
  * .antMatchers("/index").access("hasRole('ADMIN') or hasRole('USER')") 表示访问 /index 需要具备 ADMIN 或者 USER 角色。
  * .anyRequest().authenticated() 表示其他请求都必须经过权限验证才能访问。
  * .formLogin() 表示配置登录信息。
  * .loginPage("/login") 表示使用自定义的登录页面。
  * .permitAll() 表示与登录相关的请求都不需要验证。
  * .logout() 表示集成退出功能。
  * .permitAll() 表示与退出相关的请求都不需要验证。
  * .csrf().disable() 表示关闭 csrf。

修改 Handler，添加 /admin 和 /login 的请求。

    
    
    @Controller
    public class SecurityHandler {
    
        @GetMapping("/index")
        public String index(){
            return "index";
        }
    
          @GetMapping("/admin")
        public String admin() {
            return "admin";
        }
    
        @GetMapping("/login")
        public String login() {
            return "login";
        }
    
    }
    

创建 HTML。

login.html：

    
    
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <title>login</title>
    </head>
    <body>
    <p th:if="${param.error}">
        用户名或密码错
    </p>
    <form th:action="@{/login}" method="post">
        <table>
            <tr>
                <td>用户名：</td>
                <td>
                    <input type="text" name="username"/>
                </td>
            </tr>
            <tr>
                <td>密码：</td>
                <td>
                    <input type="password" name="password"/>
                </td>
            </tr>
            <tr>
                <td>
                    <input type="submit" value="登录"/>
                </td>
            </tr>
        </table>
    </form>
    </body>
    </html>
    

index.html：

    
    
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Title</title>
    </head>
    <body>
        <p>欢迎回来</p>
        <form action="/logout" method="post" >
            <input type="submit" value="退出"/>
        </form>
    </body>
    </html>
    

admin.html：

    
    
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <title>admin</title>
    </head>
    <body>
        <p>后台管理系统</p>
        <a href="">用户管理</a>
        <a href="">商品管理</a>
        <a href="">订单管理</a>
        <form action="/logout" method="post">
            <button  type="submit">退出</button>
        </form>
    </body>
    </html>
    

启动 Application，访问 http://localhost:8080/index，直接跳转到登录页面，如下所示。

![](https://images.gitbook.cn/69c0f800-c761-11e9-a5ba-f1eeeb548c06)

USER 和 ADMIN 都拥有访问 index.html 的权限，所以两个用户都可以访问，如果输入的用户名和密码有错，返回到 login.html
并给出提示信息，如下所示。

![](https://images.gitbook.cn/74eb17b0-c761-11e9-a5ba-f1eeeb548c06)

输入 user/000 或者 admin/123，则登录成功，跳转到 index.html，如下所示。

![](https://images.gitbook.cn/9978b290-c761-11e9-99c1-c37abd23c4b1)

如果访问 admin.html，则 USER 无法访问，只有 ADMIN 有访问权限，即只有 admin/123 可以访问，登录成功之后跳转到
admin.html 如下所示。

![](https://images.gitbook.cn/a65193b0-c761-11e9-99c1-c37abd23c4b1)

### 总结

本节课我们讲解了 Spring Boot 整合安全框架 Spring Security 的具体操作，相比于 Shiro，Spring Security
因其是 Spring 全家桶的一员，可以和 Spring 应用无缝衔接，所以使用起来更加方便，系统安全是每个项目的重中之重，使用 Spring
Security 可以对访问权限进行严格管理，提高项目的安全性。

[请点击这里查看源码](https://github.com/southwind9801/gcspringbootsecurity.git)

[点击这里获取 Spring Boot
视频专题](https://pan.baidu.com/s/1K2cNTk6JmZa50RYSKwvwGA)，提取码：e4wc

