第五部分（第 35 ~ 48 篇）内容将为大家详细讲解 Spring 全家桶的重头戏 Spring Boot 核心模块的使用，Spring Boot
作为一个快速构建 Spring 应用的利器，对各种主流框架模块做了很好的集成，开箱即用，这部分内容将为大家详细讲解具体操作。

（1）谈谈你对 Spring Boot 的理解？

**点击查看答案**

Spring Boot 是在 Spring 生态基础上发展而来的，发明 Spring Boot 是为了简化 Spring 的开发。因此说没有 Spring
作为基础，就不会有 Spring Boot，Spring Boot 使用约定优于配置的理念，重新重构了 Spring 的使用，让 Spring
后续的发展更有生命力。Spring Boot 是 Spring 组件提供的一站式处理方案，主要是简化了使用 Spring
的难度，简省了繁重的配置，提供了各种启动器，开发者能快速上手。

    

（2）Spring Boot 的优势是什么？为什么要使用 Spring Boot？

**点击查看答案**

使用 Spring Boot 可以简化开发者搭建 Spring 应用的步骤，让开发者将关注点集中在业务逻辑的处理上，提高开发效率，Spring Boot
的优势：

  * 不需要任何 XML 配置文件
  * 内嵌 Web 服务器，可直接部署
  * 默认支持 JSON 数据，不需要额外配置
  * 支持 RESTful 风格
  * 最少一个配置文件可以配置所有的个性化信息

    

（3）Spring Boot 的配置文件有几种格式？区别是什么？

**点击查看答案**

Properties 和 YAML，它们的区别主要是书写格式不同，如下所示。

  * Properties

    
    
    #端口
    server.port=9090
    #项目访问路径
    server.servlet.context-path=/gcspringboot
    #cookie失效时间，单位为秒
    server.servlet.session.cookie.max-age=100
    #session失效时间，单位为秒
    server.servlet.session.timeout=100
    #请求编码格式
    server.tomcat.uri-encoding=UTF-8
    

  * YAML

    
    
    server:
      port: 8181                      #端口
      servlet:
        context-path: /gcspringboot   #项目访问路径
        session:
          cookie:                     #cookie失效时间，单位为秒
            max-age: 100
          timeout: 100                #session失效时间，单位为秒
      tomcat:
        uri-encoding: UTF-8           #请求编码格式
    

需要注意的是 YAML 格式书写规范非常严格，属性名和属性值之间必须有至少一个空格，即 port: 8181，如果没有空格，写成
port:8181，则启动抛出异常，另外 YAML 格式不支持 @PropertySource 注解导入配置。

    

（4）谈谈你知道的 Spring Boot 核心注解。

**点击查看答案**

Spring Boot 的核心注解是添加在启动类上的 @SpringBootApplication，主要包含了以下 3 个注解：

  * @SpringBootConfiguration：组合了 @Configuration 注解，实现配置文件的功能。
  * @EnableAutoConfiguration：打开自动配置的功能，也可以关闭某个自动配置的选项，如关闭数据源自动配置功能：@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })。
  * @ComponentScan：用来自动扫描被这些注解标识的类，最终生成 IoC 容器里的 bean。

    

（5）如何自动开启 Spring Boot 各个组件？

**点击查看答案**

  * 继承 spring-boot-starter-parent 项目
  * 导入 spring-boot-dependencies 项目依赖

    

（6）Spring Boot 中 starter 的原理是什么？

**点击查看答案**

starter 可以理解为启动器，它包含了一系列可以集成到应用里的依赖包，开发者可以一站式集成 Spring 及其他技术，而不需要手动导入依赖包。如你想使用
Spring Data JPA 访问数据库，只要加入 spring-boot-starter-data-jpa 启动器依赖就可以使用了。

    

（7）Spring Boot 不能使用 XML 配置，这句话对吗?

**点击查看答案**

Spring Boot 推荐使用 Java 配置而非 XML 配置，但是 Spring Boot 中也可以使用 XML 配置，通过
@ImportResource 注解可以引入一个 XML 配置。

    

（8）谈谈你对 Redis 的理解。

**点击查看答案**

Redis 是一个完全开源免费的 key-value 内存数据库，通常被认为是一个数据结构服务器，主要是因为其有着丰富的数据结构
strings、map、list、sets、sorted sets。

  * Redis 使用最佳方式是全部数据 in-memory。
  * Redis 更多场景是作为 Memcached 的替代者来使用。
  * 当需要除 key/value 之外的更多数据类型支持时，使用 Redis 更合适。
  * 当存储的数据不能被剔除时，使用 Redis 更合适。

    

（9）简单说说 Redis 的实现原理。

**点击查看答案**

  * Redis 是一个 key-value 存储系统，和 Memcached 类似，但是解决了断电后数据完全丢失的情况，而且它支持更多的 value 类型，除了 string 外，还支持 lists（链表）、sets（集合）和 zsets（有序集合）几种数据类型。这些数据类型都支持 push/pop、add/remove 及取交集、并集、差集等操作，而且这些操作都具有原子性。
  * Redis 是一种基于客户端-服务端模型以及请求/响应协议的 TCP 服务，这意味着通常情况下一个请求会遵循以下步骤：客户端向服务端发送一个查询请求，并监听 Socket 返回，通常是以阻塞模式，等待服务端响应。服务端处理命令，并将结果返回给客户端。

    

（10）什么是 Thymeleaf？

**点击查看答案**

  * Thymeleaf 是一个支持原生 HTML 文件的 Java 模版引擎，可以实现前后端分离的交互方式，即视图与业务数据分开响应，它可以直接将服务端返回的数据生成 HTML 格式，同时也可以处理 XML、JavaScript、CSS 等格式。
  * Thymeleaf 最大的特点是既可以直接在浏览器打开，就像访问静态页面一样看到样式，也可以结合服务端将业务数据填充进去看到动态生成的页面。Spring Boot 对 Thymeleaf 模版做了很好的集成，在 Spring Boot 应用中使用 Thymeleaf 非常方便。

    

