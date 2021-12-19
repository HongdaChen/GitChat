### 前言

上节课我们讲解了 Spring Boot 与 Thymeleaf 的整合，以及常用标签的使用，本节课我们继续学习 Thymeleaf 模版标签的使用。

  * th:value

th:value 用作给标签赋值，具体使用如下所示。

Handler

    
    
    @GetMapping("/value")
    public ModelAndView value(){
      ModelAndView modelAndView = new ModelAndView();
      modelAndView.setViewName("test");
      modelAndView.addObject("value","Spring Boot");
      return modelAndView;
    }
    

HTML

    
    
    <input th:value="${value}"/>
    

运行结果如下所示。

![](https://images.gitbook.cn/368fd540-c31b-11e9-a368-8d833bf4550a)

  * th:src

th:src 用作引入静态资源，相当于 HTML 原生标签 img、script 的 src 属性，具体使用如下所示。

首先在工程中添加图片，这里需要注意所有的静态资源，包括图片、JavaScript 资源、CSS 资源，HTML 资源（通过 Handler
访问的除外）等都需要放置在 /resources/static 路径下才可以访问，因为 Spring Boot 默认从 static 路径下读取静态资源。

![](https://images.gitbook.cn/75a86af0-c1d4-11e9-9166-bdb140d6509f)

Handler

    
    
    @GetMapping("/src")
    public ModelAndView src(){
      ModelAndView modelAndView = new ModelAndView();
      modelAndView.setViewName("test");
      modelAndView.addObject("src","/images/springboot.png");
      return modelAndView;
    }
    

HTML

    
    
    <img th:src="${src}"/>
    

运行结果如下所示。

![](https://images.gitbook.cn/83b7a930-c1d4-11e9-9969-976e2ac29eb2)

src 的值可以从模式数据中获取，也可以在 HTML 中直接定义，如果采用这种方式，Handler 中就无需传递业务数据，如下所示。

Handler

    
    
    @GetMapping("/src")
    public ModelAndView src(){
      ModelAndView modelAndView = new ModelAndView();
      modelAndView.setViewName("test");
      return modelAndView;
    }
    

HTML

    
    
    <img th:src="@{../images/springboot.png}">
    

此时的 `th:src="@{../images/springboot.png}"`，注意与 `th:src="${src}"`
的区别，如果是从业务数据中取值，则需要使用 `${}` 取值，如果在静态页面直接取值则使用 `@{}`。

  * th:href

th:href 用作设置超链接的 href，具体使用如下所示。

Handler

    
    
    @GetMapping("/href")
    public ModelAndView href(){
      ModelAndView modelAndView = new ModelAndView();
      modelAndView.setViewName("test");
      modelAndView.addObject("href","https://spring.io/projects/spring-boot/");
      return modelAndView;
    }
    

HTML

    
    
    <a th:href="${href}">Spring Boot</a>
    

运行结果如下所示。

![](https://images.gitbook.cn/03597b40-c31b-11e9-a797-b7290d4ef0b1)

点击 Spring Boot 超链接即可跳转到 Spring Boot 官网。

![](https://images.gitbook.cn/987d0b30-c1d4-11e9-9969-976e2ac29eb2)

这里也可以使用 `<a th:href="@{https://spring.io/projects/spring-boot/}">Spring
Boot</a>`，原理同 th:src，就不再赘述了。

  * th:selected

th:selected 用作给 HTML 元素设置选中，条件成立则选中，否则不选中，具体使用如下所示。

Handler

    
    
    @GetMapping("/selected")
    public ModelAndView selected(){
      List<User> list = new ArrayList<>();
      list.add(new User(1L,"张三",1));
      list.add(new User(2L,"李四",0));
      list.add(new User(3L,"王五",1));
      ModelAndView modelAndView = new ModelAndView();
      modelAndView.setViewName("test");
      modelAndView.addObject("list",list);
      modelAndView.addObject("name","李四");
      return modelAndView;
    }
    

HTML

    
    
    <select>
        <option th:each="user:${list}" th:value="${user.id}" th:text="${user.name}" th:selected="${user.name == name}"></option>
    </select>
    

这里结合 th:each 来使用，首先遍历 list 集合动态创建 option 元素，接下来根据每次遍历出的 user.name 与业务数据中的 name
是否相等来决定是否要选择，业务数据中的 `name = "李四"`，所以 "李四" 对应的 option 为选中状态，运行结果如下所示。

![](https://images.gitbook.cn/b77338c0-c1d4-11e9-9166-bdb140d6509f)

![](https://images.gitbook.cn/c1ee53c0-c1d4-11e9-97a8-35dcf136a505)

  * th:attr

th:attr 用作给 HTML 标签的任意属性赋值，具体使用如下所示。

Handler

    
    
    @GetMapping("/attr")
    public ModelAndView attr(){
      ModelAndView modelAndView = new ModelAndView();
      modelAndView.setViewName("test");
      modelAndView.addObject("attr","Spring Boot");
      return modelAndView;
    }
    

HTML

    
    
    <input th:attr="value=${attr}"/>
    

运行结果如下所示。

![](https://images.gitbook.cn/cf126eb0-c1d4-11e9-97a8-35dcf136a505)

这里也可以使用 `<input th:attr="value=@{Spring Boot}"/>`，原理同 th:src。

### Thymeleaf 对象

Thymeleaf 支持直接访问 Servlet Web 原生资源，即
HttpServletRequest、HttpServletResponse、HttpSession、ServletContext 对象，具体使用如下所示。

  * `#request`：获取 HttpServletRequest 对象
  * `#response`：获取 HttpServletResponse 对象
  * `#session`：获取 HttpSession 对象
  * `#servletContext`：获取 ServletContext 对象

Handler

    
    
    @GetMapping("/servlet")
    public String servlet(HttpServletRequest request){
      request.setAttribute("value","request");
      request.getSession().setAttribute("value","session");
      request.getServletContext().setAttribute("value","servletContext");
      return "test";
    }
    

HTML

    
    
    <p th:text="${#request.getAttribute('value')}"></p>
    <p th:text="${#session.getAttribute('value')}"></p>
    <p th:text="${#servletContext.getAttribute('value')}"></p>
    <p th:text="${#response}"></p>
    

运行结果如下所示。

![](https://images.gitbook.cn/e38eb9c0-c1d4-11e9-8621-c1fbe3716b21)

同时 Thymeleaf 也支持直接访问 session，通过 `${session.name}` 可直接获取，如果使用 ModelAndView
对象来封装视图和业务数据，在视图层业务数据也是保存在 request 对象中的，业务数据可以通过
`${#request.getAttribute('name')}` 获取，也可以通过 `${name}` 获取，具体使用如下所示。

Handler

    
    
    @GetMapping("/servlet2")
    public ModelAndView servlet2(HttpSession session){
      session.setAttribute("name","李四");
      ModelAndView modelAndView = new ModelAndView();
      modelAndView.setViewName("test");
      modelAndView.addObject("name","张三");
      return modelAndView;
    }
    

HTML

    
    
    <p th:text="${name}"></p>
    <p th:text="${#request.getAttribute('name')}"></p>
    <p th:text="${session.name}"></p>
    <p th:text="${#session.getAttribute('name')}"></p>
    

运行结果如下所示。

![](https://images.gitbook.cn/f3258260-c1d4-11e9-97a8-35dcf136a505)

Thymeleaf 除了可以访问 Servlet Web
原生资源，同时还提供了内置对象来简化视图层对于业务数据的处理，可以把内置对象简单理解为工具类，通过相关方法可以实现业务需求，常用的内置对象如下所示。

  * dates：日期格式化内置对象，参照 java.util.Date 的使用。
  * calendars：日期操作内置对象，参照 java.util.Calendar 的使用。
  * numbers： 数字格式化内置对象。
  * strings：字符串格式化内置对象，参照 java.lang.String 的使用。
  * bools：boolean 类型内置对象。
  * arrays：数组操作内置对象，参照 java.utils.Arrays 的使用。
  * lists：List 集合内置对象，参照 java.util.List 的使用。
  * sets：Set 集合内置对象，参照 java.util.Set 的使用。
  * maps：Map 集合内置对象，参照 java.util.Map 的使用。

具体实现如下所示。

Handler

    
    
    @GetMapping("/utility")
    public ModelAndView utility(){
      ModelAndView modelAndView = new ModelAndView();
      modelAndView.setViewName("test");
      modelAndView.addObject("date",new Date());
      Calendar calendar = Calendar.getInstance();
      calendar.set(2019,5,5);
      modelAndView.addObject("calendar",calendar);
      modelAndView.addObject("number",0.06);
      modelAndView.addObject("string","Spring Boot");
      modelAndView.addObject("boolean",true);
      modelAndView.addObject("array", Arrays.asList("张三","李四","王五"));
      List<User> list = new ArrayList<>();
      list.add(new User(1L,"张三",1));
      list.add(new User(2L,"李四",0));
      list.add(new User(3L,"王五",1));
      modelAndView.addObject("list",list);
      Set<User> set = new HashSet<>();
      set.add(new User(1L,"张三",1));
      set.add(new User(2L,"李四",0));
      set.add(new User(3L,"王五",1));
      modelAndView.addObject("set",set);
      Map<Long,User> map = new HashMap<>();
      map.put(1L,new User(1L,"张三",1));
      map.put(2L,new User(2L,"李四",0));
      map.put(3L,new User(3L,"王五",1));
      modelAndView.addObject("map",map);
      return modelAndView;
    }
    

HTML

    
    
    date格式化：<span th:text="${#dates.format(date,'yyyy-MM-dd')}"></span><br/>
    当前日期：<span th:text="${#dates.createToday()}"></span><br/>
    当前时间：<span th:text="${#dates.createNow()}"></span><br/>
    calendar格式化：<span th:text="${#calendars.format(calendar,'yyyy-MM-dd')}"></span><br/>
    number百分比格式化：<span th:text="${#numbers.formatPercent(number,2,2)}"></span><br/>
    name是否为空：<span th:text="${#strings.isEmpty(string)}"></span><br/>
    name的长度：<span th:text="${#strings.length(string)}"></span><br/>
    name拼接：<span th:text="${#strings.concat('I love ',string)}"></span><br/>
    boolean是否为true：<span th:text="${#bools.isTrue(boolean)}"></span><br/>
    arrays的长度：<span th:text="${#arrays.length(array)}"></span><br/>
    arrays是否包含张三：<span th:text="${#arrays.contains(array,'张三')}"></span><br/>
    List是否为空：<span th:text="${#lists.isEmpty(list)}"></span><br/>
    List的长度：<span th:text="${#lists.size(list)}"></span><br/>
    Set是否为空：<span th:text="${#sets.isEmpty(set)}"></span><br/>
    Set的长度：<span th:text="${#sets.size(set)}"></span><br/>
    Map是否为空：<span th:text="${#maps.isEmpty(map)}"></span><br/>
    Map的长度：<span th:text="${#maps.size(map)}"></span><br/>
    

运行结果如下所示。

![](https://images.gitbook.cn/1b04b940-c1d5-11e9-8621-c1fbe3716b21)

### 总结

本节课我们讲解了 Thymeleaf 模版标签的使用，相比于 JSP，Thymeleaf
渲染页面的性能更高，可以提高整个程序的运行效率，同时提供了功能非常强大模版标签，JSP 能实现的各种功能，Thymeleaf
也同样可以实现，所以在实际开发中，建议使用 Thymeleaf 作为视图层解决方案。

[请点击这里查看源码](https://github.com/southwind9801/gcspringbootthymeleaf.git)

[点击这里获取 Spring Boot
视频专题](https://pan.baidu.com/s/1K2cNTk6JmZa50RYSKwvwGA)，提取码：e4wc

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

