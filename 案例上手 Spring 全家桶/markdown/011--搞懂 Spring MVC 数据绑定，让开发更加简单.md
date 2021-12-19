### 前言

这一讲来学习 Spring MVC 的数据绑定，什么是数据绑定？在后台业务方法中，直接获取前端 HTTP 请求中的参数。

首先来了解一下底层原理，HTTP 请求传输的参数都是 String 类型，但是 Hanlder 业务方法中的参数都是我们指定的数据类型，如
int、Object 等，因此需要处理参数的类型转换。此项工作不需要我们开发人员去完成，Spring MVC 的 HandlerAdapter 组件会在执行
Handler 业务方法之前，完成参数的绑定。

了解完大致理论，接下来我们就直接上代码，实践出真知。

### 基本数据类型

以 int 为例，后台需要 int 类型的参数，直接在业务方法定义处添加 int 类型的形参即可，HTTP 请求参数名必须与形参名一致。

@ResponseBody 注解直接返回字符串到客户端，不需要返回 jsp 页面。

    
    
    @RequestMapping(value="/baseType")
    @ResponseBody
    public String baseType(int id){
        return "id:"+id;
    }
    

测试，HTTP 请求不带参数，直接报 500 错误。

![](https://images.gitbook.cn/18a831b0-96f5-11e8-a80f-e9b555a946c4)

错误原因：

![](https://images.gitbook.cn/221eadf0-96f5-11e8-a80f-e9b555a946c4)

可选的参数“id”不能转为 null，因为我们都知道，基本数据类型不能赋值 null。

测试：参数类型为字符串。

![](https://images.gitbook.cn/29e57e60-96f5-11e8-ab3d-7b3c8b8e2dff)

400 错误，错误原因：

![](https://images.gitbook.cn/30d8f940-96f5-11e8-a80f-e9b555a946c4)

String 类型不能转换为 int 类型。

正确使用：

![](https://images.gitbook.cn/3938d560-96f5-11e8-9e0c-8bfb55c56242)

### 包装类

    
    
    @RequestMapping(value="/packageType")
    @ResponseBody
    public String packageType(Integer id){
        return "id:"+id;
    }
    

测试：不传参数。

![](https://images.gitbook.cn/415d7d40-96f5-11e8-ab3d-7b3c8b8e2dff)

没有报错，直接打印 null，因为包装类可以赋值 null。

测试：参数类型为字符串。

![](https://images.gitbook.cn/48fc7a60-96f5-11e8-9e0c-8bfb55c56242)

400 错误，错误原因如下：

![](https://images.gitbook.cn/51463670-96f5-11e8-9f54-b3cc9167c22b)

String 类型不能转换为 Integer 类型。

正确使用：

![](https://images.gitbook.cn/5914cf10-96f5-11e8-9f54-b3cc9167c22b)

参数列表添加 @RequestParam 注解，可以对参数进行相关设置。

    
    
    @RequestMapping(value="/packageType")
    @ResponseBody
    public String packageType(@RequestParam(value="id",required=false,defaultValue="1") Integer id){
        return "id:"+id;
    }
    

  * value="id"：将 HTTP 请求中名为 id 的参数与形参进行映射。
  * required=false：id 参数非必填，可省略。
  * defaultValue="1"：若 HTTP 请求中没有 id 参数，默认值为1。

![](https://images.gitbook.cn/6045a110-96f5-11e8-9e0c-8bfb55c56242)

修改代码：required=true，删除 defaultValue 参数。

    
    
    @RequestMapping(value="/packageType")
    @ResponseBody
    public String packageType(@RequestParam(value="id",required=true) Integer id){
        return "id:"+id;
    }
    

再次运行。

![](https://images.gitbook.cn/67cfddb0-96f5-11e8-ab3d-7b3c8b8e2dff)

报错，因为 id 为必填参数，此时客户端没有传 id 参数，同时业务方法中 id 也没有默认值，所以报错。

若客户端传 id 或者 id 有 dafaultValue，程序不会报错。

### 数组

    
    
    @RequestMapping(value="/arrayType")
    @ResponseBody
    public String arrayType(String[] name){
         StringBuffer sbf = new StringBuffer();
         for(String item:name) {
             sbf.append(item).append(" ");
         }
        return "name:"+sbf.toString();
    }
    

![](https://images.gitbook.cn/73894510-96f5-11e8-a80f-e9b555a946c4)

### POJO

（1）创建 User 类。

    
    
    public class User {
        private String name;
        private int age;
    }
    

（2）JSP 页面 input 标签的 name 值与实体类的属性名对应。

    
    
    <form action="pojoType" method="post">
        姓名：<input type="text" name="name"/><br/>
        年龄：<input type="text" name="age"/><br/>
        <input type="submit" value="提交"/>
    </form>
    

（3）业务方法。

    
    
    @RequestMapping(value="/pojoType")
    @ResponseBody
    public String pojoType(User user){
        return "注册用户信息："+user;
    }
    

（4）运行。

![](https://images.gitbook.cn/7c038700-96f5-11e8-a80f-e9b555a946c4)

![](https://images.gitbook.cn/825088b0-96f5-11e8-ab3d-7b3c8b8e2dff)

处理 @ResponseBody 中文乱码：在 springmvc.xml 中配置消息转换器。

    
    
    <mvc:annotation-driven >
        <!-- 消息转换器 -->
        <mvc:message-converters register-defaults="true">
          <bean class="org.springframework.http.converter.StringHttpMessageConverter">
            <property name="supportedMediaTypes" value="text/html;charset=UTF-8"/>
          </bean>
        </mvc:message-converters>
    </mvc:annotation-driven>
    

### POJO 级联关系

（1）创建 Address 类。

    
    
    public class Address {
        private int id;
        private String name;
    }
    

（2）修改 User 类，添加 Address 属性。

    
    
    public class User {
        private String name;
        private int age;
        private Address address;
    }
    

（3）修改 JSP，添加 address 信息，为 input 标签的 name 设置属性级联，即先关联 User 的 address 属性，再级联
address 的 id 和 name。

    
    
    <form action="pojoType" method="post">
        姓名：<input type="text" name="name"/><br/>
        年龄：<input type="text" name="age"/><br/>
        地址编号：<input type="text" name="address.id"/><br/>
        地址：<input type="text" name="address.name"/><br/>
        <input type="submit" value="提交"/>
    </form>
    

（4）运行

![](https://images.gitbook.cn/8eb30150-96f5-11e8-9e0c-8bfb55c56242)

![](https://images.gitbook.cn/94d4d450-96f5-11e8-ab3d-7b3c8b8e2dff)

### List

Spring MVC 不支持 List 类型的直接转换，需要包装成 Object。

List 的自定义包装类：

    
    
    public class UserList {
        private List<User> users;
    }
    

业务方法：

    
    
    @RequestMapping(value="/listType")
    @ResponseBody
    public String listType(UserList userList){
        StringBuffer sbf = new StringBuffer();
        for(User user:userList.getUsers()){
            sbf.append(user);
        }
        return "用户："+sbf.toString();
    }
    

创建 addList.jsp，同时添加三个用户信息，input 的 name 指向自定义包装类 UserList 中的 users 属性，级联到 name
和 age，同时以下标区分集合中不同的对象。

    
    
    <form action="listType" method="post">
        用户1姓名：<input type="text" name="users[0].name"/><br/>
        用户1年龄：<input type="text" name="users[0].age"/><br/>
        用户2姓名：<input type="text" name="users[1].name"/><br/>
        用户2年龄：<input type="text" name="users[1].age"/><br/>
        用户3姓名：<input type="text" name="users[2].name"/><br/>
        用户3年龄：<input type="text" name="users[2].age"/><br/>
        <input type="submit" value="提交"/>
    </form>
    

执行代码。

![](https://images.gitbook.cn/9fa33f70-96f5-11e8-a80f-e9b555a946c4)

![](https://images.gitbook.cn/afb3f300-96f5-11e8-9f54-b3cc9167c22b)

### Set

和 List 一样，需要封装自定义包装类，将 Set 集合作为属性。不同的是，使用 Set 集合，需要在包装类构造函数中，为 Set 添加初始化对象。

    
    
    public class UserSet {
        private Set<User> users = new HashSet<User>();
    
        public UserSet(){  
            users.add(new User());  
            users.add(new User());  
            users.add(new User());  
        }
    }
    

业务方法：

    
    
    @RequestMapping(value="/setType")
    @ResponseBody
    public String setType(UserSet userSet){
        StringBuffer sbf = new StringBuffer();
        for(User user:userSet.getUsers()){
            sbf.append(user);
        }
        return "用户："+sbf.toString();
    }
    

JSP 用法与 List 一样，input 标签的 name 指向 Set 内对象的属性值，通过下标区分不同的对象。

    
    
    <form action="setType" method="post">
            用户1姓名：<input type="text" name="users[0].name"/><br/>
            用户1年龄：<input type="text" name="users[0].age"/><br/>
            用户2姓名：<input type="text" name="users[1].name"/><br/>
            用户2年龄：<input type="text" name="users[1].age"/><br/>
            用户3姓名：<input type="text" name="users[2].name"/><br/>
            用户3年龄：<input type="text" name="users[2].age"/><br/>
            <input type="submit" value="提交"/>
    </form>
    

执行代码。

![](https://images.gitbook.cn/b8d08f70-96f5-11e8-9e0c-8bfb55c56242)

![](https://images.gitbook.cn/bf4e8c30-96f5-11e8-ab3d-7b3c8b8e2dff)

### Map

自定义包装类：

    
    
    public class UserMap {
        private Map<String,User> users;
    }
    

业务方法，遍历 Map 集合的 key 值，通过 key 值获取 value。

    
    
    @RequestMapping(value="/mapType")
    @ResponseBody
    public String mapType(UserMap userMap){
        StringBuffer sbf = new StringBuffer();
        for(String key:userMap.getUsers().keySet()){
            User user = userMap.getUsers().get(key);
            sbf.append(user);
        }
        return "用户："+sbf.toString();
    }
    

JSP 与 List 和 Set 不同的是，不能通过下标区分不同的对象，改为通过 key 值区分。

    
    
    <form action="mapType" method="post">
        用户1姓名：<input type="text" name="users['a'].name"/><br/>
        用户1年龄：<input type="text" name="users['a'].age"/><br/>
        用户2姓名：<input type="text" name="users['b'].name"/><br/>
        用户2年龄：<input type="text" name="users['b'].age"/><br/>
        用户3姓名：<input type="text" name="users['c'].name"/><br/>
        用户3年龄：<input type="text" name="users['c'].age"/><br/>
        <input type="submit" value="提交"/>
    </form>
    

执行代码。

![](https://images.gitbook.cn/cae6d700-96f5-11e8-a80f-e9b555a946c4)

![](https://images.gitbook.cn/d24b3c20-96f5-11e8-9e0c-8bfb55c56242)

### JSON

JSP：Ajax 请求后台业务方法，并将 JSON 格式的参数传给后台。

    
    
    <script type="text/javascript">
        var user = {
                "name":"张三",
                "age":22
        };
        $.ajax({
            url:"jsonType",
            data:JSON.stringify(user),
            type:"post",
            contentType: "application/json;charse=UTF-8",
            dataType:"text",
            success:function(data){
                var obj = eval("(" + data + ")");
                alert(obj.name+"---"+obj.age);
            }
        })
    </script>
    

#### 注意

  * JSON 数据必须用 JSON.stringify() 方法转换成字符串
  * contentType 不能省略

业务方法：

    
    
    @RequestMapping(value="/jsonType")
    @ResponseBody
    public User jsonType(@RequestBody User user){
        //修改年龄
        user.setAge(user.getAge()+10);
        //返回前端
        return user;
    }
    

#### @RequestBody 注解

读取 HTTP 请求参数，通过 Spring MVC 提供的 HttpMessageConverter 接口将读取的参数转为 JSON、XML
格式的数据，绑定到业务方法的形参。

#### @ResponseBody 注解

将业务方法返回的对象，通过 HttpMessageConverter 接口转为指定格式的数据，JSON、XML等，响应给客户端。

我们使用的是阿里的 FastJson 来取代 Spring 默认的 Jackson 进行数据绑定。FastJson 的优势在于如果属性为空就不会将其转化为
JSON，数据会简洁很多。

#### 如何使用 FastJson

（1）在 pom.xml 中添加 FastJson 依赖。

    
    
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.18</version>
    </dependency> 
    

（2）在 springmvc.xml 中配置 FastJson。

    
    
    <mvc:annotation-driven >
         <!-- 消息转换器 -->
         <mvc:message-converters register-defaults="true">
           <bean class="org.springframework.http.converter.StringHttpMessageConverter">
             <property name="supportedMediaTypes" value="text/html;charset=UTF-8"/>
           </bean>
           <!-- 阿里 fastjson -->
           <bean class="com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter4"/>
         </mvc:message-converters>
    </mvc:annotation-driven>
    

运行代码。

![](https://images.gitbook.cn/dd18bce0-96f5-11e8-9f54-b3cc9167c22b)

前端传给后台的数据 age=22，后台对 age 进行修改，并将修改的结果返回给前端，我们在前端页面看到 age=32，修改成功。

### 总结

本节课我们讲解了 Spring MVC 的数据绑定，数据绑定就是通过框架自动将客户端传来的参数绑定到业务方法的形参中。我们知道 HTTP
请求的参数全部是文本类型的，Spring MVC 框架可以根据 Controller
端业务方法的需求自动将文本类型的参数进行数据类型转换，进而为开发者省去参数转换的繁琐步骤。

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

[请单击这里下载源码](https://github.com/southwind9801/Spring-MVC-DataBind.git)

