### Spring MVC 是什么

Spring MVC 是 Spring Framework 用来提供的 Web 组件，全称是 Spring Web MVC，我们习惯将其称之为 Spring
MVC。它是目前主流的实现 MVC 设计模式的框架，提供了前端路由映射、视图解析等功能，让 Web 开发变得更加简单，Spring MVC 是以
Spring IoC 容器为基础的，大大简化了它的配置，并且因为是 Spring 家族成员的一个组件，所以可以和 Spring Framework
无缝衔接，不存在整合的概念，使用起来非常方便，是 Java Web 开发者必须要掌握的框架。

### Spring MVC 能干什么

首先回顾一下 MVC
设计模式，其是一种常用的软件架构方式，将代码分离成三个模块：Controller（控制层）、Model（模型层）、View（视图层），各个模块负责不同的工作，整合实现整体的业务需求。

MVC 的流程：Controller 接受到客户端请求，调用 Model 相关代码完成业务逻辑操作，获取业务数据并返回给
Controller，Controller 再结合 View 完成业务数据的视图层渲染，并将结果响应给客户端，如下图所示。

![enter image description
here](https://images.gitbook.cn/19cf3820-9abc-11e8-b37c-dd4feba3837e)

Spring MVC 对这套 MVC 流程进行了封装，帮助开发者屏蔽了底层代码，并开放出相关接口供开发者调用，让 MVC 开发变得更加简单快捷。

### Spring MVC 实现原理

#### 核心组件

（1）DispatcherServlet：前端控制器，负责调度其他组件的执行，可降低不同组件之间的耦合性，是整个 Spring MVC 的核心模块。

（2）Handler：处理器，完成具体业务逻辑，相当于 Servlet 或 Action。

（3）HandlerMapping：DispatcherServlet 是通过 HandlerMapping 将请求映射到不同的 Handler。

（4）HandlerInterceptor：处理器拦截器，是一个接口，如果我们需要做一些拦截处理，可以来实现这个接口。

（5）HandlerExecutionChain：处理器执行链，包括两部分内容，即 Handler 和
HandlerInterceptor（系统会有一个默认的 HandlerInterceptor，如果需要额外拦截处理，可以添加拦截器设置）。

（6）HandlerAdapter：处理器适配器，Handler 执行业务方法之前，需要进行一系列的操作包括表单数据的验证、数据类型的转换、将表单数据封装到
POJO 等，这一系列的操作，都是由 HandlerAdapter 来完成，DispatcherServlet 通过 HandlerAdapter
执行不同的 Handler。

（7）ModelAndView：装载了模型数据和视图信息，作为 Handler 的处理结果，返回给 DispatcherServlet。

（8）ViewResolver：视图解析器，DispatcherServlet 通过它将逻辑视图解析成物理视图，最终将渲染结果响应给客户端。

以上就是 Spring MVC 的核心组件，那么这些组件之间是如何进行交互的呢？

#### 工作流程

（1）客户端请求被 DispatcherServlet（前端控制器）接收。

（2）根据 Handler Mapping 映射到 Handler。

（3）生成 Handler 和 HandlerInterceptor（如果有则生成）。

（4）Handler 和 HandlerInterceptor 以 HandlerExecutionChain 的形式一并返回给
DispatcherServlet。

（5）DispatcherServlet 通过 HandlerAdapter 调用 Handler 的方法做业务逻辑处理。

（6）返回一个 ModelAndView 对象给 DispatcherServlet。

（7）DispatcherServlet 将获取的 ModelAndView 对象传给 ViewResolver 视图解析器，将逻辑视图解析成物理视图
View。

（8）ViewResolver 返回一个 View 给 DispatcherServlet。

（9）DispatcherServlet 根据 View 进行视图渲染（将模型数据填充到视图中）。

（10）DispatcherServlet 将渲染后的视图响应给客户端。

![enter image description
here](https://images.gitbook.cn/27ce22b0-9abc-11e8-831e-0180aea56660)

#### 如何使用

看到上面的实现原理，大家可能会有这样的担心，Spring MVC 如此众多的组件，开发起来一定很麻烦吧？答案是否定的，Spring MVC
使用起来非常简单，很多组件都由框架提供，作为开发者我们直接使用即可，并不需要自己手动编写代码，真正需要我们开发者进行编写的组件只有两个，即
Handler（处理业务逻辑）和 View（JSP 做展示）。

Spring MVC 的具体使用如下所示。

（1）环境搭建

创建 Maven 工程，创建 pom.xml 配置 Spring MVC 依赖。

    
    
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-webmvc</artifactId>
      <version>4.3.1.RELEASE</version>
    </dependency>
    

（2）在 web.xml 中配置 Spring MVC 的 DispatcherServlet。

    
    
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
             <param-name>contextConfigLocation</param-name>
             <!-- 指定springmvc.xml的路径 -->
             <param-value>classpath:springmvc.xml</param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping> 
    

（3）创建 springmvc.xml 配置文件。

    
    
    <!-- 配置自动扫描 -->
    <context:component-scan base-package="com.southwind.handler"></context:component-scan>
    <!-- 配置视图解析器 -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <!-- 前缀 -->
        <property name="prefix" value="/"></property>
        <!-- 后缀 -->
        <property name="suffix" value=".jsp"></property>
    </bean>
    

自动扫描的类结合注解交给 IoC 容器来管理，视图解析器是 Spring MVC 底层的类，开发者只需要进行配置即可完成 JSP
页面的跳转，如何配置很简单，记住一条：目标资源路径 = 前缀 + 返回值 + 后缀。

比如 DispatcherServlet 返回 index，配置文件中前缀是 /，后缀是 .jsp，代入上述公式：

目标资源路径 = /index.jsp，一目了然，不需要再解释。

（4）创建 Handler 类。

    
    
    @Controller
    public class HelloHandler {
    
        @RequestMapping("/index")
        public String index(){
            System.out.println("执行index业务代码");
            return "index";
        }
    
    }
    

直接在业务方法出添加注解 @RequestMapping("/index")，将 URL 请求 index 直接与 index
业务方法映射，使用起来非常方便。

（5）启动 Tomcat，打开浏览器输入 URL 进行测试。

![](https://images.gitbook.cn/387aa160-96e8-11e8-9f54-b3cc9167c22b)

![](https://images.gitbook.cn/444a01c0-96e8-11e8-9f54-b3cc9167c22b)

#### 流程梳理

（1）DispatcherServlet 接收到 URL 请求 index，结合 @RequestMapping("/index") 注解将该请求交给
index 业务方法。

（2）执行 index 业务方法，控制打印日志，并返回 "index" 字符串。

（3）结合 springmvc.xml 中的视图解析器配置，找到目标资源：/index.jsp，即根目录的 index.jsp，将该 JSP
资源返回客户端完成响应。

Spring MVC 环境搭建成功。

### 总结

本讲我们介绍了 Spring MVC 的基本概念、工作流程以及如何快速搭建环境。作为整个 Spring MVC 课程的入门篇，先对 Spring MVC
有一个基本的认识和了解，在后续的内容中会详细讲解 Spring MVC 各个组件的具体使用。

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

[请单击这里下载源码](https://github.com/southwind9801/springmvc-1.git)

