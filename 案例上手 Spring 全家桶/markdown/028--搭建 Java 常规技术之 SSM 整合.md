### 前言

在前面的课程中，我们已经分别学习了 Spring、Spring MVC、MyBatis 框架的使用，在实际开发中我们通常会将这 3
个框架整合起来使用，就是所谓的 SSM 框架，是目前企业中比较常用的一种开发方式，首先来回顾一下这 3 个框架的基本概念。

**Spring**

Spring 是 2003 年兴起的一个轻量级的企业级开发框架，可以替代传统 Java 应用的 EJB
开发方式，解决企业应用开发的复杂性问题。经过十几年的发展，现在的 Spring 已经不仅仅是一个替代 EJB 的框架了，Spring Framework
已经发展成为一套完整的生态，为现代软件开发的各个方面提供解决方案，是目前 Java 开发的行业标准。

**Spring MVC**

Spring MVC 全称为 Spring Web MVC，是 Spring 家族的一员，基于 Spring 实现 MVC 设计模式的框架，Spring
MVC 使得服务端开发变得更加简单方便。

**MyBatis**

MyBatis 是当前主流的 ORM 框架，完成对 JDBC 的封装，使得开发者可以更加方便地进行持久层代码开发，它的特点是简单易上手，更加灵活。

在 SSM 框架整合架构中，Spring、Spring MVC、MyBatis 分别负责不同的业务模块，共同完成企业级项目的开发需求。具体来讲，Spring
MVC 负责 MVC 设计模式的实现，MyBatis 提供了数据持久层解决方案，Spring 来管理 Spring MVC 和 MyBatis，IoC
容器负责 Spring MVC 和 MyBatis 相关对象的创建和依赖注入，AOP 负责事务管理，SSM 整体结构如下图所示。

