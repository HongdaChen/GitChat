第二部分（第 8 ~ 20 篇）的内容详细讲解了 Spring MVC，包括常用模块的使用以及为大家梳理 Spring MVC 的底层实现原理。

（1）谈谈你对 MVC 的理解。

**点击查看答案**

  * MVC 设计模式是一种常用的软件架构方式：以 Controller（控制层）、Model（模型层）、View（视图层）三个模块分离的形式来组织代码。
  * MVC 工作流程：控制层接受到客户端请求，调用模型层生成业务数据，传递给视图层，将最终的业务数据和视图响应给客户端做展示。

    

（2）什么是 Spring MVC ？简单介绍下你对 Spring MVC 的理解?

**点击查看答案**

MVC 是一种流行的架构方式，通过将 Model、View、Controller 进行分离，把较为复杂的 Web
应用分成逻辑清晰的模块，简化开发、提高效率，方便组内开发人员之间的配合，Spring MVC 就是一个实现 MVC 设计模式的企业级框架，它是 Spring
的一个子模块，可以非常方便地进行整合。

    

（3）Spring MVC 有哪些优点？

**点击查看答案**

  * 它是基于组件技术的，全部的应用对象，无论控制器和视图，还是业务对象之类的都是 Java 组件，并且和 Spring 提供的其他基础结构紧密集成。
  * 不依赖于 Servlet API（目标虽是如此，但是在实现的时候确实是依赖于 Servlet 的）。
  * 可以任意使用各种视图技术，而不仅仅局限于 JSP。
  * 支持各种请求资源的映射策略。
  * 它是易于扩展的。 

    

（4）Spring MVC 的核心组件有哪些？

**点击查看答案**

  * DispatcherServlet：前端控制器，是整个流程控制的核心，控制其他组件的执行，统一调度，降低组件之间的耦合性，相当于总指挥。
  * Handler：处理器，完成具体业务逻辑，相当于 Servlet 或 Action。
  * HandlerMapping：DispatcherServlet 接收到请求之后，通过 HandlerMapping 将不同的请求分发到不同的 Handler。
  * HandlerInterceptor：处理器拦截器，是一个接口，如果我们需要做一些拦截处理，可以来实现这个接口。
  * HandlerExecutionChain：处理器执行链，包括两部分内容：Handler 和 HandlerInterceptor（系统会有一个默认的 HandlerInterceptor，如果需要额外拦截处理，可以添加拦截器设置）。
  * HandlerAdapter：处理器适配器，Handler 执行业务方法之前，需要进行一系列的操作包括表单数据的验证，数据类型的转换，将表单数据封装到 POJO 等，这一系列的操作，都是由 HandlerAdapter 来完成，DispatcherServlet 通过 HandlerAdapter 执行不同的 Handler。
  * ModelAndView：装载了模型数据和视图信息，作为 Handler 的处理结果，返回给 DispatcherServlet。
  * ViewResolver：视图解析器，DispatcherServlet 通过它将逻辑视图解析成物理视图，最终将渲染结果响应给客户端。

    

（5）Spring MVC 的实现流程是什么？

**点击查看答案**

  * 客户端请求被 DispatcherServlet（前端控制器）接收。
  * 根据 HandlerMapping 映射到 Handler。
  * 生成 Handler 和 HandlerInterceptor（如果有则生成）。
  * Handler 和 HandlerInterceptor 以 HandlerExecutionChain 的形式一并返回给 DispatcherServlet。
  * DispatcherServlet 通过 HandlerAdapter 调用 Handler 的方法做业务逻辑处理。
  * 返回一个 ModelAndView 对象给 DispatcherServlet。
  * DispatcherServlet 将获取的 ModelAndView 对象传给 ViewResolver 视图解析器，将逻辑视图解析成物理视图 View。
  * ViewResolver 返回一个 View 给 DispatcherServlet。
  * DispatcherServlet 根据 View 进行视图渲染（将模型数据填充到视图中）。
  * DispatcherServlet 将渲染后的视图响应给客户端。

    

（6）Spring MVC 怎样设定重定向和转发？

**点击查看答案**

  * 在返回值前面加 "forward:" 就可以让结果转发，譬如 "forward:user.do?name=zhangsan"。
  * 在返回值前面加 "redirect:" 就可以让返回值重定向，譬如 "redirect:index.jsp"。

    

（7）如何解决 POST 请求和 GET 请求的中文乱码问题？

**点击查看答案**

1）解决 POST 请求乱码问题。

在 web.xml 中加入：

    
    
      <filter>
      &nbsp;&nbsp;&nbsp;&nbsp;<filter-name>CharacterEncodingFilter</filter-name>
      &nbsp;&nbsp;&nbsp;&nbsp;<filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
      &nbsp;&nbsp;&nbsp;&nbsp;<init-param>
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<param-name>encoding</param-name>
      &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<param-value>utf-8</param-value>
      &nbsp;&nbsp;&nbsp;&nbsp;</init-param>
      </filter>
    
      <filter-mapping>
      &nbsp;&nbsp;&nbsp;&nbsp;<filter-name>CharacterEncodingFilter</filter-name>
      &nbsp;&nbsp;&nbsp;&nbsp;<url-pattern>/*</url-pattern>
      </filter-mapping>
    

2）GET 请求中文参数出现乱码解决方法有两个。

  * 修改 Tomcat 配置文件添加编码与工程编码一致，如下：

    
    
        <ConnectorURIEncoding="utf-8" connectionTimeout="20000" port="8080" protocol="HTTP/1.1" redirectPort="8443"/>
    

3）另外一种方法对参数进行重新编码：

    
    
        String userName = new String(request.getParamter("userName").getBytes("ISO8859-1"),"utf-8");
    

    

（8）@ModelAttribute 注解应该如何使用？

**点击查看答案**

  * 定义一个方法，该方法用来返回要填充到模型数据中的对象。
  * 给该方法添加 @ModelAttribute 注解。
  * 添加 @ModelAttribute 注解的方法，会在 Spring MVC 在调用任何一个业务方法之前被自动调用。

    

（9）说说你对自定义数据类型转换器的理解。

**点击查看答案**

  * 通过自定义数据类型转换器可以根据需求对 HTTP 请求中的参数进行解析，转换成需要的数据类型。具体操作是创建一个 Java 类，实现 org.springframework.core.convert.converter.Converter 接口，这样自定义的 Java 类就具备了转换数据的功能，然后在 convert 方法中完成转换的具体业务流程。
  * 当服务器接收到一个请求之后，Spring MVC 首先将请求分发到数据类型转换器进行格式转换，然后再进入相应的业务方法。

    

（10）使用 Hibernate Validator 注解方式校验 Email 数据格式应该怎么写？

**点击查看答案**

    
    
    @Email(regexp = "^[a-zA-Z0-9_.-]+@[a-zA-Z0-9-]+(\\.[a-zA-Z0-9-]+)*\\.[a-zA-Z0-9]{2,6}$", message = "请输入正确的邮箱格式")
    private String email;
    

    

