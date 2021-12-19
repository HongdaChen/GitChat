### 前言

这节课我们用本阶段所学知识实现一个权限管理系统，帮助大家掌握所学技术在实际开发中的使用。

需求简介：

  1. 权限管理：查询、创建、修改、删除
  2. 角色管理：查询、创建、修改、删除、添加权限
  3. 用户管理：查询、创建、修改、删除、添加角色
  4. 用户登录：登录成功根据该用户角色动态生成权限菜单

开发环境：

  * JDK 10.0.1
  * Maven 3.5.3
  * tomcat 9.0.8
  * Eclipse Photon

技术选型：

> layui-v2.3.0 + Spring MVC 4.3.1 + Spring Data 1.8.0 + MongoDB 4.0.0

代码实现：

1\. 新建 Maven Project，选择 webapp 项目。

![](https://images.gitbook.cn/7b248f00-9ae2-11e8-a178-519d5b470954)

2\. pom.xml 添加相关依赖。

    
    
    <!-- JSTL -->
    <dependency>
        <groupId>jstl</groupId>
        <artifactId>jstl</artifactId>
        <version>1.2</version>
    </dependency>
    
    <!-- ServletAPI -->
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <version>3.0.1</version>
        <scope>provided</scope>
    </dependency>
    
    <!-- Spring MVC -->
    <dependency>
         <groupId>org.springframework</groupId>
         <artifactId>spring-webmvc</artifactId>
         <version>4.3.1.RELEASE</version>
    </dependency>
    
    <!-- Spring Data MongoDB -->
    <dependency>
         <groupId>org.springframework.data</groupId>
         <artifactId>spring-data-mongodb</artifactId>
         <version>1.8.0.RELEASE</version>
    </dependency>
    
    <!-- alifastjson -->
    <dependency>
         <groupId>com.alibaba</groupId>
         <artifactId>fastjson</artifactId>
         <version>1.2.18</version>
    </dependency>
    

3\. 创建实体类 Authority，添加注解与 MongoDB 集合进行映射。

    
    
    @Document
    public class Authority {
        @Id
        private String id;
        @Field
        private String name;
        private boolean has;    
    }
    

4\. 创建 AuthorityVO 类，前端使用 layui 框架，数据表格通过异步加载的方式渲染数据，后台需要返回符合 layui 格式的 JSON
数据，官方文档给出的格式如下，所以我们需要创建 AuthorityVO 类，目的是将 Authority 按照 layui 要求的格式进行封装。

![](https://images.gitbook.cn/9450a680-9ae2-11e8-8cbe-ad3f3badcc18)

AuthorityVO：

    
    
    public class AuthorityVO {
        private int code;
        private String mess;
        private long count;
        private List<Authority> data;
    }
    

同理创建 Role 和 User 的 VO 类。

5\. 创建 Reposity 接口，继承 CrudReposity 接口，完成对数据层的访问和操作。

AuthorityReposity：

    
    
    @Repository
    public interface AuthorityRepository extends CrudRepository<Authority, String>{
        public Authority findById(String id);
        public void deleteById(String id);
        //分页查询
        public PageImpl findAll(Pageable pageable);
    }
    

RoleReposity：

    
    
    @Repository
    public interface RoleRepository extends CrudRepository<Role, String>{
        public Role findById(String id);
        public void deleteById(String id);
        //分页查询
        public PageImpl findAll(Pageable pageable);
    }
    

UserReposity：

    
    
    @Repository
    public interface UserRepository extends CrudRepository<User, String>{
        public User findById(String id);
        public void deleteById(String id);
        public User findByName(String name);
        //分页查询
        public PageImpl findAll(Pageable pageable);
    }
    

6\. 创建控制层 Handler。

AuthorityHandler：

    
    
    @Controller
    @RequestMapping("/authority")
    public class AuthorityHandler {
    
        @Autowired
        private AuthorityRepository authorityRepository;
    
    }
    

RoleHandler：

    
    
    @Controller
    @RequestMapping("/role")
    public class RoleHandler {
    
        @Autowired
        private RoleRepository roleRepository;
        @Autowired
        private AuthorityRepository authorityRepository;
    
    }
    

UserHandler：

    
    
    @Controller
    @RequestMapping("/user")
    public class UserHandler {
    
        @Autowired
        private RoleRepository roleRepository;
        @Autowired
        private UserRepository userRepository;
    
    }
    

LoginHandler：

    
    
    @Controller
    @RequestMapping("/login")
    public class LoginHandler {
    
        @Autowired
        private UserRepository userRepository;
        @Autowired
        private RoleRepository roleRepository;
        @Autowired
        private AuthorityRepository authorityRepository;
    
    }
    

7\. 配置文件。

spring.xml：

    
    
    <!-- Spring 连接 MongoDB 客户端配置 -->
    <mongo:mongo-client host="127.0.0.1" port="12345" id="mongo"/>
    
    <!-- 配置 MongoDB 目标数据库 -->
    <mongo:db-factory dbname="testdb" mongo-ref="mongo" />
    
    <!-- 配置 MongoTemplate -->
    <bean id="mongoTemplate" class="org.springframework.data.mongodb.core.MongoTemplate">
      <constructor-arg name="mongoDbFactory" ref="mongoDbFactory"/>
     </bean>
    
    <!-- 扫描 Reposity 接口 -->
    <mongo:repositories base-package="com.southwind.repository"></mongo:repositories>
    

springmvc.xml：

    
    
    <!-- 配置自动扫描 -->
    <context:component-scan base-package="com.southwind"></context:component-scan>
    
    <!-- 配置视图解析器 -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <!-- 前缀 -->
        <property name="prefix" value="/"></property>
        <!-- 后缀 -->
        <property name="suffix" value=".jsp"></property>
    </bean>
    
    <mvc:annotation-driven >
        <!-- 消息转换器 -->
        <mvc:message-converters register-defaults="true">
           <!-- 阿里 fastjson -->
           <bean class="com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter4"/>
        </mvc:message-converters>
    </mvc:annotation-driven>
    

8\. JSP 页面使用 layui 框架完成布局和渲染。

程序截图：

![](https://images.gitbook.cn/90632330-9ae3-11e8-b37c-dd4feba3837e)

![](https://images.gitbook.cn/9811d7c0-9ae3-11e8-8cbe-ad3f3badcc18)

![](https://images.gitbook.cn/b1672cb0-9ae4-11e8-8cbe-ad3f3badcc18)

![](https://images.gitbook.cn/b5a5e7d0-9ae4-11e8-b37c-dd4feba3837e)

### 总结

本节课我们用 Spring MVC + LayUI + Spring Data + MongoDB
的技术选型实现了一个项目整合案例，第一个目的是带领大家对本阶段学习的重点知识点进行系统性梳理，另外一个目的是给大家提供一个新的技术选型，包括前、后端分离的一些简单应用，相信这套技术选型对大家会有一定的帮助。

### 源码

[请点击这里查看源码](https://github.com/southwind9801/AuthorityManagement.git)

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。
>
> 温馨提示：需购买才可入群哦，加小助手微信后需要截已购买的图来验证~