![](https://images.gitbook.cn/e5844800-b7eb-11e9-acbd-8feb0c5ec6b6)

了解完 SSM 框架整合的基本理解，接下来我们就来动手实现一个 SSM 框架的整合。

1\. 创建 Maven Web 工程，pom.xml 添加相关依赖。

    
    
    <dependencies>
      <!-- SpringMVC -->
      <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.0.11.RELEASE</version>
      </dependency>
    
      <!-- Spring JDBC -->
      <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-jdbc</artifactId>
        <version>5.0.11.RELEASE</version>
      </dependency>
    
      <!-- Spring AOP -->
      <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>5.0.11.RELEASE</version>
      </dependency>
      <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aspects</artifactId>
        <version>5.0.11.RELEASE</version>
      </dependency>
    
      <!-- MyBatis -->
      <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.4.5</version>
      </dependency>
    
      <!-- MyBatis 整合 Spring -->
      <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis-spring</artifactId>
        <version>1.3.1</version>
      </dependency>
    
      <!-- MySQL 驱动 -->
      <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.11</version>
      </dependency>
    
      <!-- C3P0 -->
      <dependency>
        <groupId>c3p0</groupId>
        <artifactId>c3p0</artifactId>
        <version>0.9.1.2</version>
      </dependency>
    
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
        <version>3.1.0</version>
      </dependency>
    
      <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.6</version>
        <scope>provided</scope>
      </dependency>
    </dependencies>
    

2\. web.xml 配置开启 Spring、Spring MVC，同时设置字符编码过滤器，加载静态资源（因为 Spring MVC 会拦截所有请求，导致
JSP 页面中对 JS 和 CSS 的引用也被拦截，配置后可以把对静态资源（JS、CSS、图片等）的请求交给项目的默认拦截器而不是 Spring MVC）。

    
    
    <web-app>
      <display-name>Archetype Created Web Application</display-name>
      <!-- 启动 Spring -->
      <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:applicationContext.xml</param-value>
      </context-param>
      <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
      </listener>
    
      <!-- Spring MVC 的前端控制器，拦截所有请求 -->
      <servlet>
        <servlet-name>mvc-dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
          <param-name>contextConfigLocation</param-name>
          <param-value>classpath:springmvc.xml</param-value>
        </init-param>
      </servlet>
    
      <servlet-mapping>
        <servlet-name>mvc-dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
      </servlet-mapping>
    
      <!-- 字符编码过滤器，一定要放在所有过滤器之前 -->
      <filter>
          <filter-name>CharacterEncodingFilter</filter-name>
          <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
          <init-param>
              <param-name>encoding</param-name>
              <param-value>utf-8</param-value>
          </init-param>
          <init-param>
              <param-name>forceRequestEncoding</param-name>
              <param-value>true</param-value>
          </init-param>
          <init-param>
              <param-name>forceResponseEncoding</param-name>
              <param-value>true</param-value>
          </init-param>
      </filter>
      <filter-mapping>
          <filter-name>CharacterEncodingFilter</filter-name>
          <url-pattern>/*</url-pattern>
      </filter-mapping>
    
      <!-- 加载静态资源 -->
      <servlet-mapping>
          <servlet-name>default</servlet-name>
          <url-pattern>*.js</url-pattern>
      </servlet-mapping>
    
      <servlet-mapping>
          <servlet-name>default</servlet-name>
          <url-pattern>*.css</url-pattern>
      </servlet-mapping>
    </web-app>
    

3\. 在 resources 路径下创建各个框架的配置文件。

![](https://images.gitbook.cn/57c0e100-b6cd-11e9-96e0-d90b4d8f55a3)

  * applicationContext.xml：Spring 的配置文件
  * dbconfig.properties：数据库配置文件
  * mybatis-config.xml：MyBatis 的配置文件
  * springmvc.xml：Spring MVC 的配置文件

我们知道 Spring MVC 本就是 Spring 框架的一个后续产品，所以 Spring MVC 和 Spring 不存在整合，所谓的 SSM
整合实际上是将 MyBatis 和 Spring 进行整合，换句话说，让 Spring 来管理 MyBatis。

4\. applicationContext.xml 中配置 MyBatis 相关信息，以及事务管理。

    
    
    <!-- 加载外部文件 -->
    <context:property-placeholder location="classpath:dbconfig.properties"/>
    
    <!-- 配置 C3P0 数据源 -->
    <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
      <property name="user" value="${jdbc.user}"></property>
      <property name="password" value="${jdbc.password}"></property>
      <property name="driverClass" value="${jdbc.driverClass}"></property>
      <property name="jdbcUrl" value="${jdbc.jdbcUrl}"></property>
      <property name="initialPoolSize" value="5"></property>
      <property name="maxPoolSize" value="10"></property>
    </bean>
    
    <!-- 配置 MyBatis SqlSessionFactory -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
      <!-- 指定 MyBatis 数据源 -->
      <property name="dataSource" ref="dataSource"/>
      <!-- 指定 MyBatis mapper 文件的位置 -->
      <property name="mapperLocations" value="classpath:com/southwind/repository/*.xml"/>
      <!-- 指定 MyBatis 全局配置文件的位置 -->
      <property name="configLocation" value="classpath:mybatis-config.xml"></property>
    </bean>
    
    <!-- 扫描 MyBatis 的 mapper 接口 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
      <!--扫描所有 Repository 接口的实现，加入到 IoC 容器中 -->
      <property name="basePackage" value="com.southwind.repository"/>
    </bean>
    
    <!-- 配置事务管理器 -->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
      <!-- 配置数据源 -->
      <property name="dataSource" ref="dataSource"></property>
    </bean>
    
    <!-- 配置事务增强，事务如何切入  -->
    <tx:advice id="txAdvice" transaction-manager="transactionManager">
      <tx:attributes>
        <!-- 所有方法都是事务方法 -->
        <tx:method name="*"/>
        <!-- 以get开始的所有方法  -->
        <tx:method name="get*" read-only="true"/>
      </tx:attributes>
    </tx:advice>
    
    <!-- 开启基于注解的事务  -->
    <aop:config>
      <!-- 切入点表达式 -->
      <aop:pointcut expression="execution(* com.southwind.service.impl.*.*(..))" id="txPoint"/>
      <!-- 配置事务增强 -->
      <aop:advisor advice-ref="txAdvice" pointcut-ref="txPoint"/>
    </aop:config>
    

5\. dbconfig.properties 配置数据库连接信息。

    
    
    jdbc.jdbcUrl=jdbc:mysql://localhost:3306/ssm?useUnicode=true&characterEncoding=UTF-8
    jdbc.driverClass=com.mysql.jdbc.Driver
    jdbc.user=root
    jdbc.password=root
    

6\. mybatis-config.xml 配置 MyBatis 的相关设置，因为 MyBatis 的大部分配置交给 Spring
来管理了，即在applicationContext.xml 中进行了配置，所以 mybatis-config.xml 只是配置一些辅助性设置，可以省略。

    
    
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
    <configuration>
        <settings>
            <!-- 打印 SQL-->
            <setting name="logImpl" value="STDOUT_LOGGING" />
        </settings>
    
        <typeAliases>
            <!-- 指定一个包名，MyBatis 会在包名下搜索需要的JavaBean-->
            <package name="com.southwind.entity"/>
        </typeAliases>
    
    </configuration>
    

7\. 配置 springmvc.xml。

    
    
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:context="http://www.springframework.org/schema/context"
        xmlns:mvc="http://www.springframework.org/schema/mvc"
        xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
            http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd
            http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-3.2.xsd">
    
        <!-- 告知 Spring，启用 Spring MVC 的注解驱动 -->
        <mvc:annotation-driven />
    
        <!-- 扫描业务代码 -->
        <context:component-scan base-package="com.southwind"></context:component-scan>
    
        <!-- 配置视图解析器 -->
        <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
            <property name="prefix" value="/"></property>
            <property name="suffix" value=".jsp"></property>
        </bean>
    
    </beans>
    

8\. SSM 环境搭建完成，在 MySQL 中创建数据表 department、employee。

    
    
    CREATE TABLE `department` (
      `d_id` int(11) NOT NULL AUTO_INCREMENT,
      `d_name` varchar(255) DEFAULT NULL,
      PRIMARY KEY (`d_id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;
    
    INSERT INTO `department` VALUES ('1', '研发部');
    INSERT INTO `department` VALUES ('2', '销售部');
    INSERT INTO `department` VALUES ('3', '行政部');
    
    
    
    CREATE TABLE `employee` (
      `e_id` int(11) NOT NULL AUTO_INCREMENT,
      `e_name` varchar(255) DEFAULT NULL,
      `e_gender` varchar(255) DEFAULT NULL,
      `e_email` varchar(255) DEFAULT NULL,
      `e_tel` varchar(255) DEFAULT NULL,
      `d_id` int(11) DEFAULT NULL,
      PRIMARY KEY (`e_id`),
      KEY `d_id` (`d_id`),
      CONSTRAINT `employee_ibfk_1` FOREIGN KEY (`d_id`) REFERENCES `department` (`d_id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8;
    
    INSERT INTO `employee` VALUES ('1', '张三', '男', 'zhangsan@163.com', '13567896657', '1');
    INSERT INTO `employee` VALUES ('2', '李四', '男', 'lisi@163.com', '16789556789', '1');
    INSERT INTO `employee` VALUES ('3', '王五', '男', 'wangwu@163.com', '16678906541', '2');
    INSERT INTO `employee` VALUES ('4', '小明', '男', 'xiaoming@163.com', '15678956781', '2');
    INSERT INTO `employee` VALUES ('5', '小红', '女', 'xiaohong@163.com', '13345678765', '3');
    INSERT INTO `employee` VALUES ('6', '小花', '女', 'xiaohua@163.com', '18367654678', '3');
    

9\. 创建实体类 Department、Employee。

    
    
    public class Department {
        private int id;
        private String name;
    }
    
    
    
    public class Employee {
        private int id;
        private String name;
        private String gender;
        private String email;
        private String tel;
        private Department department;
    }
    

10\. 数据库测试数据创建完成，接下来开始写业务代码，首先 Handler。

    
    
    @Controller
    public class EmployeeHandler {
        @Autowired
        private EmployeeService employeeService;
    
        @RequestMapping(value="/queryAll")
        public ModelAndView test(){
            List<Employee> list = employeeService.queryAll();
            ModelAndView modelAndView = new ModelAndView();
            modelAndView.setViewName("index");
            modelAndView.addObject("list", list);
            return modelAndView;
        }
    }
    

11\. Handler 调用 Service，创建 Service 接口及实现类。

    
    
    public interface EmployeeService {
        public List<Employee> queryAll();
    }
    
    
    
    @Service
    public class EmployeeServiceImpl implements EmployeeService{
    
        @Autowired
        private EmployeeRepository employeeRepository;
    
        public List<Employee> queryAll() {
            // TODO Auto-generated method stub
            return employeeRepository.queryAll();
        }
    
    }
    

12\. Service 调用 Repository，创建 Repository 接口，此时没有 Repository 的实现类，使用 MyBatis
框架，在 Repository.xml 中配置实现接口方法需要的 SQL，程序运行时，通过动态代理产生实现接口的代理对象。

    
    
    public interface EmployeeRepository {
        public List<Employee> queryAll();
    }
    
    
    
    <mapper namespace="com.southwind.repository.EmployeeRepository">
    
        <resultMap type="Employee" id="employeeMap">
            <id property="id" column="e_id"/>
            <result property="name" column="e_name"/>
            <result property="gender" column="e_gender"/>
            <result property="email" column="e_email"/>
            <result property="tel" column="e_tel"/>
            <association property="department" javaType="Department">
                <id property="id" column="d_id"/>
                <result property="name" column="d_name"/>
            </association>
        </resultMap> 
    
        <select id="queryAll" resultMap="employeeMap">
            select * from employee e, department d where e.d_id = d.d_id
        </select>
    
    </mapper>
    

13\. 创建 index.jsp，前端使用 Bootstrap 框架。

    
    
    <body>
        <div class="container">
            <!-- 标题 -->
            <div class="row">
                <div class="col-md-12">
                    <h1>SSM-员工管理</h1>
                </div>
            </div>
            <!-- 显示表格数据 -->
            <div class="row">
                <div class="col-md-12">
                    <table class="table table-hover" id="emps_table">
                        <thead>
                            <tr>
                                <th>
                                    <input type="checkbox" id="check_all"/>
                                </th>
                                <th>编号</th>
                                <th>姓名</th>
                                <th>性别</th>
                                <th>电子邮箱</th>
                                <th>联系电话</th>
                                <th>部门</th>
                                <th>操作</th>
                            </tr>
                        </thead>
                        <tbody>
                            <c:forEach items="${list }" var="employee">
                                <tr>
                                    <td><input type='checkbox' class='check_item'/></td>
                                    <td>${employee.id }</td>
                                    <td>${employee.name }</td>
                                    <td>${employee.gender }</td>
                                    <td>${employee.email }</td>
                                    <td>${employee.tel }</td>
                                    <td>${employee.department.name }</td>
                                    <td>
                                        <button class="btn btn-primary btn-sm edit_btn">
                                            <span class="glyphicon glyphicon-pencil">编辑</span>
                                        </button>&nbsp;&nbsp;
                                        <button class="btn btn-danger btn-sm delete_btn">
                                            <span class="glyphicon glyphicon-trash">删除</span>
                                        </button>
                                    </td>
                                </tr>
                            </c:forEach>
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </body>
    

14\. 部署 Tomcat，启动测试，结果如下图所示。

![WX20190618-162938@2x](https://images.gitbook.cn/d3723920-b6cd-11e9-96e0-d90b4d8f55a3)

SSM 框架搭建成功。

**注意**

1\. Handler、Service、Repository 交给 IoC
容器管理，一定要结合配置文件的自动扫描和类定义处的注解完成，对象之间的依赖注入通过 @Autowire 来完成。

    
    
    <!-- 扫描业务代码 -->
    <context:component-scan base-package="com.southwind"></context:component-scan>
    
    
    
    @Controller
    public class EmployeeHandler {
    
        @Autowired
        private EmployeeService employeeService;
    

2\. Repository.xml 的 namspace 与 Repository 接口一定要对应起来，不能写错。

    
    
    <mapper namespace="com.southwind.repository.EmployeeRepository">
    

3\. Repository.xml 中 的 parameterType 和 resultType，或者 resultMap 所对应的类型要与
mybatis-config.xml 中配置的 typeAliases 结合使用，组成对应实体类的全类名，如果 mybatis-config.xml
中没有配置 typeAliases，则 Repository.xml 中直接写实体类的全类名即可。

    
    
    <resultMap type="Employee" id="employeeMap">
    
    
    
    <typeAliases>
        <!-- 指定一个包名，MyBatis 会在包名下搜索需要的 JavaBean-->
        <package name="com.southwind.entity"/>
    </typeAliases>
    

### 总结

本节课我们讲解了 SSM 框架整合的具体步骤，作为一个阶段的小结，我们将前面学习的 Spring、Spring MVC、MyBatis
框架整合起来完成一个小练习，SSM 框架整合开发是当前比较主流的 Java Web 技术栈，是每一个 Java 开发者都必须掌握的基本技术。

### 源码

[请点击这里查看源码](https://github.com/southwind9801/gcssm.git)

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

