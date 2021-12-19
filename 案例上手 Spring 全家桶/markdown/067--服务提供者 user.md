### 前言

本节课我们来实现服务提供者 user，user 为系统提供用户相关服务，包括添加用户、查询用户、删除用户，具体实现如下所示。

1\. 在父工程下创建一个 Module，命名为 user，pom.xml 添加相关依赖，user 需要访问数据库，所以集成 MyBatis
相关依赖，配置文件从 Git 仓库拉取，添加配置中 Spring Cloud Config 相关依赖。

    
    
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
          name: user #对应的配置文件名称
          label: master #Git 仓库分支名
          discovery:
            enabled: true
            serviceId: configserver #连接的配置中心名称
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8761/eureka/
    

3\. 在 java 目录下创建启动类 UserApplication。

    
    
    @SpringBootApplication
    @MapperScan("com.southwind.repository")
    public class UserApplication {
        public static void main(String[] args) {
            SpringApplication.run(UserApplication.class,args);
        }
    }
    

4\. 接下来为服务提供者 user 集成 MyBatis 环境，首先在 Git 仓库配置文件 user.yml 中添加相关信息。

    
    
    server:
      port: 8050
    spring:
      application:
        name: user
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
    

5\. 在 java 目录下创建 entity 文件夹，新建 User 类对应数据表 t_user。

    
    
    @Data
    public class User {
        private long id;
        private String username;
        private String password;
        private String nickname;
        private String gender;
        private String telephone;
        private Date registerdate;
        private String address;
    }
    

6\. 新建 UserVO 类为 layui 框架提供封装类。

    
    
    @Data
    public class UserVO {
        private int code;
        private String msg;
        private int count;
        private List<User> data;
    }
    

7\. 在 java 目录下创建 repository 文件夹，新建 UserRepository 接口，定义相关业务方法。

    
    
    public interface UserRepository {
        public List<User> findAll(int index, int limit);
        public int count();
        public void save(User user);
        public void deleteById(long id);
    }
    

8\. 在 resources 目录下创建 mapping 文件夹，存放 Mapper.xml，新建 UserRepository.xml，编写
UserRepository 接口方法对应的 SQL。

    
    
    <mapper namespace="com.southwind.repository.UserRepository">
        <select id="findAll" resultType="User">
            select * from t_user order by id limit #{param1},#{param2}
        </select>
    
        <select id="count" resultType="int">
            select count(*) from t_user;
        </select>
    
        <insert id="save" parameterType="User">
            insert into t_user(username,password,nickname,gender,telephone,registerdate,address) values(#{username},#{password},#{nickname},#{gender},#{telephone},#{registerdate},#{address})
        </insert>
    
        <delete id="deleteById" parameterType="long">
            delete from t_user where id = #{id}
        </delete>
    </mapper>
    

9\. 新建 UserHandler，将 UserRepository 通过 @Autowired 注解进行注入，完成相关业务逻辑。

    
    
    @RestController
    @RequestMapping("/user")
    public class UserHandler {
    
        @Autowired
        private UserRepository userRepository;
    
        @GetMapping("/findAll/{page}/{limit}")
        public UserVO findAll(@PathVariable("page") int page, @PathVariable("limit") int limit){
            UserVO userVO = new UserVO();
            userVO.setCode(0);
            userVO.setMsg("");
            userVO.setCount(userRepository.count());
            userVO.setData(userRepository.findAll((page-1)*limit,limit));
            return userVO;
        }
    
        @PostMapping("/save")
        public void save(@RequestBody User user){
            user.setRegisterdate(new Date());
            userRepository.save(user);
        }
    
        @DeleteMapping("/deleteById/{id}")
        public void deleteById(@PathVariable("id") long id){
            userRepository.deleteById(id);
        }
    }
    

10\. 依次启动注册中心、configserver、UserApplication，使用 Postman 测试该服务的相关接口，如图所示。

![1](https://images.gitbook.cn/14a6ff60-dd58-11e9-aaec-b5744b419935)

### 总结

本节课我们讲解了项目实战 user 模块的搭建，作为一个服务提供者，user 为整个系统提供用户服务，包括添加用户、查询用户、删除用户。

[请单击这里下载源码](https://github.com/southwind9801/orderingsystem.git)

[微服务项目实战视频链接请点击这里获取](https://pan.baidu.com/s/1eheDU4XoN3BKuzocyIe0oA)，提取码：bfps

