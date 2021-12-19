### 前言

前面的课程我们已经实现了注册中心、配置中心以及各种服务提供者，本节课我们来实现服务消费者 clientfeign，完成客户端的相关业务，分别调用服务提供者
account、menu、order、user 的相关服务，并通过 Feign 实现负载均衡。

1\. 在父工程下创建一个 Module ，命名为 clientfeign，pom.xml 添加相关依赖，集成 Feign 和 Thymeleaf
模版相关依赖，配置文件从 Git 仓库拉取，添加配置中心 Spring Cloud Config 相关依赖。

    
    
    <dependencies>
        <!-- eurekaclient -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
            <version>2.0.0.RELEASE</version>
        </dependency>
        <!-- 集成 Feign -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <!-- Thymeleaf 模版 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <!-- 配置中心 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
    </dependencies>
    

2\. 在 resources 目录下创建 bootstrap.yml，在该文件中配置拉取 Git 仓库相关配置文件的信息。

    
    
    spring:
      cloud:
        config:
          name: clientfeign #对应的配置文件名称
          label: master #Git 仓库分支名
          discovery:
            enabled: true
            serviceId: configserver #连接的配置中心名称
    eureka:
      client:
        serviceUrl:
          defaultZone: http://localhost:8761/eureka/
    

3\. 在 java 目录下创建启动类 ClientFeignApplication。

    
    
    @SpringBootApplication
    @EnableFeignClients
    @ServletComponentScan
    public class ClientFeignApplication {
        public static void main(String[] args) {
            SpringApplication.run(ClientFeignApplication.class,args);
        }
    }
    

4\. 在 Git 仓库配置文件 clientfeign.yml 中添加相关信息。

    
    
    server:
      port: 8030
    spring:
      application:
        name: clientfeign
      thymeleaf:
        prefix: classpath:/static/
        suffix: .html
    eureka:
      client:
        service-url:
          defaultZone: http://localhost:8761/eureka/
      instance:
        prefer-ip-address: true
    

5\. 在 java 目录下创建 entity 文件夹，将 account、menu、order、user 中的实体类拷贝过来。

6\. 在 java 目录下创建 feign 文件夹，新建 AccountFeign 接口、MenuFeign 接口、OrderFeign
接口、UserFeign 接口，通过 @FeignClient 注解直接调用服务提供者 account、menu、order、user 的相关服务。

AccountFeign 接口：

    
    
    @FeignClient(value = "account")
    public interface AccountFeign {
    
        @GetMapping("/account/login/{username}/{password}/{type}")
        public Account login(@PathVariable("username") String username, @PathVariable("password") String password, @PathVariable("type") String type);
    }
    

MenuFeign 接口：

    
    
    @FeignClient(value = "menu")
    public interface MenuFeign {
    
        @GetMapping("/menu/findAll/{page}/{limit}")
        public MenuVO findAll(@PathVariable("page") int page, @PathVariable("limit") int limit);
    
        @GetMapping("/menu/findAll")
        public List<Type> findAll();
    
        @PostMapping("/menu/save")
        public void save(@RequestBody Menu menu);
    
        @GetMapping("/menu/findById/{id}")
        public Menu findById(@PathVariable("id") long id);
    
        @PutMapping("/menu/update")
        public void update(@RequestBody Menu menu);
    
        @DeleteMapping("/menu/deleteById/{id}")
        public void deleteById(@PathVariable("id") long id);
    }
    

OrderFeign 接口：

    
    
    @FeignClient(value = "order")
    public interface OrderFeign {
    
        @PostMapping("/order/save")
        public void save(@RequestBody Order order);
    
        @GetMapping("/order/findAllByUid/{uid}/{page}/{limit}")
        public OrderVO findAllByUid(@PathVariable("uid") long uid, @PathVariable("page") int page, @PathVariable("limit") int limit);
    
        @DeleteMapping("/order/deleteByMid/{mid}")
        public void deleteByMid(@PathVariable("mid") long mid);
    
        @DeleteMapping("/order/deleteByUid/{uid}")
        public void deleteByUid(@PathVariable("uid") long uid);
    
        @GetMapping("/order/findAllByState/{state}/{page}/{limit}")
        public OrderVO findAllByState(@PathVariable("state") int state, @PathVariable("page") int page, @PathVariable("limit") int limit);
    
        @PutMapping("/order/updateState/{id}/{state}/{aid}")
        public void updateState(@PathVariable("id") long id, @PathVariable("state") long state,@PathVariable("aid") long aid);
    }
    

