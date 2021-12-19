### 前言

本节课我们来学习服务跟踪，首先来思考一个问题，为什么要有服务跟踪，我们知道一个分布式系统中往往会部署很多个微服务，这些服务彼此之间会相互调用，整个过程就会较为复杂，我们在进行问题排查或者优化的时候工作量就会比较大。如果能准确跟踪每一个网络请求的整个运行流程，获取它在每个微服务上的访问情况、是否有延迟、耗费时间等，这样的话我们分析系统性能，排查解决问题就会容易很多，我们使用
Zipkin 组件来实现服务跟踪。

### 什么是 Zipkin

Zipkin
是一个可以采集并且跟踪分布式系统中请求数据的组件，可以为开发者采集某个请求在多个微服务之间的追踪数据，并以可视化的形式呈现出来，让开发者可以更加直观地了解到请求在各个微服务中所耗费的时间等信息。

ZipKin 组件包括两部分：Zipkin Server 和 Zipkin
Client，服务端用来采集微服务之间的追踪数据，再通过客户端完成数据的生成和展示，Spring Cloud 为服务跟踪提供了解决方案，Spring
Cloud Sleuth 集成了 Zipkin 组件。

接下来我们通过实际代码来完成服务跟踪的实现，首先来实现 Zipkin Server。

1\. 在父工程下创建 Module。

![](https://images.gitbook.cn/e9421680-d7ac-11e9-ad2d-e1c058c00235)

2\. 输入 ArtifactId，点击 Next。

![](https://images.gitbook.cn/efaf9880-d7ac-11e9-8797-4924c0d7c082)

3\. 设置工程名和工程存放路径，点击 Finish。

![](https://images.gitbook.cn/f5ad68c0-d7ac-11e9-8797-4924c0d7c082)

4\. 在 pom.xml 中添加 Zipkin Server 依赖。

    
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>io.zipkin.java</groupId>
            <artifactId>zipkin-server</artifactId>
            <version>2.9.4</version>
        </dependency>
        <dependency>
            <groupId>io.zipkin.java</groupId>
            <artifactId>zipkin-autoconfigure-ui</artifactId>
            <version>2.9.4</version>
        </dependency>
    </dependencies>
    

5\. 在 resources 路径下创建配置文件 application.yml，添加 Zipkin 相关配置。

    
    
    server:
      port: 9090
    

属性说明：

  * server.port：当前 Zipkin Server 服务端口。

6\. 在 java 路径下创建启动类 ZipkinApplication。

    
    
    @SpringBootApplication
    @EnableZipkinServer
    public class ZipkinApplication {
        public static void main(String[] args) {
            SpringApplication.run(ZipkinApplication.class,args);
        }
    }
    

注解说明：

  * @SpringBootApplication：声明该类是 Spring Boot 服务的入口。
  * @EnableZipkinServer：声明启动 Zipkin Server。

Zipkin Server 搭建成功，接下来创建 Zipkin Client。

7\. 在父工程下创建 Module。

![](https://images.gitbook.cn/10a28070-d7ad-11e9-8fae-816b29059b0c)

8\. 输入 ArtifactId，点击 Next。

![](https://images.gitbook.cn/1998d030-d7ad-11e9-8fae-816b29059b0c)

9\. 设置工程名和工程存放路径，点击 Finish。

![](https://images.gitbook.cn/21fc07b0-d7ad-11e9-8797-4924c0d7c082)

10\. 在 pom.xml 中添加 Zipkin 依赖。

    
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zipkin</artifactId>
        </dependency>
    </dependencies>
    

11\. 在 resources 路径下创建配置文件 application.yml，添加 Zipkin 相关配置。

    
    
    server:
      port: 8090
    spring:
      application:
        name: zipkinclient
      sleuth:
        web:
          client:
            enabled: true
        sampler:
          probability: 1.0
      zipkin:
        base-url: http://localhost:9090/
    eureka:
      client:
        service-url:
          defaultZone: http://localhost:8761/eureka/
    

属性说明：

  * server.port：当前 Zipkin Client 服务端口。
  * spring.application.name：当前服务注册在 Eureka Server 上的名称。
  * spring.sleuth.web.client.enabled：设置是否开启 Sleuth。
  * spring.sleuth.sampler.probability：设置采样比例，默认是 0.1.
  * spring.zipkin.base-url：Zipkin Server 地址。
  * eureka.client.service-url.defaultZone：注册中心的访问地址。

12\. 在 java 路径下创建启动类 ZipkinClientApplication。

    
    
    @SpringBootApplication
    public class ZipkinClientApplication {
        public static void main(String[] args) {
            SpringApplication.run(ZipkinClientApplication.class,args);
        }
    }
    

注解说明：

  * @SpringBootApplication：声明该类是 Spring Boot 服务的入口。

13\. 创建 ZipkinHandler，定义相关业务方法。

    
    
    package com.southwind.controller;
    
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;
    
    @RestController
    @RequestMapping("/zipkin")
    public class ZipkinHandler {
    
        @Value("${server.port}")
        private String port;
    
        @GetMapping("/index")
        public String index(){
            return "当前端口："+this.port;
        }
    }
    

14\. 依次启动注册中心、Zipkin、ZipkinClient，如下图所示。

![6](https://images.gitbook.cn/5a8524e0-d7ad-11e9-ad2d-e1c058c00235)

![7](https://images.gitbook.cn/60eba200-d7ad-11e9-a536-c512dee3d564)

15\. 打开浏览器访问 http://localhost:9090/zipkin/，可看到 Zipkin 首页，如下图所示。

![8](https://images.gitbook.cn/670a19a0-d7ad-11e9-a536-c512dee3d564)

16\. 点击 Find Traces 按钮可看到监控数据情况，当前没有监控到任何数据，如下图所。

![9](https://images.gitbook.cn/9a8592f0-d7ad-11e9-8797-4924c0d7c082)

17\. 通过 Postman 访问 http://localhost:8090/zipkin/index，如下图所示。

![10](https://images.gitbook.cn/a1a6d490-d7ad-11e9-a536-c512dee3d564)

18\. 再次刷新 http://localhost:9090/zipkin/，可看到监控数据，如下图所示。

![11](https://images.gitbook.cn/a7fc88d0-d7ad-11e9-8797-4924c0d7c082)

19\. 点击可查看详情，如下图所示。

![12](https://images.gitbook.cn/aede2960-d7ad-11e9-8797-4924c0d7c082)

![13](https://images.gitbook.cn/b4cc4230-d7ad-11e9-ad2d-e1c058c00235)

### 总结

本节课我们讲解了使用 Zipkin
来实现服务链路追踪的具体操作，通过服务跟踪，我们可以追踪到每个网络请求，了解它整个运行流程，经过了哪些微服务、是否有延迟、耗费时间等，在此基础上我们能够更好的分析系统性能，解决系统问题。

[请点击这里查看源码](https://github.com/southwind9801/myspringclouddemo.git)

