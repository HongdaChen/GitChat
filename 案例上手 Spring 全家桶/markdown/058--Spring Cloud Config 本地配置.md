### 前言

在基于微服务的分布式系统中，每个业务模块都可以拆分为一个独立自治的服务，多个请求来协同完成某个需求，在一个具体的业务场景中某个请求可能需要同时调用多个服务来完成，这就存在一个问题，多个微服务所对应的配置项也会非常多，一旦某个微服务进行了修改，则其他服务也需要作出调整，直接在每个微服务中修改对应的配置项是非常麻烦的，改完之后还需要重新部署项目。

微服务是分布式的，但是我们希望可以对所有微服务的配置文件进行集中统一管理，便于部署和维护，Spring Cloud 提供了对应的解决方案，即 Spring
Cloud Config，通过服务端可以为多个客户端提供配置服务。

> [官方文档地址可点击这里查看](https://cloud.spring.io/spring-cloud-
> static/Finchley.RELEASE/single/spring-cloud.html#_spring_cloud_config)。

Spring Cloud Config 可以将配置文件存放在本地，也可以存放在远程 Git 仓库中。拿远程 Git
仓库来说，具体的操作思路是将所有的外部配置文件集中放置在 Git 仓库中，然后创建 Config
Server，通过它来管理所有的配置文件，需要更改某个微服务的配置信息时，只需要在本地进行修改，然后推送到远程 Git 仓库即可，所有的微服务实例都可以通过
Config Server 来读取对应的配置信息。

接下来我们就一起来搭建本地 Config Server。

### 本地文件系统

我们可以将微服务的相关配置文件存储在本地文件中，然后让微服务来读取本地配置文件，具体操作如下所示。

首先创建本地 Config Server。

1\. 在父工程下创建 Module。

![1](https://images.gitbook.cn/faa36b20-d79a-11e9-a536-c512dee3d564)

2\. 输入 ArtifactId，点击 Next。

![2](https://images.gitbook.cn/00e6a8d0-d79b-11e9-8797-4924c0d7c082)

3\. 设置工程名和工程存放路径，点击 Finish。

![3](https://images.gitbook.cn/07e32460-d79b-11e9-ad2d-e1c058c00235)

4\. 在 pom.xml 中添加 Spring Cloud Config 依赖。

    
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
    </dependencies>
    

5\. 在 resources 路径下创建配置文件 application.yml，添加 Config Server 相关配置。

    
    
    server:
      port: 8762
    spring:
      application:
        name: nativeconfigserver
      profiles:
        active: native
      cloud:
        config:
          server:
            native:
              search-locations: classpath:/shared
    

属性说明：

  * server.port：当前 Config Server 服务端口
  * spring.application.name：当前服务注册在 Eureka Server 上的名称
  * profiles.active：配置文件获取方式
  * cloud.config.server.native.search-locations：本地配置文件的存放路径

6\. 在 resources 路径下新建 shared 文件夹，并在此目录下创建本地配置文件 configclient-dev.yml，定义 port 和
foo 信息。

    
    
    server:
      port: 8070
    foo: foo version 1
    

7\. 在 java 路径下创建启动类 NativeConfigServerApplication。

    
    
    @SpringBootApplication
    @EnableConfigServer
    public class NativeConfigServerApplication {
        public static void main(String[] args) {
            SpringApplication.run(NativeConfigServerApplication.class,args);
        }
    }
    

注解说明：

  * @SpringBootApplication：声明该类是 Spring Boot 服务的入口。
  * @EnableConfigServer：声明配置中心。

本地配置中心已经创建完成，接下来创建客户端来读取本地配置中的配置文件。

8\. 在父工程下创建 Module。

![1](https://images.gitbook.cn/a9089c40-d79a-11e9-ad2d-e1c058c00235)

9\. 输入 ArtifactId，点击 Next。

![5](https://images.gitbook.cn/b31230c0-d79a-11e9-a536-c512dee3d564)

10\. 设置工程名和工程存放路径，点击 Finish。

![6](https://images.gitbook.cn/b93e8b10-d79a-11e9-ad2d-e1c058c00235)

11\. 在 pom.xml 中添加 Spring Cloud Config 依赖。

    
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
    </dependencies>
    

12\. 在 resources 路径下创建 bootstrap.yml，配置读取本地配置中心的相关信息。

    
    
    spring:
      application:
        name: configclient
      profiles:
        active: dev
      cloud:
        config:
          uri: http://localhost:8762
          fail-fast: true
    

属性说明：

  * spring.application.name：当前服务注册在 Eureka Server 上的名称。
  * profiles.active：配置文件名，这里需要与当前微服务在 Eureka Server 注册的名称结合起来使用，两个值用 `-` 连接，比如当前微服务的名称是 configclient，profiles.active 的值是 dev，那么就会在本地 Config Server 中查找名为 configclient-dev 的配置文件。
  * cloud.config.uri：本地 Config Server 的访问路径。
  * cloud.config.fail-fast：设置客户端优先判断 config server 获取是否正常，并快速响应失败内容。

13\. 在 java 路径下创建启动类 NativeConfigClientApplication。

    
    
    @SpringBootApplication
    public class NativeConfigClientApplication {
        public static void main(String[] args) {
            SpringApplication.run(NativeConfigClientApplication.class,args);
        }
    }
    

注解说明：

  * @SpringBootApplication：声明该类是 Spring Boot 服务的入口。

14\. 创建 NativeConfigHandler，定义相关业务方法。

    
    
    @RestController
    @RequestMapping("/native")
    public class NativeConfigHandler {
    
        @Value("${server.port}")
        private String port;
    
        @Value("${foo}")
        private String foo;
    
        @GetMapping("/index")
        public String index(){
            return this.port+"-"+this.foo;
        }
    }
    

15\. 依次启动 NativeConfigServer、ConfigClient，如下图所示。

![7](https://images.gitbook.cn/7614f090-d79a-11e9-a536-c512dee3d564)

![8](https://images.gitbook.cn/7cb876b0-d79a-11e9-8797-4924c0d7c082)

16\. 通过 Postman 工具访问 http://localhost:8070/native/index，如下图所示。

![9](https://images.gitbook.cn/82c53b10-d79a-11e9-8fae-816b29059b0c)

读取本地配置成功。

### 总结

本节课我们讲解了使用 Spring Cloud Config 来实现本地配置的具体操作，Spring Cloud Config 包括服务端（Config
Server）和客户端（Config Client），提供了分布式系统外部化配置的功能，下节课我们来学习远程配置中心的搭建方式。

[请点击这里查看源码](https://github.com/southwind9801/myspringclouddemo.git)