UserFeign 接口：

    
    
    @FeignClient(value = "user")
    public interface UserFeign {
    
        @GetMapping("/user/findAll/{page}/{limit}")
        public UserVO findAll(@PathVariable("page") int page, @PathVariable("limit") int limit);
    
        @PostMapping("/user/save")
        public void save(@RequestBody User user);
    
        @DeleteMapping("/user/deleteById/{id}")
        public void deleteById(@PathVariable("id") long id);
    }
    

7\. 在 java 目录下创建 controller 文件夹，新建
AccountHandler、MenuHandler、OrderHandler、UserHandler 类完成相关操作。

AccountHandler：

    
    
    @Controller
    @RequestMapping("/account")
    public class AccountHandler {
    
        @Autowired
        private AccountFeign accountFeign;
    
        @PostMapping("/login")
        public String login(@RequestParam("username") String username, @RequestParam("password") String password, @RequestParam("type") String type, HttpSession session){
            Account account = accountFeign.login(username,password,type);
            String target = null;
            if(account == null){
                target = "login";
            }else{
                switch (type){
                    case "user":
                        User user = convertUser(account);
                        session.setAttribute("user",user);
                        target = "redirect:/account/redirect/index";
                        break;
                    case "admin":
                        Admin admin = convertAdmin(account);
                        session.setAttribute("admin",admin);
                        target = "redirect:/account/redirect/main";
                        break;
                }
            }
            return target;
        }
    
        @GetMapping("/logout")
        public String logout(HttpSession session){
            session.invalidate();
            return "login";
        }
    
        @RequestMapping("/redirect/{target}")
        public String redirect(@PathVariable("target") String target){
            return target;
        }
    
        private User convertUser(Account account){
            User user = new User();
            user.setUsername(ReflectUtils.getFieldValue(account,"username")+"");
            user.setPassword(ReflectUtils.getFieldValue(account,"password")+"");
            user.setGender(ReflectUtils.getFieldValue(account,"gender")+"");
            user.setId((long)(ReflectUtils.getFieldValue(account,"id")));
            user.setNickname(ReflectUtils.getFieldValue(account,"nickname")+"");
            user.setRegisterdate((Date)(ReflectUtils.getFieldValue(account,"registerdate")));
            user.setTelephone(ReflectUtils.getFieldValue(account,"telephone")+"");
            return user;
        }
    
        private Admin convertAdmin(Account account){
            Admin admin = new Admin();
            admin.setUsername(ReflectUtils.getFieldValue(account,"username")+"");
            admin.setPassword(ReflectUtils.getFieldValue(account,"password")+"");
            admin.setId((long)(ReflectUtils.getFieldValue(account,"id")));
            return admin;
        }
    }
    

