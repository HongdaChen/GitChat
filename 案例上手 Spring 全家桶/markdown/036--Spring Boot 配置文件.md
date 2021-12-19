### 前言

上节课我们成功地启动了一个基于 Spring Boot 的 Web 应用，在业务代码编写和使用上与 Spring MVC
几乎一致，但是在配置上省了很多事，比如，不需要在 web.xml 中配置 Spring MVC 的 DispatcherServlet，不需要配置
springmvc.xml 中的自动扫包，视图解析器等，甚至连这个 springmvc.xml 文件都不需要了。同时也不需要开发者手动将项目部署到
Tomcat 中，直接运行 Application 类的 main 方法即可启动一个 Web 应用。

真正做到了简化配置，让开发者可以快速搭建一个基于 Spring
的应用，极大地提高了开发效率，把开发者从繁琐的配置中解放出来，将注意力集中到业务代码的开发上，本节课我们继续深入学习 Spring Boot。

### 自定义 Banner

当我们的 Spring Boot 应用启动成功之后，会在控制台打印 banner，默认是 Spring Boot 的 logo。

![](https://images.gitbook.cn/5c2ac5c0-bcaa-11e9-8296-ad04873de5ea)

如果你想替换成自定义的 banner，Spring Boot 也是支持的，具体操作是在 resources 路径下创建 banner.txt。

![](https://images.gitbook.cn/69b124f0-bcaa-11e9-8296-ad04873de5ea)

并在 banner.txt 中定义 banner 的具体内容，如果希望输出个性化的字体，可以通过第三方网站在线生成，非常简单，访问
<http://patorjk.com/software/taag>，在网页中输入自定义文本，选择字体即可生成个性化内容。

![](https://images.gitbook.cn/7f136330-bcaa-11e9-ac77-f5b1a77a87b3)

将生成好的文本复制到 banner.txt 中，再启动 Application，即可在控制台看到自定义 banner，如下图所示。

![](https://images.gitbook.cn/8b7629f0-bcaa-11e9-ac77-f5b1a77a87b3)

同时我们可以看到，Tomcat 的默认端口是 8080。

![WX20190602-160750@2x](https://images.gitbook.cn/96651560-bcaa-11e9-8296-ad04873de5ea)

如果需要修改端口，可以在 Spring Boot 的配置文件中设置，因此我们要明确一点：使用 Sprint Boot 并不是完全脱离了配置文件，只能说
Spring Boot 将原有的大部分通用配置进行了简化，比如 Spring MVC
中的视图解析器、自动扫包等。如果是一些开发者自定义的配置，还是需要自己手动来配，这种配置也非常简单，Spring Boot
支持两种格式的配置文件，Properties 格式和 YAML 格式。

#### Properties 格式

这种格式大家应该很熟悉，我们在使用 ORMapping 框架的时候可以在 Properties 文件中配置数据源的相关信息，这里也是一样的，首先在
resources 路径下创建一个后缀为 properties 的文件，然后在该文件中添加修改 Tomcat 端口的配置即可，如下所示。

    
    
    server.port=9090
    

![](https://images.gitbook.cn/b36f7880-bcaa-11e9-b095-45b8601f64cd)

重启 Application，看到控制台输出如下内容。

![WX20190603-114527@2x](https://images.gitbook.cn/bcc8cfd0-bcaa-11e9-a349-65f0a13339ef)

可以看到。端口已经成功修改为 9090，除了修改端口，还可以在该文件中添加其他配置，如下所示。

    
    
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
    

注释写的很清楚，我们就不再重复解释了，如果此时设置了 server.servlet.context-path=/gcspringboot，则表示访问项目的
URL 中需要添加 gcspringboot，如果不添加直接访问 http://localhost:9090/index，抛出 404 异常。

![](https://images.gitbook.cn/cf3d15e0-bcaa-11e9-8296-ad04873de5ea)

访问 http://localhost:9090/gcspringboot/index，则请求正常。

![](https://images.gitbook.cn/da96f820-bcaa-11e9-b095-45b8601f64cd)

#### YAML 格式

YAML 是不同于 Properties 的另外一种文本格式，同样可以用来写配置文件，Spring Boot 默认支持 YAML 格式，YAML
格式的优点在于编写简单，结构清晰，利用缩进的形式来表示层级关系。相比于 Properties，可以进一步简化配置文件的编写，使用 YAML 格式，只需要在
resources 路径下创建 application.yml 文件即可。

![](https://images.gitbook.cn/e9c93290-bcaa-11e9-a349-65f0a13339ef)

上述的 Properties 如果用 YAML 格式来编写的话，代码如下所示。

    
    
    server:
      port: 8181                      #端口
      servlet:
        context-path: /gcspringboot   #项目访问路径
        session:
          cookie:                     #cookie 失效时间，单位为秒
            max-age: 100
          timeout: 100                #session 失效时间，单位为秒
      tomcat:
        uri-encoding: UTF-8           #请求编码格式
    

需要注意的是 YAML 格式书写规范非常严格，属性名和属性值之间必须有至少一个空格，即 port:8181，如果没有空格，写成
port:8181，则启动抛出异常，如下所示。

![WX20190603-145205@2x](https://images.gitbook.cn/ff07bc30-bcaa-11e9-8296-ad04873de5ea)

正常启动 Application，可以看到控制台输出如下信息，端口为 8181，表明 YAML 配置文件读取成果。

![WX20190614-100518@2x](https://images.gitbook.cn/250d5110-bcab-11e9-8296-ad04873de5ea)

如果 Properties 和 YAML 两种类型的配置文件同时存在，Spring Boot 会优先读取哪个呢？我们启动
Application，看到控制台输出如下信息。

![WX20190603-114527@2x](https://images.gitbook.cn/325c2580-bcab-11e9-ac77-f5b1a77a87b3)

结果表明，两种类型的配置文件同时存在的情况下，Properties 的优先级更高。

配置文件的位置除了放置在 resources 路径下之外，还有 3 个地方可以放置，如下图所示。

![](https://images.gitbook.cn/3dbce590-bcab-11e9-b095-45b8601f64cd)

这 4 个位置的优先级按上图中标注的 1、2、3、4 依次排列，同时我们也可以直接在 Handler 中读取 YAML
文件中的数据，比如在业务方法中向客户端返回当前服务的端口信息，具体代码如下所示。

    
    
    @RestController
    public class HelloHandler {
    
        @Value("${server.port}")
        private String port;
    
        @GetMapping("/index")
        public String index(){
            return "当前端口是："+this.port;
        }
    }
    

通过 @Value 注解即可自动读取 YAML 文件中的数据，通过 \${server.port} 读取具体的值，启动 Application，访问
http://localhost:8181/gcspringboot/index，看到如下结果。

![](https://images.gitbook.cn/51254fa0-bcab-11e9-8296-ad04873de5ea)

同时 @Value("${server.port}") 也适用于 Properties 配置。

### 总结

本节课我们讲解了 Spring Boot 的两种配置文件格式：Properties 和 YAML，两者的功能是一样的，区别在于具体的编写方式不同，YAML
格式的结构更加清晰，编写也更加简便，实际开发中推荐使用，同时我们还讲解了 Spring Boot 如何自定义 Banner。

[请点击这里查看源码](https://github.com/southwind9801/gcspringboot.git)

[点击这里获取 Spring Boot
视频专题](https://pan.baidu.com/s/1K2cNTk6JmZa50RYSKwvwGA)，提取码：e4wc

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。
>
> 温馨提示：需购买才可入群哦，加小助手微信后需要截已购买的图来验证~

