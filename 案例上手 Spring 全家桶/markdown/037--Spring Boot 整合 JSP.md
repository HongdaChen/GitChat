### 前言

在前面的课程中我们已经用 Spring Boot 快速构建了一个 Web 应用，可以向客户端返回数据，如果是在非前后端分离的传统 Web
项目中，只返回数据是不够的，同时还需要返回视图信息。接下来我们就来一起学习 Spring Boot 与视图层的整合，主要介绍两种视图层技术：JSP 和
Thymeleaf。

JSP 是传统 Java Web 开发中的技术层组件，Thymeleaf 是当前比较流行的技术，我们首先来学习 Spring Boot 如何整合 JSP。

首先对 JSP 的基本概念做一个解释。JSP 全称是 Java Server Page，即 Java 服务页面，是 Java Web
提供的一种动态网页技术，可以在 HTML 代码中插入 Java 程序。其本质是一个 Servlet，当客户端第一次访问 JSP 资源的时候，JSP
引擎会自动为目标 JSP 生成一个对应的 Servlet 文件，在这个 Servlet 文件中，通过 response 将定义在 JSP 中的 HTML
代码返回给客户端，来看一个实际案例，比如 JSP 代码定义如下。

    
    
    <%@ page contentType="text/html;charset=UTF-8" language="java" %>
    <html>
    <head>
        <title>Title</title>
    </head>
    <body>
        <h1>Hello World</h1>
    </body>
    </html>
    

JSP 引擎为其生成的 Servlet 代码如下。

    
    
    response.setContentType("text/html;charset=UTF-8");
    out.write("\n");
    out.write("\n");
    out.write("\n");
    out.write("\n");
    out.write("<html>\n");
    out.write("<head>\n");
    out.write("    <title>Title</title>\n");
    out.write("</head>\n");
    out.write("<body>\n");
    out.write("         <h1>Hello World</h1>");
    out.write("\n");
    out.write("</body>\n");
    out.write("</html>\n");
    

JSP 相当于一个中间层组件，开发者在这个组件中将 Java 代码和 HTML 代码进行整合，由 JSP 引擎将组件转为
Servlet，再把开发者定义在组件中的混合代码翻译成 Servlet 的响应语句，输出给客户端的，这就是 JSP 的底层原理。

虽说只有在第一次访问 JSP 资源的时候才会生成对应的 Servlet，后续访问不再需要重复生成 Servlet，当然如果修改了 JSP
代码，肯定是要更新的。很显然这种方式的效率并不高，因此这也是目前 Java Web 开发逐渐倾向于使用原生的 HTML 作为视图层组件的原因，因为原生
HTML 的效率更高，不需要做中间转换。

了解完 JSP 的底层原理，接下来我们一起学习如何在 Spring Boot 工程中使用 JSP。

1\. 创建基于 Mave 的 Web 工程。