这里不能直接完成类型转换，因为虚拟机的默认类加载机制是通过双亲委派实现的，Spring Boot 为了实现程序动态性，破坏了双亲委派模型。用户自定义类会被
Spring Boot 自定义的加载器 RestartClassLoader 所截获，一旦发现类路径下有文件的修改，Spring Boot 中的
spring-boot-devtools
会重新生成新的类加载器来加载新的类文件，从而实现动态功能，但是也会带来因类加载器不同导致的转换异常问题，这里我们编写一个工具类
ReflectUtils，通过反射机制手动完成类型转换，ReflectUtils 结合 convertUser 和 convertAdmin 即可完成
Account 类型到 User 及 Admin 类型的转换，ReflectUtils 代码如下。

    
    
    public class ReflectUtils{
        public static Object getFieldValue(Object obj, String fieldName){
            if(obj == null){
                return null ;
            }
            Field targetField = getTargetField(obj.getClass(), fieldName);
    
            try {
                return FieldUtils.readField(targetField, obj, true ) ;
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
            return null ;
        }
    
        public static Field getTargetField(Class<?> targetClass, String fieldName) {
            Field field = null;
    
            try {
                if (targetClass == null) {
                    return field;
                }
    
                if (Object.class.equals(targetClass)) {
                    return field;
                }
    
                field = FieldUtils.getDeclaredField(targetClass, fieldName, true);
                if (field == null) {
                    field = getTargetField(targetClass.getSuperclass(), fieldName);
                }
            } catch (Exception e) {
            }
    
            return field;
        }
    }
    

MenuHandler：

    
    
    @Controller
    @RequestMapping("/menu")
    public class MenuHandler {
    
        @Autowired
        private MenuFeign menuFeign;
        @Autowired
        private OrderFeign orderFeign;
    
        @GetMapping("/findAll")
        @ResponseBody
        public MenuVO findAll(@RequestParam("page") int page, @RequestParam("limit") int limit){
            return menuFeign.findAll(page, limit);
        }
    
        @GetMapping("/prepareSave")
        public String prepareSave(Model model){
            model.addAttribute("list",menuFeign.findAll());
            return "menu_add";
        }
    
        @PostMapping("/save")
        public String save(Menu menu){
            menuFeign.save(menu);
            return "redirect:/account/redirect/menu_manage";
        }
    
        @GetMapping("/findById/{id}")
        public String findById(@PathVariable("id") long id,Model model){
            model.addAttribute("list",menuFeign.findAll());
            model.addAttribute("menu",menuFeign.findById(id));
            return "menu_update";
        }
    
        @PostMapping("/update")
        public String update(Menu menu){
            menuFeign.update(menu);
            return "redirect:/account/redirect/menu_manage";
        }
    
        @GetMapping("/deleteById/{id}")
        public String deleteById(@PathVariable("id") long id){
            orderFeign.deleteByMid(id);
            menuFeign.deleteById(id);
            return "redirect:/account/redirect/menu_manage";
        }
    }
    

OrderHandler：

    
    
    @Controller
    @RequestMapping("/order")
    public class OrderHandler {
    
        @Autowired
        private OrderFeign orderFeign;
    
        @GetMapping("/save/{mid}")
        public String save(@PathVariable("mid") long mid, HttpSession session){
            User user = (User) session.getAttribute("user");
            Order order = new Order();
            Menu menu = new Menu();
            menu.setId(mid);
            order.setUser(user);
            order.setMenu(menu);
            order.setDate(new Date());
            orderFeign.save(order);
            return "redirect:/account/redirect/order";
        }
    
        @GetMapping("/findAllByUid")
        @ResponseBody
        public OrderVO findAllByUid(@RequestParam("page") int page, @RequestParam("limit") int limit,HttpSession session){
            User user = (User) session.getAttribute("user");
            return orderFeign.findAllByUid(user.getId(), page, limit);
        }
    
        @GetMapping("/findAllByState")
        @ResponseBody
        public OrderVO findAllByState(@RequestParam("page") int page, @RequestParam("limit") int limit){
            return orderFeign.findAllByState(0, page, limit);
        }
    
        @GetMapping("/updateState/{id}/{state}")
        public String updateState(@PathVariable("id") long id,@PathVariable("state") int state,HttpSession session){
            Admin admin = (Admin) session.getAttribute("admin");
            orderFeign.updateState(id,state,admin.getId());
            return "redirect:/account/redirect/order_handler";
        }
    }
    

UserHandler：

    
    
    @Controller
    @RequestMapping("/user")
    public class UserHandler {
    
        @Autowired
        private UserFeign userFeign;
        @Autowired
        private OrderFeign orderFeign;
    
        @GetMapping("/findAll")
        @ResponseBody
        public UserVO findAll(@RequestParam("page") int page, @RequestParam("limit") int limit){
            return userFeign.findAll(page, limit);
        }
    
        @PostMapping("/save")
        public String save(User user){
            userFeign.save(user);
            return "redirect:/account/redirect/user_manage";
        }
    
        @GetMapping("/deleteById/{id}")
        public String deleteById(@PathVariable("id") long id){
            orderFeign.deleteByUid(id);
            userFeign.deleteById(id);
            return "redirect:/account/redirect/user_manage";
        }
    }
    

8\. 添加过滤器，防止用户在未登录的状态下访问资源页面，在 java 目录下创建 filter 文件夹，新建 AdminFilter 和
UserFilter，实现登录过滤的业务逻辑。

AdminFilter：

    
    
    @Component
    @WebFilter(urlPatterns = {"/main.html","/account/redirect/main"},filterName = "adminFilter")
    public class AdminFilter implements Filter {
        @Override
        public void init(FilterConfig filterConfig) throws ServletException {
    
        }
    
        @Override
        public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
            HttpServletRequest request = (HttpServletRequest) servletRequest;
            HttpServletResponse response = (HttpServletResponse) servletResponse;
            HttpSession session = request.getSession();
            Admin admin = (Admin) session.getAttribute("admin");
            if(admin == null){
                response.sendRedirect("login.html");
            }else{
                filterChain.doFilter(servletRequest,servletResponse);
            }
        }
    
        @Override
        public void destroy() {
    
        }
    }
    

