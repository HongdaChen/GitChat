### 前言

数据校验是每个项目中必不可少的模块，Spring MVC 作为一款成熟的框架，也为我们提供了校验组件，有两种方式可供开发者选择：（1）基于
Validator 接口进行校验；（2）使用 Annotaion JSR-303 标准进行校验。

使用 Validator 接口进行数据校验会稍微复杂一些，具体的数据验证规则需要开发者手动进行设置。而使用 Annotaion JSR-303
标准就相对简单很多，开发者不需要编写验证逻辑，直接通过注解的形式就可以给每一条数据添加验证规则，具体操作是直接在实体类的属性上添加对于的校验规则即可，使用起来更加方便。

两种校验方式的具体使用如下所示。

### 基于 Validator 接口

我们通过学生登录的场景来学习使用基于 Validator 接口的验证器。

（1）创建实体类 Student

    
    
    public class Student {
        private String name;
        private String password;
    }
    

（2）自定义校验器 StudentValidation，实现 Validator 接口，重新接口的抽象方法，加入校验规则

    
    
    public class StudentValidator implements Validator{
    
        public boolean supports(Class<?> clazz) {
            // TODO Auto-generated method stub
            return Student.class.equals(clazz);
        }
    
        public void validate(Object target, Errors errors) {
            // TODO Auto-generated method stub
            ValidationUtils.rejectIfEmpty(errors, "name", null, "姓名不能为空"); 
            ValidationUtils.rejectIfEmpty(errors, "password", null, "密码不能为空");
        }
    
    }
    

（3）创建控制器 HelloHandler，在业务方法 login 参数列表中的 @Validated 表示参数 student
是需要校验的对象，@BindingResult 用来存储错误信息，两者缺一不可，而且必须挨着写，不能中间有其他参数。

    
    
    @Controller
    @RequestMapping("/hello")
    public class HelloHandler {
    
        @GetMapping(value = "/login")
        public String login(Model model){
            model.addAttribute(new Student());
            return "login";
        }
    
        @PostMapping(value="/login")
        public String login(@Validated Student student,BindingResult br) {
            if (br.hasErrors()) {
                return "login";
            }
            return "success";
        }
    
    }
    

（4）在 springmvc.xml 中配置 validator

    
    
    <!-- 配置自动扫描 -->
    <context:component-scan base-package="com.southwind"></context:component-scan>
    <!-- 配置视图解析器 -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <!-- 前缀 -->
        <property name="prefix" value="/"></property>
        <!-- 后缀 -->
        <property name="suffix" value=".jsp"></property>
        </bean>
    <!-- 基于 Validator 的配置 -->
    <mvc:annotation-driven validator="studentValidator"/>
    <bean id="studentValidator" class="com.southwind.validator.StudentValidator"/>
    

（5）login.jsp

    
    
    <h1>学生登录</h1>
    <form:form modelAttribute="student" action="login" method="post">
        学生姓名：<form:input path="name" /><form:errors path="name"/><br/>
        学生密码：<form:password path="password" /><form:errors path="password"/><br/>
        <input type="submit" value="提交"/>
    </form:form>
    

（6）运行，通过地址栏发送 GET 请求访问 login 方法，绑定模型数据，然后页面跳转到
login.jsp，如下所示，不输入用户名密码直接单击提交按钮。

