### 前言

前面内容对 Spring IoC 的基本使用做了详细的讲解，这一讲来学习 IoC 在实际开发中的应用，通过 IoC
容器可以更好地构建程序的分层结构，基本原理是 IoC 提供了各个组件的实例化对象，然后根据具体需求从 IoC
中取出相应组件完成依赖注入，就类似于搭积木，IoC 把零件提供好了，我们只需要进行组装即可。

实现方式有两种：基于 XML 配置文件和基于注解。

具体思路如下：把程序分为 Controller 层、Service 层和 DAO 层三层。

关系为 Controller 层调用 Service 层，Service 层调用 DAO 层，并且 Service 层和 DAO
层设计为接口，这是一个典型的 MVC 模式后台代码分层结构。

接下来我们学习两种实现方式的具体使用。

### 基于 XML 配置方式

（1）创建 UserController 类

    
    
    public class UserController {
    
        private UserService userService;
    
        public User getUserById(int id){
            return userService.getUserById(id);
        }
    
    }
    

（2）创建 UserService 接口以及实现类 UserServiceImpl

    
    
    public interface UserService {
        public User getUserById(int id);
    }
    
    public class UserServiceImpl implements UserService{
    
        private UserDAO userDAO;
    
        @Override
        public User getUserById(int id) {
            // TODO Auto-generated method stub
            return userDAO.getUserById(id);
        }
    
    }
    

（3）创建 UserDAO 接口以及实现类 UserDAOImpl

    
    
    public interface UserDAO {
        public User getUserById(int id);
    }
    
    public class UserDAOImpl implements UserDAO{
    
        private static Map<Integer,User> users;
    
        static{
            users = new HashMap<Integer,User>();
            users.put(1, new User(1, "张三"));
            users.put(2, new User(2, "李四"));
            users.put(3, new User(3, "王五"));
        }
    
        @Override
        public User getUserById(int id) {
            // TODO Auto-generated method stub
            return users.get(id);
        }
    
    }
    

（4）创建 User 实体类

    
    
    public class User {
        private int id;
        private String name;
        public User(int id, String name) {
            super();
            this.id = id;
            this.name = name;
        }
    }
    

（5）在 spring.xml 配置 userController、userService、userDAO，并完成依赖注入。

    
    
    <!-- 配置UserController -->
    <bean id="userController" class="com.southwind.controller.UserController">
        <property name="userService" ref="userService"></property>
    </bean>
    <!-- 配置UserService -->
    <bean id="userService" class="com.southwind.service.impl.UserServiceImpl">
        <property name="userDAO" ref="userDAO"></property>
    </bean>
    <!-- 配置UserDAO -->
    <bean id="userDAO" class="com.southwind.dao.impl.UserDAOImpl"></bean>
    

（6）在测试类中获取 userController 对象，调用方法获取 user 对象。

    
    
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    UserController userController = (UserController) applicationContext.getBean("userController");
    User user = userController.getUserById(1);
    System.out.println(user);
    

运行结果如下图所示。

