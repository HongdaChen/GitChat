### 什么是 Spring Boot

Spring Boot 是由 Pivotal 团队提供的基于 Spring 的全新框架，其设计目的是为了简化 Spring
应用的搭建和开发过程。该框架遵循“约定大于配置”原则，采用特定的方式进行配置，从而使开发者无需定义大量的 XML 配置。通过这种方式，Spring Boot
致力于在蓬勃发展的快速应用开发领域成为领导者。

Spring Boot 并不重复造轮子，而且在原有 Spring 的框架基础上封装了一层，并且它集成了一些类库，用于简化开发。换句话说，Spring
Boot 就是一个大容器。

下面几张图展示了[官网](http://projects.spring.io/spring-boot/)上提供的 Spring Boot 所集成的所有类库：

![这里写图片描述](http://images.gitbook.cn/a4bfe2f0-5353-11e8-aed0-2dd5314cde1c)

![这里写图片描述](http://images.gitbook.cn/ab6301a0-5358-11e8-b3f7-510ebd62a866)

![这里写图片描述](http://images.gitbook.cn/bd289120-5358-11e8-ab64-81de3901ec9a)

Spring Boot 官方推荐使用 Maven 或 Gradle 来构建项目，本教程采用 Maven。

### 第一个 Spring Boot 项目

大多数教程都是以 Hello World 入门，本教程也不例外，接下来，我们就来搭建一个最简单的 Spring Boot 项目。

1.创建一个 Maven 工程，请看下图：

![这里写图片描述](http://images.gitbook.cn/d07f9610-5358-11e8-aed0-2dd5314cde1c)

2.在 pom.xml 加入 Spring Boot 依赖：

    
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.3.RELEASE</version>
    </parent>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
    

3.创建应用程序启动类 DemoApplication，并编写以下代码：

    
    
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    
    @SpringBootApplication
    public class DemoApplication {
    
        public static void main(String[] args) {
            SpringApplication.run(DemoApplication.class, args);
        }
    }
    

4.创建一个 Controller 类 HelloController，用以测试我们的第一个基于 Spring Boot 的 Web 应用：

```java  
import org.springframework.web.bind.annotation.RequestMapping; import
org.springframework.web.bind.annotation.RestController;

@RestController public class HelloController {

    
    
    @RequestMapping("hello")
    String hello() {
        return "Hello World!";
    }
    

}

    
    
    5.运行 DemoApplication 类中的 main 方法，看到下图所示内容说明应用启动成功：
    
    ![enter image description here](https://images.gitbook.cn/6ea65570-3b5e-11e9-9e5d-2bbeaaa8d636)
    
    6.浏览器访问：http://localhost:8080/hello，则会看到下图所示界面：
    
    ![这里写图片描述](http://images.gitbook.cn/224a5c50-535e-11e8-b3f7-510ebd62a866)
    
    我们可以注意到，没有写任何的配置文件，更没有显示使用任何容器，只需要启动 Main 方法即可开启 Web 服务，从而访问到 HelloController 类里定义的路由地址。
    
    这就是 Spring Boot 的强大之处，它默认集成了 Tomcat 容器，通过 Main 方法编写的 SpringApplication.run 方法即可启动内置 Tomcat。
    
    它是如何启动的，内部又是如何运行的呢？具体原理我将在《第03课：Spring Boot 启动原理》一节中具体分析。
    
    在上面的示例中，我们没有定义应用程序启动端口，可以看到控制台，它开启了 8080 端口，这是 Spring Boot 默认的启动端口。Spring Boot 提供了默认的配置，我们也可以改变这些配置，具体方法将在后面介绍。
    
    我们在启动类里加入 @SpringBootApplication  注解，则这个类就是整个应用程序的启动类。如果不加这个注解，启动程序将会报错，读者可以尝试一下。
    
    Spring Boot 还有一个特点是利用注解代码繁琐的 XML 配置，整个应用程序只有一个入口配置文件，那就是 application.yml 或 application.properties。接下来，我将介绍其配置文件的用法。
    
    ### properties 和 yaml
    
    在前面的示例代码中，我们并没有看到该配置文件，那是因为 Spring Boot 对每个配置项都有默认值。当然，我们也可以添加配置文件，用以覆盖其默认值，这里以 .properties 文件为例，首先在 resources 下新建一个名为 application.properties（注意：文件名必须是 application）的文件，键入内容为：
    

properties server.port=8081 server.servlet.context-path=/api

    
    
    并且启动 Main 方法，这时程序请求地址则变成了：http://localhost:8081/api/hello。
    
    Spring Boot 支持 properties 和 yaml 两种格式的文件，文件名分别对应 application.properties 和 application.yml，下面贴出 yaml 文件格式供大家参考：
    

yaml server: port: 8080 servlet: context-path: /api

    
    
    可以看出 properties 是以逗号隔开，而 yaml 则换行+ 两个空格 隔开，这里需要注意的是冒号后面必须空格，否则会报错。yaml 文件格式更清晰，更易读，这里作者建议大家都采用 yaml 文件来配置。
    
    以上示例只是小试牛刀，更多的配置将在后面的课程中讲解。
    
    本教程的所有配置均采用 yaml 文件。
    
    ### 打包、运行
    
    Spring Boot 打包分为 war 和 jar 两个格式，下面将分别演示如何构建这两种格式的启动包。
    
    在 pom.xml 加入如下配置：
    

xml war api src/main/resources true org.springframework.boot spring-boot-
maven-plugin maven-resources-plugin 2.5 UTF-8 org.apache.maven.plugins maven-
surefire-plugin 2.18.1 true

    
    
    这个时候运行 mvn package 就会生成 war 包，然后放到 Tomcat 当中就能启动，但是我们单纯这样配置在 Tomcat 是不能成功运行的，会报错，需要通过编码指定 Tomcat 容器启动，修改 DemoApplication 类：
    

java import org.springframework.boot.SpringApplication; import
org.springframework.boot.autoconfigure.SpringBootApplication; import
org.springframework.boot.builder.SpringApplicationBuilder; import
org.springframework.boot.web.servlet.support.SpringBootServletInitializer;

@SpringBootApplication public class DemoApplication extends
SpringBootServletInitializer {

    
    
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
    
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(DemoApplication.class);
    }
    

}

    
    
    这时再打包放到 Tomcat，启动就不会报错了。
    
    在上述代码中，DemoApplication 类继承了 SpringBootServletInitializer，并重写 configure 方法，目的是告诉外部 Tomcat，启动时执行该方法，然后在该方法体内指定应用程序入口为 DemoApplication 类，如果通过外部 Tomcat 启动 Spring Boot 应用，则其配置文件设置的端口和 contextPath 是无效的。这时，应用程序的启动端口即是 Tomcat 的启动端口，contextPath 和 war 包的文件名相同。
    
    接下来我们继续看如果打成 jar 包，在 pom.xml 加入如下配置：
    

xml

jar api src/main/resources true org.springframework.boot spring-boot-maven-
plugin true com.lynn.DemoApplication repackage maven-resources-plugin 2.5
UTF-8 true org.apache.maven.plugins maven-surefire-plugin 2.18.1 true
org.apache.maven.plugins maven-compiler-plugin 2.3.2 1.8 1.8

    
    
    然后通过 mvn package 打包，最后通过 java 命令启动：
    

shell java -jar api.jar ```

如果是 Linux 服务器，上述命令是前台进程，点击 Ctrl+C 进程就会停止，可以考虑用 nohup 命令开启守护进程，这样应用程序才不会自动停止。

这样，最简单的 Spring Boot 就完成了，但是对于一个大型项目，这是远远不够的，Spring Boot
的详细操作可以参照[官网](https://docs.spring.io/spring-
boot/docs/2.0.1.RELEASE/reference/htmlsingle/)。

下面展示一个最基础的企业级 Spring Boot 项目的结构：

![这里写图片描述](http://images.gitbook.cn/3d0be0e0-535e-11e8-ab64-81de3901ec9a)

其中，Application.java 是程序的启动类，Startup.java 是程序启动完成前执行的类，WebConfig.java 是配置类，所有
Bean 注入、配置、拦截器注入等都放在这个类里面。

以上实例只是最简单的 Spring Boot 项目入门实例，后面会深入研究 Spring Boot。

> [点击了解《Spring Cloud
> 快速入门》，解决更多实际问题](https://gitbook.cn/gitchat/column/5af108d20a989b69c385f47a?utm_source=lysd001)。