![](https://images.gitbook.cn/452cc7e0-9ade-11e8-8cbe-ad3f3badcc18)

单击提交按钮后，form 表单发送 POST 请求访问 login 方法，完成数据校验，并将校验结果再次返回到 login.jsp，如下图所示。

![](https://images.gitbook.cn/4d637120-9ade-11e8-a178-519d5b470954)

### Annotaion JSR-303 标准

使用 Annotation JSR-303 标准进行验证，需要导入支持这种标准的 jar 包，这里我们使用 Hibernate Validator。

标准详解如下所示：

![](https://images.gitbook.cn/8abe0c30-9ae1-11e8-831e-0180aea56660)

接下来通过用户注册的场景来学习使用 JSR-303 标准进行数据校验。

（1）在 pom.xml 中添加 Hibernate Validator 依赖。

    
    
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.0.8.RELEASE</version>
    </dependency>
    <!-- JRS-303 -->
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-validator</artifactId>
        <version>5.1.3.Final</version>
    </dependency>
    <dependency>
        <groupId>javax.validation</groupId>
        <artifactId>validation-api</artifactId>
        <version>1.1.0.Final</version>
    </dependency>
    <dependency>
        <groupId>org.jboss.logging</groupId>
        <artifactId>jboss-logging</artifactId>
        <version>3.1.1.GA</version>
     </dependency>
    <!-- 解决 JDK9 以上版本没有 JAXB API jar 的问题，JDK9 以下版本不需要配置 -->
    <dependency>
       <groupId>javax.xml.bind</groupId>
       <artifactId>jaxb-api</artifactId>
       <version>2.3.0</version>
    </dependency>
    <dependency>
       <groupId>com.sun.xml.bind</groupId>
       <artifactId>jaxb-impl</artifactId>
       <version>2.3.0</version>
    </dependency>
    <dependency>
        <groupId>com.sun.xml.bind</groupId>
        <artifactId>jaxb-core</artifactId>
        <version>2.3.0</version>
    </dependency>
    <dependency>
        <groupId>javax.activation</groupId>
        <artifactId>activation</artifactId>
        <version>1.1.1</version>
    </dependency>
    

这里我们需要注意，如果环境是 JDK9 以下的，可以不用添加以下四个依赖；如果环境是 JDK9 以上版本，必须添加这四个依赖。

    
    
    <dependency>
        <groupId>javax.xml.bind</groupId>
        <artifactId>jaxb-api</artifactId>
        <version>2.3.0</version>
    </dependency>
    <dependency>
        <groupId>com.sun.xml.bind</groupId>
        <artifactId>jaxb-impl</artifactId>
        <version>2.3.0</version>
    </dependency>
    <dependency>
        <groupId>com.sun.xml.bind</groupId>
        <artifactId>jaxb-core</artifactId>
        <version>2.3.0</version>
    </dependency>
    <dependency>
        <groupId>javax.activation</groupId>
        <artifactId>activation</artifactId>
        <version>1.1.1</version>
    </dependency>
    

（2）创建实体类 User，通过注解的方式给属性指定校验规则。

校验规则详解如下所示：

![](https://images.gitbook.cn/dcda6720-9ae1-11e8-8cbe-ad3f3badcc18)

创建 User 实体类，并通过注解给每个属性添加校验规则：

    
    
    public class User {
        @NotEmpty(message = "用户名不能为空")
        private String username;
        @Size(min = 6,max = 20,message = "密码长度为6-12位")
        private String password;
        @Email(regexp = "^[a-zA-Z0-9_.-]+@[a-zA-Z0-9-]+(\\.[a-zA-Z0-9-]+)*\\.[a-zA-Z0-9]{2,6}$", message = "请输入正确的邮箱格式")
        private String email;
        @Pattern(regexp = "^((13[0-9])|(14[5|7])|(15([0-3]|[5-9]))|(18[0,5-9]))\\\\d{8}$",message="请输入正确的电话格式")
        private String phone;
    }
    

（3）创建控制器 HelloHandler，在业务方法 register 使用 @Valid 来绑定校验对象，@BindingResult 来保存错误信息。

    
    
    @Controller
    @RequestMapping("/hello")
    public class HelloHandler {
    
        @GetMapping(value="/register")
        public String register(Model model){
            model.addAttribute(new User());
            return "register";
        }
    
        @PostMapping(value="/register")
        public String register2(@Valid User user,BindingResult br) {
            if (br.hasErrors()){
                return "register";
            }
            return "success";
        }
    
    }
    

（4）在 springmvc.xml 中添加配置

    
    
    <!-- 配置自动扫描 -->
    <context:component-scan base-package="com.southwind"></context:component-scan>
    <!-- 配置视图解析器 -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <!-- 前缀 -->
        <property name="prefix" value="/"></property>
        <!-- 后缀 -->
        <property name="suffix" value=".jsp"></property>
    </bean>
    <!-- JSR-303配置 -->
    <mvc:annotation-driven />
    

（5）register.jsp

    
    
    <h1>用户注册</h1>
    <form:form modelAttribute="user" action="register" method="post">
         用户名：<form:input path="username" /><form:errors path="username" /><br/>
         密码：<form:password path="password" /><form:errors path="password" /><br/>
         邮箱：<form:input path="email" /><form:errors path="email" /><br/>
         电话：<form:input path="phone" /><form:errors path="phone" /><br/>
         <input type="submit" value="提交"/>
    </form:form>
    

（6）运行，通过地址栏发送 GET 请求访问 register 方法，绑定模型数据，然后页面跳转到
register.jsp，如下所示，不输入用户名、密码、邮箱、电话信息，直接单击提交按钮。

![](https://images.gitbook.cn/09dcc5b0-9ae2-11e8-8cbe-ad3f3badcc18)

单击提交按钮后，form 表单发送 POST 请求访问 register 方法，完成数据校验，并将校验结果再次返回到 register.jsp，如下所示。

![](https://images.gitbook.cn/14853240-9ae2-11e8-a178-519d5b470954)

### 总结

本节课我们讲解了 Spring MVC 对于数据校验的支持，Spring MVC 提供了两种数据校验的方式，基于 Validator 接口的校验、使用
Annotaion JSR-303 标准进行校验。Annotaion JSR-303 标准可以结合注解来完成校验工作，实际开发中推荐使用。

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

[请单击这里下载源码](https://github.com/southwind9801/springmvc_datavalidator.git)