![](https://images.gitbook.cn/3b741510-c02f-11e9-8e2c-3b4fd17ad6da)

2\. 选择 Maven，右侧菜单勾选 Create from archetype，选中
org.apache.maven.archetypes:maven-archetype-webapp，点击 Next。

![](https://images.gitbook.cn/f9dd4c10-bcab-11e9-ac77-f5b1a77a87b3)

3\. 工程创建完成之后，在 pom.xml 中添加相关依赖，tomcat-embed-jasper 是用来加载 JSP 资源的依赖。

    
    
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
        <groupId>org.apache.tomcat.embed</groupId>
        <artifactId>tomcat-embed-jasper</artifactId>
      </dependency>
    </dependencies>
    

4\. 创建 HelloHandler，定义业务方法向客户端返回视图模型数据。

    
    
    @Controller
    public class HelloHandler {
    
        @GetMapping("/index")
        public ModelAndView index(){
            ModelAndView modelAndView = new ModelAndView();
            modelAndView.addObject("name","张三");
            modelAndView.setViewName("index");
            return modelAndView;
        }
    }
    

5\. 在 webapp 路径下创建 JSP 资源 index.jsp，用 EL 表达式取出服务端返回的模型数据，`<%@ page
isELIgnored="false" %>` 指令是设置让 JSP 页面解析 EL 表达式。

    
    
    <%@ page contentType="text/html;charset=UTF-8" language="java" %>
    <%@ page isELIgnored="false" %>
    <html>
    <head>
        <title>Title</title>
    </head>
    <body>
        ${name}
    </body>
    </html>
    

6\. 在 resources 路径下创建配置文件 application.yml。

    
    
    server:
      port: 8080                      #端口
      servlet:
        context-path: /               #项目访问路径
        session:
          cookie:                     #cookie 失效时间，单位为秒
            max-age: 100
          timeout: 100                #session 失效时间，单位为秒
      tomcat:
        uri-encoding: UTF-8           #请求编码格式
    spring:
      mvc:
        view:
          prefix: /                   #视图资源所在路径
          suffix: .jsp                #视图资源的后缀
    

webapp 是 Web 资源的根目录，因为当前的 index.jsp 放置在 webapp 路径下，所以 prefix 的值为 /，如果我们把 JSP
资源放置在 webapp/jsp 路径下，那么 prefix 的值就应该设置为 /jsp/。

7\. 创建启动类 Application。

    
    
    @SpringBootApplication
    public class Application {
        public static void main(String[] args) {
            SpringApplication.run(Application.class,args);
        }
    }
    

整个工程结构如下图所示。

![](https://images.gitbook.cn/28c97940-bcac-11e9-8296-ad04873de5ea)

8\. 启动 Application，打开浏览器访问 http://localhost:8080/index，即可看到如下页面。

![](https://images.gitbook.cn/30d14a50-bcac-11e9-a349-65f0a13339ef)

### 实际应用

上面我们只是实现了最基本的 Spring Boot 整合 JSP，实际开发中我们往往需要结合业务需求进行功能更为丰富的操作，这里我们结合
JSTL（JavaServer Pages Standard Tag Library，JSP 标准标签库）来完成一个数据展示功能。

1\. pom.xml 中添加相关依赖，这里我们引入 JSTL 和 Lombok 依赖。

    
    
    <dependency>
      <groupId>jstl</groupId>
      <artifactId>jstl</artifactId>
      <version>1.2</version>
    </dependency>
    
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
    </dependency>
    

使用 Lombok 可以简化实体类代码的编写工作，常用方法诸如 getter、setter、toString 等都可以由 Lombok
自动生成，开发者不需要自己手动编写，但是使用 Lombok 需要首先在 IDEA 中安装插件，具体安装步骤如下所示。

![WX20190604-142856@2x](https://images.gitbook.cn/73ca37e0-bcac-11e9-b095-45b8601f64cd)

2\. 创建实体类 User。

    
    
    @Data
    @AllArgsConstructor
    public class User {
        private Long id;
        private String name;
        private Integer gender;
    }
    

3\. 在 HelloHandler 中创建业务方法，返回一个 User 对象集合给客户端。

    
    
    @GetMapping("/findAll")
    public ModelAndView findAll(){
      ModelAndView modelAndView = new ModelAndView();
      List<User> list = new ArrayList<>();
      list.add(new User(1L,"张三",1));
      list.add(new User(2L,"李四",0));
      list.add(new User(3L,"王五",1));
      modelAndView.addObject("list",list);
      modelAndView.setViewName("show");
      return modelAndView;
    }
    

4\. 创建 show.jsp，使用 JSTL+EL 表达式取出模型数据。

    
    
    <body>
        <h1>用户信息</h1>
        <table>
            <tr>
                <th>编号</th>
                <th>姓名</th>
                <th>性别</th>
            </tr>
            <c:forEach items="${requestScope.list}" var="user">
                <tr>
                    <td>${user.id}</td>
                    <td>${user.name}</td>
                    <td>
                        <c:if test="${user.gender == 1}">男</c:if>
                        <c:if test="${user.gender == 0}">女</c:if>
                    </td>
                </tr>
            </c:forEach>
        </table>
    </body>
    

5\. 启动 Application，访问 http://localhost:8080/findAll，结果如下所示。

![](https://images.gitbook.cn/81abd670-bcac-11e9-b095-45b8601f64cd)

### 总结

本节课我们讲解了 Spring Boot 整合视图层技术 JSP 的具体操作，在 Java Web 开发中，视图层技术必不可少，当前 Spring Boot
支持的主流视图层解决方案为 JSP 和 Thymeleaf，一个是用传统的 JSP 组件进行开发，另外一个是用原生 HTML 进行开发，下节课我们将对
Spring Boot 整合 Thymeleaf 进行详细讲解。

### 源码

[请点击这里查看源码](https://github.com/southwind9801/gcspringbootjsp.git)

[点击这里获取 Spring Boot
视频专题](https://pan.baidu.com/s/1K2cNTk6JmZa50RYSKwvwGA)，提取码：e4wc

