### 前言

本节课我们来实现服务提供者 account，account 为系统提供所有的账户相关业务，包括用户和管理员登录、退出，具体实现如下所示。

1\. 在父工程下创建建一个 Module，命名为 account，pom.xml 添加相关依赖，account 需要访问数据库，因此要添加 MyBatis
相关依赖，同时配置文件从 Git 仓库拉取，所以还要添加 Spring Cloud Config 相关依赖。

    
    
    <dependencies>
        <!-- eurekaclient -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
            <version>2.0.0.RELEASE</version>
        </dependency>
        <!-- MyBatis -->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.0</version>
        </dependency>
        <!-- MySQL 驱动 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.11</version>
        </dependency>
        <!-- 配置中心 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
    </dependencies>
    

2\. 在 resources 目录下创建 bootstrap.yml，在该文件中配置拉取 Git 仓库相关配置文件的信息。

    
    
    spring:
      cloud:
        config:
          name: account #对应的配置文件名称
          label: master #Git 仓库分支名
          discovery:
            enabled: true
            serviceId: configserver #连接的配置中心名称
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8761/eureka/
    

3\. 在 java 目录下创建启动类 AccountApplication。

    
    
    @SpringBootApplication
    public class AccountApplication {
        public static void main(String[] args) {
            SpringApplication.run(AccountApplication.class,args);
        }
    }
    

4\. 接下来为服务提供者 account 集成 MyBatis 环境，首先在 Git 仓库配置文件 account.yml 中添加相关信息。

    
    
    server:
      port: 8010
    spring:
      application:
        name: account
      datasource:
        name: orderingsystem
        url: jdbc:mysql://localhost:3306/orderingsystem?useUnicode=true&characterEncoding=UTF-8
        username: root
        password: root
    eureka:
      client:
        service-url:
          defaultZone: http://localhost:8761/eureka/
      instance:
        prefer-ip-address: true
    mybatis:
      mapper-locations: classpath:mapping/*.xml
      type-aliases-package: com.southwind.entity
    

5\. 在 java 目录下创建 entity 文件夹，新建 Account 类。

    
    
    @Data
    public class Account {
        private long id;
        private String username;
        private String password;
        private String nickname;
        private String gender;
        private String telephone;
        private Date registerdate;
        private String address;
    }
    

6\. 新建 User 类继承 Account 类，对应数据表 t_user。

    
    
    @Data
    public class User extends Account{
    
    }
    

7\. 新建 Admin 类继承 Account 类，对应数据表 t_admin。

    
    
    @Data
    public class Admin extends Account{
    
    }
    

8\. 在 java 目录下创建 repository 文件夹，新建 UserRepository 接口。

    
    
    public interface UserRepository {
        public User login(String username,String password);
    }
    

9\. 新建 AdminRepository 接口。

    
    
    public interface AdminRepository {
        public Admin login(String username,String password);
    }
    

10\. 在 resources 目录下创建 mapping 文件夹，存放 Mapper.xml，新建 UserRepository.xml，编写
UserRepository 接口方法对应的 SQL。

    
    
    <mapper namespace="com.southwind.repository.UserRepository">
        <select id="login" resultType="User">
            select * from t_user where username = #{param1} and password = #{param2}
        </select>
    </mapper>
    

11\. 新建 AdminRepository.xml，编写 AdminRepository 接口方法对应的 SQL。

    
    
    <mapper namespace="com.southwind.repository.AdminRepository">
        <select id="login" resultType="Admin">
            select * from t_admin where username = #{param1} and password = #{param2}
        </select>
    </mapper>
    

12\. 将 Mapper 注入，在启动类添加注解 @MapperScan("com.southwind.repository")。

    
    
    @SpringBootApplication
    @MapperScan("com.southwind.repository")
    public class AccountApplication {
        public static void main(String[] args) {
            SpringApplication.run(AccountApplication.class,args);
        }
    }
    

13\. 新建 AccountHandler，将 UserRepository 和 AdminRepository 通过 @Autowired
注解进行注入，完成相关业务逻辑。

    
    
    @RestController
    @RequestMapping("/account")
    public class AccountHandler {
    
        @Autowired
        private UserRepository userRepository;
        @Autowired
        private AdminRepository adminRepository;
    
        @GetMapping("/login/{username}/{password}/{type}")
        public Account login(@PathVariable("username") String username,@PathVariable("password") String password,@PathVariable("type") String type){
            Account account = null;
            switch (type){
                case "user":
                    account = userRepository.login(username, password);
                    break;
                case "admin":
                    account = adminRepository.login(username, password);
                    break;
            }
            return account;
        }
    }
    

14\. 依次启动注册中心、configserver、AccountApplication，使用 Postman 测试该服务的相关接口，如图所示。

![1](https://images.gitbook.cn/84676cc0-dd55-11e9-9cc8-a572519b0723)

### 总结

本节课我们讲解了项目实战 account 模块的搭建，作为一个服务提供者，account 为整个系统提供账户服务，包括用户和管理员登录。

[请单击这里下载源码](https://github.com/southwind9801/orderingsystem.git)

[微服务项目实战视频链接请点击这里获取](https://pan.baidu.com/s/1eheDU4XoN3BKuzocyIe0oA)，提取码：bfps

