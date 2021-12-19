### 前言

前面的课程我们学习了本地 Config Server 的搭建方式，本节课我们一起学习远程 Config Server
的环境搭建，即将各个微服务的配置文件放置在远程 Git 仓库中，通过 Config Server 进行统一管理，本课程中我们使用基于 Git
的第三方代码托管远程仓库 GitHub 作为远程仓库，实际开发中也可以使用 Gitee、SVN 或者自己搭建的私服作为远程仓库，Config Server
结构如下图所示。

![17](https://images.gitbook.cn/0e7b26e0-d79d-11e9-8797-4924c0d7c082)

接下来我们就来一起搭建远程 Config Server。

### GitHub 远程配置文件

首先将配置文件上传到 GitHub 仓库。

1\. 在父工程下创建文件夹 config，config 中创建 configclient.yml。

![](https://images.gitbook.cn/246fd8c0-d7ab-11e9-8797-4924c0d7c082)

2\. configclient.yml 中配置客户端相关信息。

    
    
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8761/eureka/
    server:
      port: 8070
    spring:
      application:
        name: configclient
    management:
      security:
        enabled: false
    

3\. 将 config 上传至 GitHub，作为远程配置文件。

![2](https://images.gitbook.cn/2d82ff50-d7ab-11e9-a536-c512dee3d564)

### 创建 Config Server

1\. 在父工程下创建 Module。

![](https://images.gitbook.cn/356a5010-d7ab-11e9-a536-c512dee3d564)

2\. 输入 ArtifactId，点击 Next。

![](https://images.gitbook.cn/3e28ee00-d7ab-11e9-ad2d-e1c058c00235)

3\. 设置工程名和工程存放路径，点击 Finish。

![](https://images.gitbook.cn/4690b960-d7ab-11e9-ad2d-e1c058c00235)

4\. 在 pom.xml 中添加 Spring Cloud Config 依赖。

    
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
    </dependencies>
    

5\. 在 resources 路径下创建配置文件 application.yml，添加 Config Server 相关配置。

    
    
    server:
      port: 8888
    spring:
      application:
        name: configserver
      cloud:
        config:
          server:
            git:
              uri: https://github.com/southwind9801/myspringclouddemo.git
              searchPaths: config
              username: root
              password: root
          label: master
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8761/eureka/
    

属性说明：

  * server.port：当前 Config Server 服务端口。
  * spring.application.name：当前服务注册在 Eureka Server 上的名称。
  * cloud.config.server.git：Git 仓库配置文件信息。
  * uri：Git Repository 地址。
  * searchPaths：配置文件路径。
  * username：访问 Git Repository 的用户名。
  * password：访问 Git Repository 的密码。
  * cloud.config.server：Git Repository 的分支。
  * eureka.client.service-url.defaultZone：注册中心的访问地址。

6\. 在 java 路径下创建启动类 ConfigServerApplication。

    
    
    @SpringBootApplication
    @EnableConfigServer
    public class ConfigServerApplication {
        public static void main(String[] args) {
            SpringApplication.run(ConfigServerApplication.class,args);
        }
    }
    

注解说明：

  * @SpringBootApplication：声明该类是 Spring Boot 服务的入口。
  * @EnableConfigServer：声明配置中心。

远程 Config Server 环境搭建完成，接下来创建 Config Client，读取远程配置中心的配置信息。

### 创建 Config Client

1\. 在父工程下创建 Module。

![](https://images.gitbook.cn/8059f7b0-d7ab-11e9-8fae-816b29059b0c)

2\. 输入 ArtifactId，点击 Next。

![](https://images.gitbook.cn/8db37a80-d7ab-11e9-8fae-816b29059b0c)

3\. 设置工程名和工程存放路径，点击 Finish。

![](https://images.gitbook.cn/949d0a50-d7ab-11e9-8797-4924c0d7c082)

4\. 在 pom.xml 中添加 Eureka Client、Spring Cloud Config 依赖。

    
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
    
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
    </dependencies>
    

5\. 在 resources 路径下新建 bootstrap.yml，配置读取远程配置中心的相关信息。

    
    
    spring:
      cloud:
        config:
          name: configclient
          label: master
          discovery:
            enabled: true
            serviceId: configserver
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8761/eureka/
    

属性说明：

  * spring.cloud.config.name：当前服务注册在 Eureka Server 上的名称，与远程 Git 仓库的配置文件名对应。
  * spring.cloud.config.label：Git Repository 的分支。
  * spring.cloud.config.discovery.enabled：是否开启 Config 服务发现支持。
  * spring.cloud.config.discovery.serviceId：配置中心的名称。
  * eureka.client.service-url.defaultZone：注册中心的访问地址。

6\. 在 java 路径下创建启动类 ConfigClientApplication。

    
    
    @SpringBootApplication
    public class ConfigClientApplication {
        public static void main(String[] args) {
            SpringApplication.run(ConfigClientApplication.class,args);
        }
    }
    

注解说明：

  * @SpringBootApplication：声明该类是 Spring Boot 服务的入口。

7\. 创建 HelloHandler，定义相关业务方法。

    
    
    @RequestMapping("/config")
    @RestController
    public class HelloHandler {
    
        @Value("${server.port}")
        private int port;
    
        @RequestMapping(value = "/index")
        public String index(){
            return "当前端口："+this.port;
        }
    }
    

8\. 依次启动注册中心、configserver、configclient，如下图所示。

![8](https://images.gitbook.cn/b687e6d0-d7ab-11e9-a536-c512dee3d564)

![9](https://images.gitbook.cn/c0826020-d7ab-11e9-a536-c512dee3d564)

通过控制台输出信息可以看到，configclient 已经读取到了 Git 仓库中的配置信息。

9\. 通过 Postman 工具访问 http://localhost:8070/config/index，如下图所示。

![10](https://images.gitbook.cn/c9cf8270-d7ab-11e9-ad2d-e1c058c00235)

### 动态更新

如果此时对远程配置中心的配置文件进行修改，微服务需要重启以读取最新的配置信息，实际运行环境中这种频繁重启服务的方式是需要避免的，我们可以通过动态更新的方式，实现在不重启服务的前提下自动更新配置信息的功能。

动态更新的实现需要借助于 Spring Cloud Bus 来完成，Spring Cloud Bus
是一个轻量级的分布式通信组件，它的原理是将各个微服务组件与消息代理进行连接，当配置文件发生改变时，会自动通知相关微服务组件，从而实现动态更新，具体实现如下。

1\. 修改 Config Server 的 application.yml，添加 RabbitMQ。

    
    
    server:
      port: 8888
    spring:
      application:
        name: configserver
      cloud:
        bus:
          trace:
            enable: true
        config:
          server:
            git:
              uri: https://github.com/southwind9801/myspringclouddemo.git
              searchPaths: config
              username: root
              password: root
          label: master
      rabbitmq:
        host: localhost
        port: 5672
        username: guest
        password: guest
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8761/eureka/
    management:
      endpoints:
        web:
          exposure:
            include: bus-refresh
    

2\. 修改 Config Client 的 pom.xml，添加 actuator、bus-amqp 依赖。

    
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
    
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
    
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
    
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-amqp</artifactId>
        </dependency>
    </dependencies>
    

3\. 修改 Config Client 的 bootstrap.yml，添加 bus-refresh。

    
    
    spring:
      rabbitmq:
        host: localhost
        port: 5672
        username: guest
        password: guest
      cloud:
        config:
          name: configclient
          label: master
          discovery:
            enabled: true
            serviceId: configserver
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8761/eureka/
    management:
      endpoints:
        web:
          exposure:
            include: bus-refresh
    

4\. 修改 HelloHandler，添加 @RefreshScope 注解。

    
    
    @RequestMapping("/config")
    @RestController
    @RefreshScope
    public class HelloHandler {
    
        @Value("${server.port}")
        private int port;
    
        @RequestMapping(value = "/index")
        public String index(){
            return "当前端口："+this.port;
        }
    }
    

5\. 修改 config 中的配置文件，将端口改为 8080，并更新到 GitHub。

![11](https://images.gitbook.cn/e1f4d850-d7ab-11e9-ad2d-e1c058c00235)

6\. 在不重启服务的前提下，实现配置文件的动态更新，启动 RabbitMQ，成功后如下图所示。

![](https://images.gitbook.cn/e91b7120-d7ab-11e9-8797-4924c0d7c082)

7\. 发送 POST 请求，访问 http://localhost:8070/actuator/bus-refresh，如下图所示。

![13](https://images.gitbook.cn/f275b2d0-d7ab-11e9-a536-c512dee3d564)

8\. 这样就实现动态更新了，再来访问 http://localhost:8070/config/index，如下图所示。

![14](https://images.gitbook.cn/fa817b80-d7ab-11e9-8797-4924c0d7c082)

可以看到端口已经更新为 8080。

9\. 设置 GitHub 自动推送更新，添加 Webhooks，如下图所示。

![15](https://images.gitbook.cn/015c8c60-d7ac-11e9-8fae-816b29059b0c)

10\. 在 Payload URL 输入你的服务地址，如 http://localhost:8070/actuator/bus-refresh，注意将
localhost 替换成服务器的外网 IP。

![16](https://images.gitbook.cn/1ae0fd10-d7ac-11e9-ad2d-e1c058c00235)

### 总结

本节课我们讲解了使用 Spring Clound Config 来实现远程配置中心的具体操作，使用 Git
存储配置信息，每次修改配置信息后都需要重启各种微服务，非常麻烦，Spring Cloud 提供了自动刷新的解决方案，在不重启微服务的情况下，通过
RabbitMQ 来完成配置信息的自动更新。

[请点击这里查看源码](https://github.com/southwind9801/myspringclouddemo.git)