![](https://images.gitbook.cn/268ab870-96e6-11e8-9e0c-8bfb55c56242)

### 基于注解的方式

第一步：将 UserController、UserService 和 UserDAO 类扫描到 IoC 容器中。

第二步：在类中设置注解完成依赖注入。

（1）修改 spring.xml

    
    
    <!-- 将类扫描到 IoC 容器中 -->
    <context:component-scan base-package="com.southwind"></context:component-scan>
    

base-package="com.southwind" 表示将 "com.southwind" 下所有子包的类全部扫描到 IoC
容器中，一步可将所有参与项目的类完成扫描注入。注意：配置文件需要引入 context 命名空间。

（2）修改 UserController，添加注解

    
    
    @Controller
    public class UserController {
    
        @Autowired
        private UserService userService;
    
        public User getUserById(int id){
            return userService.getUserById(id);
        }
    
    }
    

对比之前的代码，有两处改动：

  * 在类名处添加 @Controller 注解，表示该类作为一个控制器；
  * userService 属性处添加 @Autowired 注解，表示 IoC 容器自动完成装载，默认是 byType 的方式。

（3）修改 UserServiceImpl

    
    
    @Service
    public class UserServiceImpl implements UserService{
    
        @Autowired
        private UserDAO userDAO;
    
        @Override
        public User getUserById(int id) {
            // TODO Auto-generated method stub
            return userDAO.getUserById(id);
        }
    
    }
    

同上，做了两处改动：

  * 在类名处添加 @Service 注解，表示该类是业务层；
  * userDAO 属性处添加 @Autowired 注解，表示 IoC 容器自动完成装载，默认是 byType 的方式。

（4）修改 UserDAOImpl

    
    
    @Repository
    public class UserDAOImpl implements UserDAO{
    
        private static Map<Integer,User> users;
    
        static{
            users = new HashMap<Integer,User>();
            users.put(1, new User(1, "张三"));
            users.put(2, new User(2, "李四"));
            users.put(3, new User(3, "王五"));
        }
    
        @Override
        public User getUserById(int id) {
            // TODO Auto-generated method stub
            return users.get(id);
        }
    
    }
    

做了一处改动：在类名处添加 @Repository 注解，表示该类是数据接口层。

（5）运行测试代码

    
    
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    UserController userController = (UserController) applicationContext.getBean("userController");
    User user = userController.getUserById(1);
    System.out.println(user);
    

运行结果如下图所示。

![](https://images.gitbook.cn/3035d940-96e6-11e8-ab3d-7b3c8b8e2dff)

成功，通过结果我们可以得出结论，使用注解的方式可大大简化代码的编写，因此在实际开发中，推荐使用基于注解的方式来架构分层。我们分别给
UserController、UserService、UserDAO 添加了 @Controller、@Service、@Repository 注解。

IoC 中可以给类添加的注解有 4 种：

  * @Controller 
  * @Service
  * @Repository
  * @Component

在实际开发中，我们使用 @Controller、@Service、@Repository 分别表示 Controller 层、Service 层、DAO
层。前面提到过，类中属性的自动装载默认是通过 byType 的方式实现的。自动装载除了 byType 的方式，还可以使用 byName，使用 byName
的方式，需要结合 @Qualifier 注解一起使用，具体操作如下所示。

    
    
    @Controller
    public class UserController {
    
        @Autowired()
        @Qualifier("userService")
        private UserService userService;
    
        public User getUserById(int id){
            return userService.getUserById(id);
        }
    }
    

我们知道 byName 的方式是通过属性名去匹配对应 bean 的 id 属性值，但是基于注解的方式我们并没有给 bean 设置 id，如何完成呢？

其实在类中添加注解时，已经设置了默认的 id，即类名首字母小写之后的值就是 id 的默认值。

    
    
    @Service
    public class UserServiceImpl implements UserService
    

此时，IoC 容器中默认赋值，UserService bean 的 id=userService，与 UserController
中的属性名一致，因此可以完成自动。

现在做出修改，手动赋值，设置 UserService bean 的 id=myUserService，如下所示。

    
    
    @Service("myUserService")
    public class UserServiceImpl implements UserService{
    
        @Autowired
        private UserDAO userDAO;
    
        @Override
        public User getUserById(int id) {
            // TODO Auto-generated method stub
            return userDAO.getUserById(id);
        }
    
    }
    

很显然，UserController 中的 userService 属性也需要去匹配 name=myUserService 的 bean，因此设置
@Qualifier("myUserService")，如下所示。

    
    
    @Controller
    public class UserController {
    
        @Autowired()
        @Qualifier("myUserService")
        private UserService userService;
    
        public User getUserById(int id){
            return userService.getUserById(id);
        }
    }
    

@Qualifier() 中的值必须与 @Service() 中的值一致，才能完成自动装载。

### 总结

本讲我们讲解了 Spring 基于注解的开发方式，相比于传统的基于 XML
配置文件的方式，基于注解的方式很显然更加方便、快捷，可以提高开发效率，在实际开发中我们也会选择这种方式。

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

![R2Y8ju](https://images.gitbook.cn/R2Y8ju.jpg)

