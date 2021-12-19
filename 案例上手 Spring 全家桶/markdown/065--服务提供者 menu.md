### 前言

本节课我们来实现服务提供者 menu，menu 为系统提供菜品相关服务，包括添加菜品、查询菜品、修改菜品、删除菜品，具体实现如下所示。

1\. 在父工程下创建一个 Module，命名为 menu，pom.xml 添加相关依赖，menu 需要访问数据库，因此集成 MyBatis
相关依赖，配置文件从 Git 仓库拉取，添加配置中心 Spring Cloud Config 相关依赖。

    
    
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
          name: menu #对应的配置文件名称
          label: master #Git 仓库分支名
          discovery:
            enabled: true
            serviceId: configserver #连接的配置中心名称
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8761/eureka/
    

3\. 在 java 目录下创建启动类 MenuApplication。

    
    
    @SpringBootApplication
    @MapperScan("com.southwind.repository")
    public class MenuApplication {
        public static void main(String[] args) {
            SpringApplication.run(MenuApplication.class,args);
        }
    }
    

4\. 接下来为服务提供者 menu 集成 MyBatis 环境，首先在 Git 仓库配置文件 menu.yml 中添加相关信息。

    
    
    server:
      port: 8020
    spring:
      application:
        name: menu
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
    

5\. 在 java 目录下创建 entity 文件夹，新建 Menu 类对应数据表 t_menu。

    
    
    @Data
    public class Menu {
        private long id;
        private String name;
        private double price;
        private String flavor;
        private Type type;
    }
    

6\. 新建 MenuVO 类为 layui 框架提供封装类。

    
    
    @Data
    public class MenuVO {
        private int code;
        private String msg;
        private int count;
        private List<Menu> data;
    }
    

7\. 新建 Type 类对应数据表 t_type。

    
    
    @Data
    public class Type {
        private long id;
        private String name;
    }
    

8\. 在 java 目录下创建 repository 文件夹，新建 MenuRepository 接口，定义相关业务方法。

    
    
    public interface MenuRepository {
        public List<Menu> findAll(int index,int limit);
        public int count();
        public void save(Menu menu);
        public Menu findById(long id);
        public void update(Menu menu);
        public void deleteById(long id);
    }
    

9\. 新建 TypeRepository 接口，定义相关业务方法。

    
    
    public interface TypeRepository {
        public List<Type> findAll();
    }
    

10\. 在 resources 目录下创建 mapping 文件夹，存放 Mapper.xml，新建 MenuRepository.xml，编写
MenuRepository 接口方法对应的 SQL。

    
    
    <mapper namespace="com.southwind.repository.MenuRepository">
        <resultMap id="menuMap" type="Menu">
            <id property="id" column="mid"/>
            <result property="name" column="mname"/>
            <result property="author" column="author"/>
            <result property="price" column="price"/>
            <result property="flavor" column="flavor"/>
            <!-- 映射 type -->
            <association property="type" javaType="Type">
                <id property="id" column="tid"/>
                <result property="name" column="tname"/>
            </association>
        </resultMap>
    
        <select id="findAll" resultMap="menuMap">
            select m.id mid,m.name mname,m.price,m.flavor,t.id tid,t.name tname from t_menu m,t_type t where m.tid = t.id order by mid limit #{param1},#{param2}
        </select>
    
        <select id="count" resultType="int">
            select count(*) from t_menu;
        </select>
    
        <insert id="save" parameterType="Menu">
            insert into t_menu(name,price,flavor,tid) values(#{name},#{price},#{flavor},#{type.id})
        </insert>
    
        <select id="findById" resultMap="menuMap">
            select id mid,name mname,price,flavor,tid from t_menu where id = #{id}
        </select>
    
        <update id="update" parameterType="Menu">
            update t_menu set name = #{name},price = #{price},flavor = #{flavor},tid = #{type.id} where id = #{id}
        </update>
    
        <delete id="deleteById" parameterType="long">
            delete from t_menu where id = #{id}
        </delete>
    </mapper>
    

11\. 新建 TypeRepository.xml，编写 TypeRepository 接口方法对应的 SQL。

    
    
    <mapper namespace="com.southwind.repository.TypeRepository">
        <select id="findAll" resultType="Type">
            select * from t_type
        </select>
    </mapper>
    

12\. 新建 MenuHandler，将 MenuRepository 通过 @Autowired 注解进行注入，完成相关业务逻辑。

    
    
    @RestController
    @RequestMapping("/menu")
    public class MenuHandler {
    
        @Autowired
        private MenuRepository menuRepository;
        @Autowired
        private TypeRepository typeRepository;
    
        @GetMapping("/findAll/{page}/{limit}")
        public MenuVO findAll(@PathVariable("page") int page, @PathVariable("limit") int limit){
            MenuVO menuVO = new MenuVO();
            menuVO.setCode(0);
            menuVO.setMsg("");
            menuVO.setCount(menuRepository.count());
            menuVO.setData(menuRepository.findAll((page-1)*limit,limit));
            return menuVO;
        }
    
        @GetMapping("/findAll")
        public List<Type> findAll(){
            return typeRepository.findAll();
        }
    
        @PostMapping("/save")
        public void save(@RequestBody Menu menu){
            menuRepository.save(menu);
        }
    
        @GetMapping("/findById/{id}")
        public Menu findById(@PathVariable("id") long id){
            return menuRepository.findById(id);
        }
    
        @PutMapping("/update")
        public void update(@RequestBody Menu menu){
            menuRepository.update(menu);
        }
    
        @DeleteMapping("/deleteById/{id}")
        public void deleteById(@PathVariable("id") long id){
            menuRepository.deleteById(id);
        }
    }
    

13\. 依次启动注册中心、configserver、MenuApplication，使用 Postman 测试该服务的相关接口，如图所示。

![1](https://images.gitbook.cn/f89a9630-dd55-11e9-9cc8-a572519b0723)

![2](https://images.gitbook.cn/feb53d40-dd55-11e9-8134-9900814ad853)

![3](https://images.gitbook.cn/04d38dd0-dd56-11e9-8134-9900814ad853)

### 总结

本节课我们讲解了项目实战 menu 模块的搭建，作为一个服务提供者，menu 为整个系统提供菜品服务，包括添加菜品、查询菜品、修改菜品、删除菜品。

[请单击这里下载源码](https://github.com/southwind9801/orderingsystem.git)

[微服务项目实战视频链接请点击这里获取](https://pan.baidu.com/s/1eheDU4XoN3BKuzocyIe0oA)，提取码：bfps

