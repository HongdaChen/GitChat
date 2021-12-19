### 前言

Spring MVC 框架为开发者提供了功能强大的注解机制，可以帮助我们简化代码的开发，提高开发效率，同时使得程序具备更好的扩展性，这一讲就来详细讲解
Spring MVC 框架中常用注解的具体使用。

### @RequestMapping

Spring MVC 通过 @RequestMapping 注解将 URL 请求与业务方法进行映射，在控制器的类定义处以及方法定义处都可添加
@RequestMapping，在类定义处添加 @RequestMapping 注解，相当于多了一层访问路径。

    
    
    @Controller
    @RequestMapping("/helloHandler")
    public class HelloHandler {
         @RequestMapping(value="hello")
          public String hello(){
              System.out.println("hello world");
              return "index";
          }
    }
    

![](https://images.gitbook.cn/fd16e590-96ea-11e8-a80f-e9b555a946c4)

### @RequestMapping 常用参数

（1）value：指定 URL 请求的实际地址，是 @RequestMapping 的默认值。

    
    
    @RequestMapping("hello")
    public String hello(){
        System.out.println("hello world");
        return "index";
    }
    

等于

    
    
    @RequestMapping(value="hello")
    public String hello(){
        System.out.println("hello world");
        return "index";
    }
    

（2）method：指定请求的 method 类型，包括 GET、POST、PUT、DELETE 等。

    
    
    @RequestMapping(value="/postTest",method=RequestMethod.POST)
    public String postTest(){
        System.out.println("postTest");
        return "index";
    }
    

上述代码表示只有 POST 请求可以访问该方法，若使用 GET 请求访问，直接抛出异常，如下图所示。

![](https://images.gitbook.cn/07be40b0-96eb-11e8-9e0c-8bfb55c56242)

其他几种请求类型同理，我们常用的是 GET 和 POST 请求，在 REST 架构中会使用到 PUT 和 DELETE 请求。

（3）params：指定 request 中必须包含的参数值，否则无法调用该方法。

    
    
    @RequestMapping(value="paramsTest",params={"name","id=10"})
    public String paramsTest(){
        System.out.println("paramsTest");
        return "index";
    }
    

URL 请求中必须包含 name 和 id 两个参数，并且 id 的值必须为 10，才能调用 paramsTest 方法。

![](https://images.gitbook.cn/118dd970-96eb-11e8-9e0c-8bfb55c56242)

![](https://images.gitbook.cn/1a36a570-96eb-11e8-a80f-e9b555a946c4)

![](https://images.gitbook.cn/212d0680-96eb-11e8-9f54-b3cc9167c22b)

上述是 3 种常见的错误，都不满足 request 同时包含 name 和 id 参数，并且 id=10 的条件，因此均无法正常访问业务方法。

### 参数绑定

params 是对 URL 请求的参数进行限制，不满足条件的 URL 无法访问业务方法，这个特性并不是我们开发中常用的技术点，需要用到的是在业务方法中获取
URL 的参数，实现这一操作很简单，两步完成：

（1）在业务方法定义时声明参数列表；

（2）给参数列表添加 @RequestParam 注解。

    
    
    @RequestMapping(value="paramsBind")
    public String paramsBind(@RequestParam("name") String name,@RequestParam("id") int id){
        System.out.println(name);
        int num = id+10;
        System.out.println(num);
        return "index";
    }
    

将 URL 请求的参数 name 和 id 分别映射给形参 name 和 id，同时进行了数据类型的转换，URL 参数都是 String
类型的，Spring MVC 可以自动根据形参的数据类型完成数据类型转换，比如将 id 转换为 int 类型，因此可以看到打印的 num 值为
20，完成了数学运算，说明已经进行了数据类型转换，具体的数据类型转换工作是由 HandlerAdapter 来完成的。

![](https://images.gitbook.cn/2c0869f0-96eb-11e8-a80f-e9b555a946c4)

![](https://images.gitbook.cn/324abd40-96eb-11e8-9e0c-8bfb55c56242)

### Spring MVC 也支持 RESTful 风格的 URL 参数获取

    
    
    @RequestMapping(value="rest/{name}")
    public String restTest(@PathVariable("name") String name){
        System.out.println(name);
        return "index";
    }
    

将参数列表的注解改为 @PathVariable("name") 即可，非常简单。

![](https://images.gitbook.cn/66e64c90-96eb-11e8-9f54-b3cc9167c22b)

![](https://images.gitbook.cn/6e86f760-96eb-11e8-9f54-b3cc9167c22b)

### 映射 Cookie

Spring MVC 通过映射可以直接在业务方法中获取 Cookie 的值。

    
    
    @RequestMapping("/cookieTest")
    public String getCookie(@CookieValue(value="JSESSIONID") String sessionId){
        System.out.println(sessionId);
        return "index";
    }
    

![](https://images.gitbook.cn/8db3c550-96eb-11e8-ab3d-7b3c8b8e2dff)

![](https://images.gitbook.cn/96416830-96eb-11e8-9f54-b3cc9167c22b)

### 使用 POJO 绑定参数

Spring MVC 会根据请求参数名和 POJO 属性名进行匹配，自动为该对象填充属性值，并且支持属性级联，具体操作如下所示。

（1）创建实体类 Address、User 并进行级联设置

    
    
    public class Address {
        private int id;
        private String name;
    }
    
    public class User {
        private int id;
        private String name;
        private Address address;
    }
    

（2）创建 addUser.jsp

    
    
    <form action="addUser" method="post">
        编号：<input type="text" name="id"/><br/>
        姓名：<input type="text" name="name"/><br/>
        地址：<input type="text" name="address.name"/><br/>
        <input type="submit" value="提交"/>
    </form>
    

（3）业务方法

    
    
    @RequestMapping("/addUser")
    public String getPOJO(User user){
        System.out.println(user);
        return "index";
    }
    

（4）运行

![](https://images.gitbook.cn/a3409600-96eb-11e8-9e0c-8bfb55c56242)

![](https://images.gitbook.cn/d0fde020-96eb-11e8-ab3d-7b3c8b8e2dff)

有的读者写到这可能会发现自己的程序中文乱码了，Spring MVC 解决中文乱码很简单，在 web.xml 中添加过滤器即可，如下所示。

    
    
    <filter>  
        <filter-name>encodingFilter</filter-name>  
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>  
        <init-param>  
            <param-name>encoding</param-name>  
            <param-value>UTF-8</param-value>  
        </init-param>  
    </filter>  
    <filter-mapping>  
        <filter-name>encodingFilter</filter-name>  
        <url-pattern>/*</url-pattern>  
    </filter-mapping>
    

### JSP 页面的转发和重定向

Spring MVC 默认以转发的形式响应 JSP，也可以手动进行修改，重定向：

    
    
    @RequestMapping("redirectTest")
    public String redirectTest(){
        return "redirect:/index.jsp";
    }
    

![](https://images.gitbook.cn/ddffcd10-96eb-11e8-a80f-e9b555a946c4)

![](https://images.gitbook.cn/e56b8530-96eb-11e8-a80f-e9b555a946c4)

通过地址栏可以看到，地址改变，实现了重定向跳转。需要注意的是业务方法中，设置重定向不能写逻辑视图，必须写明目标资源的物理路径，如
"redirect:/index.jsp"。

转发：

    
    
    @RequestMapping("forwardTest")
    public String forwardTest(){
        return "forward:/index.jsp";
    }
    

![](https://images.gitbook.cn/f14c71c0-96eb-11e8-a80f-e9b555a946c4)

![](https://images.gitbook.cn/02dc7250-96ec-11e8-a80f-e9b555a946c4)

可以看到请求前后，地址栏没有改变，实现了转发跳转。

### 总结

本讲讲解了 Spring MVC 常用注解的使用方法，主要应用于客户端到服务器端的请求映射，包括请求名称、请求类型、请求参数等。完成了 MVC 设计模式中
V（View）—— C（Controller）的映射，省去了在 XML 文件中繁琐的映射配置。

### 分享交流

我们为本课程付费读者创建了微信交流群，以方便更有针对性地讨论课程相关问题。入群方式请添加小助手的微信号：GitChatty5，并注明「全家桶」。

阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

> 温馨提示：需购买才可入群哦，加小助手微信后需要截已购买的图来验证~

[请单击这里下载源码](https://github.com/southwind9801/Spring-MVC-1-2.git)

