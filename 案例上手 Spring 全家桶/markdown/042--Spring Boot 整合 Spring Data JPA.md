### 前言

上节课我们学习了 Spring Boot 整合 MyBatis 的具体操作，MyBatis
是一款优秀的持久层框架，可以帮助开发者简化代码开发，从而提高工作效率。作为行业 Java 开发的行业标准，功能强大的 Spring
当然也提供了持久层解决方案，它就是 Spring Data。

Spring Data 又根据具体的数据库分为不同的子模块，无论是关系型数据库还是非关系型数据库，Spring Data 都提供了良好的支持，比如
Spring Data JPA 就可以完成对关系型数据库的访问，Spring Data MongoDB、Spring Data Redis
则分别对应非关系数据库中的 MongoDB 和 Redis。

我们这节课要讲解的是 Spring Data JPA，它是 Spring Data
大家族中的一员，在学习具体使用方法之前，我们先要搞清楚几个容易混淆的概念。

**JPA 和 Spring Data JPA 是一回事吗？**

JPA 和 Spring Data JPA 是完全不同的两个概念，JPA（Java Persistence API）是 Java 持久层规范，定义了一些列
ORM 接口，它本身是不能直接使用的，因为接口需要实现才能使用，Hibernate 框架就是实现 JPA 规范的框架。

Spring Data JPA 是 Spring 框架提供的对 JPA
规范的抽象，通过约定的命名规范完成持久层接口的编写，在不需要实现接口的情况下，就可以实现对数据库的操作。简单理解就是通过 Spring Data
JPA，你只需要定义接口而不需要实现，就能完成 CRUD 操作。

Spring Data JPA 本身并不是一个具体实现，它只是一个抽象层，底层还是需要 Hibernate 这样的 JPA 实现来提供支持。

**Spring Data JPA 和 Spring JdbcTemplate 有什么关联？**

两者没有关联，Spring JdbcTemplate 是 Spring 框架提供的一套操作数据库的模版，Spring Data JPA 是 JPA
规范的抽象。

搞清楚了几个概念的区别之后，我们进入正题，实现 Spring Boot 整合 Spring Data JPA，数据库我们选择 MySQL。

1\. 创建 Maven 工程，pom.xml 中添加相关依赖，spring-boot-starter-data-jpa 是 Spring Boot 整合
Spring Data JPA 的依赖，同时别忘了添加 MySQL 驱动依赖。

    
    
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
        <artifactId>spring-boot-starter-data-jpa</artifactId>
      </dependency>
    
      <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.15</version>
      </dependency>
    
      <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
      </dependency>
    </dependencies>
    

2\. 创建数据表。

    
    
    create table student(
      id int primary key auto_increment,
      name varchar(11),
      score double,
      birthday date
    );
    

预先向数据库添加 3 条记录，如下所示。