UserFilter：

    
    
    @Component
    @WebFilter(urlPatterns = {"/index.html","/account/redirect/index","/order.html","/account/redirect/order"},filterName = "userFilter")
    public class UserFilter implements Filter {
    
        @Override
        public void init(FilterConfig filterConfig) throws ServletException {
    
        }
    
        @Override
        public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
            HttpServletRequest request = (HttpServletRequest) servletRequest;
            HttpServletResponse response = (HttpServletResponse) servletResponse;
            HttpSession session = request.getSession();
            User user = (User) session.getAttribute("user");
            if(user == null){
                response.sendRedirect("login.html");
            }else{
                filterChain.doFilter(servletRequest,servletResponse);
            }
        }
    
        @Override
        public void destroy() {
    
        }
    }
    

9\. 在启动类 ClientFeignApplication 定义处添加 @ServletComponentScan 注解，让过滤器生效。

![1](https://images.gitbook.cn/7e2d7680-dd58-11e9-aaec-b5744b419935)

10\. 在 resources 目录下创建 static 文件夹，前端的静态资源全部放入 static
目录下，客户端即可通过浏览器直接访问这些静态资源而不必通过后台映射。

![](https://images.gitbook.cn/842bbbf0-dd58-11e9-a584-59c5758c1abc)

但是需要注意的是，Spring Boot 中使用 Thymeleaf 模版来处理 HTML 的话，Session 会失效，必须通过后台 Handler
的映射来到 HTML，才可以访问到 Session。那么为什么直接访问 JSP 就可以拿到 Session 呢？因为 Session 是 JSP
内置对象，也就是一个 Java 对象，能被访问的前提是必须实例化，JSP 资源在被访问时，JSP 引擎会将 JSP 转化成 Servlet，然后在这个
Servlet 中会实例化 Session 对象，即在 JSP 资源中，Session 已经完成了实例化，所以是可以访问。

但是在静态 HTML 页面中，没有实例化 Session 的动作，所以无法访问 Session，必须先通过 Handler 来实例化 Session
对象，并且以转发的形式来到 HTML，Session 才可以被访问。这里必须是转发，如果后台通过重定向来到 HTML 页面，同样无法访问
Session，核心问题在于 Session 被访问之前是否已经被实例化。

11\. 代码全部完成之后，依次启动注册中心、配置中心、account、menu、order、user、clientfeign，打开浏览器访问
http://localhost:8761，可以看到所有微服务的注册信息，如下图所示。

![3](https://images.gitbook.cn/9cdc88f0-dd58-11e9-9cc8-a572519b0723)

4 个服务提供者和 1 个服务消费者已经全部完成注册，接下来就可以访问服务消费者 clientfeign 的相关业务功能了，如下图所示。

![4](https://images.gitbook.cn/a3579f80-dd58-11e9-aaec-b5744b419935)

![5](https://images.gitbook.cn/aa1da1c0-dd58-11e9-9cc8-a572519b0723)

![6](https://images.gitbook.cn/afce3c60-dd58-11e9-aaec-b5744b419935)

### 总结

本节课我们讲解了项目实战客户端模块的搭建，在整个系统中作为服务消费者，调用各个服务提供者包括 account、menu、order、user
的相关接口完成具体业务需求，客户端模块相当于一个大集成者，统一处理系统的业务请求，同时整合了视图层技术 Thymeleaf，完成用户交互。

[请单击这里下载源码](https://github.com/southwind9801/orderingsystem.git)

[微服务项目实战视频链接请点击这里获取](https://pan.baidu.com/s/1eheDU4XoN3BKuzocyIe0oA)，提取码：bfps

