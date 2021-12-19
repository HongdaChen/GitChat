### 前言

这一讲来学习 Spring MVC 对于 RESTful 架构的支持，首先简单了解什么是 RESTful。

RESTful 是当前比较流行的一种互联网软件架构模型，通过统一的规范完成不同终端的数据访问和交互，REST 全称为 Representational
State Transfer，翻译成中文的意思是资源表现层状态转化。

### RESTful 简介

RESTful 的优点是结构清晰、有统一的标准、扩展性好，要想详细了解 RESTful 可以从以下几个关键词入手。

#### Resources

资源指的是网络中的某个具体文件，类型不限，可以是文本、图片、视频、音频、数据流等，总之就是网络中真实存在的一个实体。如何获取它呢？我们可以使用统一资源定位符来找到它，即
URI，每个资源都有一个特定的 URI，就相当于网络中的每个终端都有一个独一无二的 IP 地址一样，通过 URI 就可以锁定具体的资源。

#### Representation

资源表现层，什么意思呢？就是资源的具体展现形式，比如资源是一段文字，那么我们可以使用 TXT 文件来描述，或者 HTML 文件、XML 文件、JSON
数据等都可以。

#### State Transfer

状态转化是指客户端和服务端之间的数据交互，因为 HTTP
请求不能传输数据的状态，所有的状态都保存在服务端，如果客户端希望访问服务端的数据，就需要使其发生状态转换，同时这种状态转换是建立在表现层上的，完成转换就表示资源的表现形式发生了改变。

RESTful 的概念还是比较抽象的，我们可以先简单做一个理解，RESTful 有以下两个特点：

（1）URL 传参更加简洁，如下所示：

  * 非 RESTful 的 URL：http://...../queryUserById?id=1
  * RESTful 的 URL：http://..../queryUserById/1

（2）完成不同终端之间的资源共享，RESTful 提供了一套规范，不同终端之间只需要遵守该规范，就可以实现数据交互。

Restful 具体来讲就是四种表现形式，HTTP 协议中四种请求类型（GET、POST、PUT、DELETE）分别表示四种常规操作，即 CRUD：

  * GET 用来获取资源
  * POST 用来创建资源
  * PUT 用来修改资源
  * DELETE 用来删除资源

这里有一个问题，传统的 Web 开发中，form 表单只支持 GET 与 POST 请求，不支持 DELETE、PUT，如何解决这一问题？通过添加
HiddenHttpMethodFilter 过滤器，即可将 POST 请求转为 PUT 或 DELETE。

### HiddenHttpMethodFilter 的实现原理

HiddenHttpMethodFilter 的实现原理是这样的：检测请求参数中是否包含 _method
这个参数，如果包含则获取其值，判断是哪种操作后完成请求类型的转换，然后继续传递。

**具体步骤**

（1）在 form 表单中添加隐藏域标签，name="_method"，value="PUT/DELETE"。

    
    
    <form action="httpPut" method="post">
          <input type="hidden" name="_method" value="PUT"/>
          <input type="submit" value="修改"/>
    </form>    
    
    <form action="httpDelete" method="post">
          <input type="hidden" name="_method" value="DELETE"/>
          <input type="submit" value="修改"/>
    </form>
    

（2）web.xml 中配置 HiddenHttpMethodFilter。

    
    
    <filter>
        <filter-name>HiddenHttpMethodFilter</filter-name>
        <filter-class>org.springframework.web.filter.HiddenHttpMethodFilter</filter-class>
    </filter>
    
    <filter-mapping>
        <filter-name>HiddenHttpMethodFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    

接下来，通过完成一个课程管理模块，带大家学习如何使用 RESTful 架构进行代码开发。

**需求分析**

  * 添加课程，成功则返回全部课程信息。
  * 查询课程，通过 id 可查询对应课程信息。
  * 修改课程，成功则返回修改之后的全部课程信息。
  * 删除课程，成功则返回删除之后的全部课程信息。

**代码实现**

（1）JSP 页面资源

  * 添加课程：add.jsp
  * 修改课程：edit.jsp
  * 课程信息展示：index.jsp

（2）Course 实体类

    
    
    public class Course {
        private int id;
        private String name;
        private double price;
    }
    

（3）CourseDAO

    
    
    @Repository
    public class CourseDAO {
        private Map<Integer,Course> courses = new HashMap<Integer,Course>();
    
        public void add(Course course){
            courses.put(course.getId(),course);
        }
    
        public Collection<Course> getAll(){
            return courses.values();
        }
    
        public Course getById(int id){
            return courses.get(id);
        }
    
        public void update(Course course){
            courses.put(course.getId(),course);
        }
    
        public void deleteById(int id){
            courses.remove(id);
        }
    }
    

（4）CourseController、@PostMapping、@GetMapping、@PutMapping、@DeleteMapping 分别用来映射
Post、Get、Put、Delete 请求。

    
    
    @Controller
    public class CourseController {
        @Autowired
        private CourseDAO courseDAO;
    
        @PostMapping(value = "/add")
        public String add(Course course){
            courseDAO.add(course);
            return "redirect:/getAll";
        }
    
        @GetMapping(value = "/getAll")
        public ModelAndView getAll(){
            ModelAndView modelAndView = new ModelAndView();
            modelAndView.setViewName("index");
            modelAndView.addObject("courses",courseDAO.getAll());
            return modelAndView;
        }
    
        @GetMapping(value = "/getById/{id}")
        public ModelAndView getById(@PathVariable(value = "id") int id){
            ModelAndView modelAndView = new ModelAndView();
            modelAndView.setViewName("edit");
            modelAndView.addObject("course",courseDAO.getById(id));
            return modelAndView;
        }
    
        @PutMapping(value = "/update")
        public String update(Course course){
            courseDAO.update(course);
            return "redirect:/getAll";
        }
    
        @DeleteMapping(value = "/delete/{id}")
        public String delete(@PathVariable(value = "id")  int id){
            courseDAO.deleteById(id);
            return "redirect:/getAll";
        }
    
    }
    

（5）运行。

添加课程：

![](https://images.gitbook.cn/f9009f20-9a0d-11e8-992f-9dfb28d2b53f)

添加成功：

![](https://images.gitbook.cn/01a7e480-9a0e-11e8-bd2f-43e393597943)

修改课程：

![](https://images.gitbook.cn/0e406140-9a0e-11e8-9724-fb3ff7de9e38)

修改成功：

![](https://images.gitbook.cn/18e4fd40-9a0e-11e8-bd2f-43e393597943)

删除课程：

![](https://images.gitbook.cn/272620f0-9a0e-11e8-992f-9dfb28d2b53f)

删除成功：

![](https://images.gitbook.cn/2b6fb180-9a0e-11e8-bd2f-43e393597943)

### 总结

本节课我们讲解了 RESTful 架构在 Spring MVC 框架中的应用，RESTful
是目前流行的一种互联网软件架构方式，结构清晰、有统一的标准、扩展性好。Spring MVC 对 RESTful
做了很好的支持，通过框架的特定操作方式就可以完成一个 RESTful 架构项目。

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

[请单击这里下载源码](https://github.com/southwind9801/Spring-MVC-RESTful.git)