![](https://images.gitbook.cn/4a54f810-c1d8-11e9-9969-976e2ac29eb2)

3\. 创建数据表对应的实体类，Spring Data JPA 支持 ORM
规范，可以直接将实体类与数据表进行映射，具体的一行数据可以直接映射成一个实例化对象。我们在定义实体类的时候，就需要添加相关注解完成实体类与表，成员变量与表中字段的映射，常用的注解如下所示。

  * @Entity 将实体类与数据表进行映射。
  * @Id 将实体类中的成员变量与数据表的主键进行映射，一般都是 id。
  * @GeneratedValue 表示自动生成主键，strategy 为主键生成策略，在 MySQL 数据库中我们一般选择自增的方式。
  * @Column 将实体类中的成员变量与数据表的普通字段进行映射。

实体类的具体代码如下所示。

    
    
    @Data
    @Entity
    public class Student {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;
        @Column
        private String name;
        @Column
        private Double score;
        @Column
        private Date birthday;
    }
    

4\. 创建 StudentRepository 接口，直接继承 JpaRepository
接口即可，泛型列表中第一个泛型为实体类的数据类型，第二个泛型为实体类中与主键映射的成员变量数据类型，在这里就是 Student 和 Long。

同时我们只需要定义接口即可，而不需要完成接口的实现，这一点和 MyBatis 框架相类似，但是使用 Spring Data JPA 比 MyBatis
更加方便，MyBatis 虽然也不需要实现 Mapper 接口，但是开发者需要自己定义接口方法的 SQL，同时设置解析策略，所以说 MyBatis
只是一个半自动化的 ORM 框架，一部分工作仍需要开发者来完成。

Spring Data JPA 是一个全自动化的 ORM 框架，底层是 Hibernate 框架提供支持，开发者只需要调用接口方法即可，不必关心 SQL
语句和结果集的解析，非常方便。

    
    
    public interface StudentRepository extends JpaRepository<Student,Long> {
    }
    

5\. 创建 StudentHandler，同时将 StudentRepository 直接注入。

    
    
    @RestController
    public class StudentHandler {
        @Autowired
        private StudentRepository studentRepository;
    
        @GetMapping("/findAll")
        public List<Student> findAll(){
            return studentRepository.findAll();
        }
    
        @GetMapping("/findById/{id}")
        public Student get(@PathVariable("id") Long id){
            return studentRepository.findById(id).get();
        }
    
        @PostMapping("/save")
        public Student save(@RequestBody Student student){
            return studentRepository.save(student);
        }
    
        @PutMapping("/update")
        public Student update(@RequestBody Student student){
            return studentRepository.save(student);
        }
    
        @DeleteMapping("/deleteById/{id}")
        public void deleteById(@PathVariable("id") Long id){
            studentRepository.deleteById(id);
        }
    
    }
    

6\. 创建配置文件 application.yml，添加了 MySQL 数据源和 Spring Data JPA 的相关配置，如格式化打印 SQL 语句。

    
    
    spring:
      datasource:
        url: jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=UTF-8
        username: root
        password: root
        driver-class-name: com.mysql.cj.jdbc.Driver
      jpa:
        show-sql: true            #输出SQL
        properties:
          hibernate:
            format_sql: true      #格式化SQL
    

7\. 创建启动类 Application。

    
    
    @SpringBootApplication
    public class Application {
        public static void main(String[] args) {
            SpringApplication.run(Application.class,args);
        }
    }
    

8\. 启动 Application，使用 Postman 工具来测试相关接口，结果如下所示。

  * findAll

![](https://images.gitbook.cn/6472c290-c1d8-11e9-97a8-35dcf136a505)

同时我们可以在控制台看到由 Hibernate 输出的 SQL 语句，如下图所示。

![](https://images.gitbook.cn/887ed160-c1d8-11e9-9166-bdb140d6509f)

  * findById

![](https://images.gitbook.cn/9af58870-c1d8-11e9-8621-c1fbe3716b21)

  * save

![](https://images.gitbook.cn/a3f30420-c1d8-11e9-9166-bdb140d6509f)

添加完成，测试 findAll，可以看到新数据已经添加成功。

![](https://images.gitbook.cn/b609e7f0-c1d8-11e9-9969-976e2ac29eb2)

  * update

![](https://images.gitbook.cn/c150de70-c1d8-11e9-8621-c1fbe3716b21)

修改完成，测试 findAll，可以看到数据已经修改成功。

![](https://images.gitbook.cn/ca8c7490-c1d8-11e9-8621-c1fbe3716b21)

  * deleteById

![](https://images.gitbook.cn/d38c1320-c1d8-11e9-8621-c1fbe3716b21)

删除完成，测试 findAll，可以看到数据已经删除成功。

![](https://images.gitbook.cn/db49e2e0-c1d8-11e9-97a8-35dcf136a505)

### 总结

本节课我们讲解了 Spring Boot 整合另外一个数据持久层解决方案 Spring Data JPA 的具体操作，说到 Spring Data
JPA，很多人容易混淆几个概念，什么是 JPA？什么是 Spring Data JPA？Spring Data JPA 和 Spring
JdbcTemplate 是不是一回事？我们只有先搞清楚这些技术的基本概念，才能更好地去应用。

[请点击这里查看源码](https://github.com/southwind9801/gcspringbootspringdatajpa.git)

[点击这里获取 Spring Boot
视频专题](https://pan.baidu.com/s/1K2cNTk6JmZa50RYSKwvwGA)，提取码：e4wc

