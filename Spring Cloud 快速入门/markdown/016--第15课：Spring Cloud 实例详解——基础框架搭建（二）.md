接着上一篇，我们继续来搭建基础框架，本文我们将搭建客户端基础模块，集成熔断器，集成持久层框架 MyBatis。

在上一篇我们已经构建好了配置中心，因此，此后搭建的所有工程都是将配置文件放到 Git
上（点击[这里](https://github.com/springcloudlynn/springcloudinactivity)获取本课程配置文件的
Git 仓库地址），通过配置中心将配置文件从 Git 仓库上拉取下来。

### 客户端基础模块

为了便于应用的可读性，我们在顶级工程下，先创建一个 packaging 为 pom 的工程，命名为 client，然后在 client
下创建我们的客户端模块，如图所示：

![enter image description
here](http://images.gitbook.cn/453d6330-754e-11e8-a3f9-f1e4d8a7adbe)

client 的 pom 内容如下：

    
    
    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <parent>
            <artifactId>news</artifactId>
            <groupId>com.lynn</groupId>
            <version>1.0-SNAPSHOT</version>
        </parent>
        <modelVersion>4.0.0</modelVersion>
    
        <artifactId>client</artifactId>
        <description>客户端</description>
        <modules>
            <module>index</module>
            <module>article</module>
            <module>comment</module>
            <module>user</module>
        </modules>
        <packaging>pom</packaging>
        <properties>
            <mybatis.version>1.1.1</mybatis.version>
            <mysql.version>5.1.40</mysql.version>
            <druid.version>1.1.10</druid.version>
        </properties>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-openfeign</artifactId>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-actuator</artifactId>
            </dependency>
            <dependency>
                <groupId>org.mybatis.spring.boot</groupId>
                <artifactId>mybatis-spring-boot-starter</artifactId>
                <version>${mybatis.version}</version>
            </dependency>
            <dependency>
                <groupId>mysql</groupId>
                <artifactId>mysql-connector-java</artifactId>
                <version>${mysql.version}</version>
            </dependency>
            <dependency>
                <groupId>com.alibaba</groupId>
                <artifactId>druid-spring-boot-starter</artifactId>
                <version>${druid.version}</version>
            </dependency>
            <dependency>
                <groupId>com.lynn</groupId>
                <artifactId>common</artifactId>
                <version>1.0-SNAPSHOT</version>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
            </dependency>
        </dependencies>
    </project>
    

接着继续创建客户端工程：index（首页）、article（文章）、comment（评论）、user（用户）。

在每个客户端模块创建启动类，添加以下内容：

    
    
    @SpringCloudApplication
    @ComponentScan(basePackages = "com.lynn")
    public class Application {
    
        public static void main(String[] args) {
            SpringApplication.run(Application.class,args);
        }
    }
    

最后在每个客户端工程下创建 bootstrap.yml 配置文件，在 Git
仓库创建每个客户端模块自己的配置项，并添加相应的内容。接下来，我们具体看下每个客户端下需要添加的代码内容。

  * 首页

首页客户端下 bootstrap.yml 配置文件的代码如下：

    
    
    spring:
      cloud:
        config:
          name: index
          label: master
          discovery:
            enabled: true
            serviceId: config
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8888/eureka/
    

首页配置项 index.yml 中添加如下代码：

    
    
    server:
      port: 8081
    spring:
      application:
        name: index
      profiles:
        active: dev
    

  * 文章

文章客户端下 bootstrap.yml 配置文件的代码如下：

    
    
    spring:
      cloud:
        config:
          name: article
          label: master
          discovery:
            enabled: true
            serviceId: config
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8888/eureka/
    

文章配置项 article.yml 中添加如下代码：

    
    
    server:
      port: 8082
    spring:
      application:
        name: article
      profiles:
        active: dev
    

  * 评论

评论客户端下 bootstrap.yml 配置文件的代码如下：

    
    
    spring:
      cloud:
        config:
          name: comment
          label: master
          discovery:
            enabled: true
            serviceId: config
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8888/eureka/
    

评论配置项 comment.yml 中添加如下代码：

    
    
    server:
      port: 8083
    spring:
      application:
        name: comment
      profiles:
        active: dev
    

  * 用户

用户客户端下 bootstrap.yml 配置文件的代码如下：

    
    
    spring:
      cloud:
        config:
          name: user
          label: master
          discovery:
            enabled: true
            serviceId: config
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8888/eureka/
    

用户配置项 user.yml 中添加如下代码：

    
    
    server:
      port: 8084
    spring:
      application:
        name: user
      profiles:
        active: dev
    

### 熔断器

熔断机制可以有效提升应用的健壮性，通过 Hystrix Dashboard 也可以监控 Feign
调用，便于我们随时观察服务的稳定性，因此集成熔断器是很有必要的，本实例将集成 Feign 和 Hystrix 框架。

首先，在 client 的 pom 中加入依赖（因为所有客户端都需要依赖它，所以在 client 中依赖即可），代码如下：

    
    
     <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-openfeign</artifactId>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-actuator</artifactId>
            </dependency>
    

在每个 client 的 Application 都加入一下注解：

    
    
    @EnableHystrixDashboard
    @EnableFeignClients
    

并加入一下代码：

    
    
    @Bean
        public ServletRegistrationBean getServlet(){
            HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
            ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet );
            registrationBean.setLoadOnStartup(1);
            registrationBean.addUrlMappings("/hystrix.stream");
            registrationBean.setName("HystrixMetricsStreamServlet");
            return registrationBean;
        }
    

我们随便启动一个客户端来看看效果。

依次启动 register、config 和 index，访问地址：http://localhost:8081/hystrix，即可看到如下图所示界面：

![enter image description
here](http://images.gitbook.cn/8adc3c10-7551-11e8-a3f9-f1e4d8a7adbe)

说明我们成功集成了 Hystrix Dashboard。

### 持久层框架 MyBatis

一个 Web 应用免不了数据库的操作，因此我们继续来集成数据库框架，本应用采取 MyBatis 框架，连接池使用阿里巴巴的 Druid 框架。

首先在 client 下加入依赖：

    
    
    <dependency>
                <groupId>org.mybatis.spring.boot</groupId>
                <artifactId>mybatis-spring-boot-starter</artifactId>
                <version>1.1.1</version>
            </dependency>
            <dependency>
                <groupId>mysql</groupId>
                <artifactId>mysql-connector-java</artifactId>
                <version>5.1.40</version>
            </dependency>
            <dependency>
                <groupId>com.alibaba</groupId>
                <artifactId>druid-spring-boot-starter</artifactId>
                <version>1.1.10</version>
            </dependency>
    

然后在 Git 仓库创建配置文件 database.yml：

    
    
    spring:
      datasource:
        druid:
          url: jdbc:mysql://localhost:3306/blog_db?useUnicode=true&characterEncoding=UTF-8&useSSL=false
          username: root
          password: 1qaz2wsx
          stat-view-servlet:
            login-username: admin
            login-password: admin
    

依次启动 register、config 和
index，然后访问：http://localhost:8081/druid，输入配置文件设置的用户名和密码，即可进入如下界面：

![enter image description
here](http://images.gitbook.cn/b54a8570-7555-11e8-9016-addc5ac06bc8)

