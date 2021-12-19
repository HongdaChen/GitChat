### 前言

上一讲介绍了 Spring 的 IoC，即控制反转，程序中由 Spring 来管理对象，当需要使用某个对象时，直接通过 IoC 容器来获取对象，并通过 DI
来完成对象之间的注入关系。下面继续来学习 IoC 的相关知识。

### Spring 中的 bean

bean 是根据 scope 来生成的，表示 bean 的作用域，scope 有 4 种类型：

  * singleton，单例，表示通过 Spring 容器获取的该对象是唯一的；
  * prototype，原型，表示通过 Spring 容器获取的对象都是不同的；
  * request，请求，表示在一次 HTTP 请求内有效；
  * session，会话，表示在一个用户会话内有效。

后两种只适用于 Web 项目，大多数情况下，我们只会使用 singleton 和 prototype 两种 scope，并且 scope 的默认值是
singleton。

我们通过一个例子来学习这两种配置的区别。

（1）创建 User 实体类

    
    
    public class User {
        private int id;
        private String name;
        private int age;
        public User() {
             System.out.println("创建了User对象");
        }   
    }
    

（2）在 spring.xml 配置 User 的实例化 bean

    
    
    <bean id="user" class="com.southwind.entity.User">
       <property name="id" value="1"></property>
       <property name="name" value="张三"></property>
       <property name="age" value="23"></property>
    </bean>
    

（3）测试类中通过 Spring 容器获取两个 User 实例化对象 user1 和 user2，并且通过 == 方法判断是否为同一个对象

    
    
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    User user = (User) applicationContext.getBean("user");
    User user2 = (User) applicationContext.getBean("user");
    System.out.println(user == user2);
    

运行结果如下图所示。

![](https://images.gitbook.cn/446b73d0-96e5-11e8-9e0c-8bfb55c56242)

看到结果打印 true，并且 User 的构造函数只执行了一次，表示 user1 和 user2 是同一个对象，因此 scope 默认值为
singleton，即默认通过 Spring 容器创建的对象都是单例模式。

修改 spring.xml 中的配置，将 scope 改为 prototype。

    
    
    <bean id="user" class="com.southwind.entity.User" scope="prototype">
       <property name="id" value="1"></property>
       <property name="name" value="张三"></property>
       <property name="age" value="23"></property>
    </bean>
    

执行代码，结果如下图所示。

![](https://images.gitbook.cn/50b449f0-96e5-11e8-a80f-e9b555a946c4)

可以看到，调用了两次构造函数，并且 == 的结果为 false，表示现在的 user1 和 user2 是两个对象。

> [点击这里了解《Spring
> 全家桶》](https://gitbook.cn/gitchat/column/5d2daffbb81adb3aa8cab878?utm_source=springquan003)

### Spring 的继承

Spring 的继承与 Java 的继承不一样，但思想很相似，子 bean 可以继承父 bean 中的属性。看到这里，你可能会有这样的疑问：子 bean
可以继承父 bean 中的方法吗？

其实这里不存在方法的继承，Spring 的继承是在对象层面进行操作的，即两个 bean 来自同一个类，因此方法是一样的，不存在继承关系，具体使用如下所示。

（1）在 spring.xml 中配置两个 User bean，并建立继承关系。

    
    
    <bean id="user" class="com.southwind.entity.User">
       <property name="id" value="1"></property>
       <property name="name" value="张三"></property>
       <property name="age" value="23"></property>
    </bean>
    <bean id="user2" class="com.southwind.entity.User" parent="user"></bean>
    

（2）运行代码，结果如下。

    
    
    User user2 = (User) applicationContext.getBean("user2");
    System.out.println(user2);
    

![](https://images.gitbook.cn/9e5ecbd0-96e5-11e8-9e0c-8bfb55c56242)

可以看到，创建了两个 User 对象 user1 和 user2，并且 user2 继承了 user1 的所有属性。user2 在继承 user1
所有属性的基础之上，还可以对属性进行覆盖，直接在 spring.xml 中添加 property 即可。

    
    
    <bean id="user" class="com.southwind.entity.User">
       <property name="id" value="1"></property>
       <property name="name" value="张三"></property>
       <property name="age" value="23"></property>
    </bean>
    <bean id="user2" class="com.southwind.entity.User" parent="user">
       <!-- 覆盖 name 属性 -->
       <property name="name" value="李四"></property>
    </bean>
    

再次运行代码，结果如下图所示。

![](https://images.gitbook.cn/b10d34b0-96e5-11e8-9f54-b3cc9167c22b)

name 属性已经被修改为李四，完成覆盖。有读者可能会问，Spring 中的 bean 能不能在不同类之间继承？

答案是可以的，但是需要这两个类的属性列表完全一致，否则会报错，实际开发中并不会使用到这种方式。

### Spring 的依赖

与继承类似，依赖也是 bean 和 bean 之间的一种关联方式，配置依赖关系后，被依赖的 bean 一定先创建，再创建依赖的
bean，我们还是通过代码来理解。

（1）创建 Car 实体类

    
    
    public class Car {
        private int id;
        private String brand;
        public Car() {
            System.out.println("创建了Car对象");
        }
    }
    

（2）在 spring.xml 中配置 User bean、Car bean

    
    
    <bean id="user" class="com.southwind.entity.User">
       <property name="id" value="1"></property>
       <property name="name" value="张三"></property>
       <property name="age" value="23"></property>
    </bean>
    <bean id="car" class="com.southwind.entity.Car">
       <property name="id" value="1"></property>
       <property name="brand" value="宝马"></property>
    </bean>
    

（3）测试类中获取两个 bean

    
    
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    User user = (User) applicationContext.getBean("user");
    Car car = (Car) applicationContext.getBean("car");
    

结果如下图所示。

![](https://images.gitbook.cn/bb27b920-96e5-11e8-a80f-e9b555a946c4)

看到结果，先创建 User，后创建 Car，这是由 spring.xml 中 bean 的配置顺序来决定的，先到先得，先配置 User
bean，因此先创建了 User 对象。现在修改 spring.xml 配置，User 依赖 Car，设置 depends-on 属性，如下所示。

    
    
    <bean id="user" class="com.southwind.entity.User" depends-on="car">
       <property name="id" value="1"></property>
       <property name="name" value="张三"></property>
       <property name="age" value="23"></property>
    </bean>
    <bean id="car" class="com.southwind.entity.Car">
       <property name="id" value="1"></property>
       <property name="brand" value="宝马"></property>
    </bean>
    

再次运行代码，看到结果，先创建 Car，后创建 User，因为 User 依赖于 Car，所以必须先创建 Car 对象，User 对象才能完成依赖。

![](https://images.gitbook.cn/c54fab10-96e5-11e8-9e0c-8bfb55c56242)

### Spring 读取外部资源

在实际开发中，数据库配置一般会保存在 Properties 文件中方便维护，如果使用 Spring 容器来生成数据源对象，如何读取到 properties
配置文件中的内容？

（1）创建 jdbc.properties

    
    
    driverName = com.mysql.jdbc.Driver
    url = jdbc:mysql://localhost:3306/myTest?useUnicode=true&characterEncoding=UTF-8
    user = root
    pwd = root
    

（2）spring.xml 中配置 C3P0 数据源

    
    
    <!-- 导入外部的资源文件 -->
    <context:property-placeholder location="classpath:jdbc.properties"></context:property-placeholder>
    <!-- 创建 C3P0 数据源 -->
    <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
       <property name="user" value="${user}"></property>
       <property name="password" value="${pwd}"></property>
       <property name="driverClass" value="${driverName}"></property>
       <property name="jdbcUrl" value="${url}"></property>
    </bean>
    

第一步：导入外部资源文件。

使用 context:property-placeholder 标签，需要导入 context 命名空间。

第二步：通过 ${} 表达式取出外部资源文件的值。

（3）测试类中获取 Spring 创建的 dataSource 对象

    
    
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    DataSource ds = (DataSource) applicationContext.getBean("dataSource");
    Connection conn = null;
    try {
         conn = ds.getConnection();
    } catch (SQLException e) {
         // TODO Auto-generated catch block
         e.printStackTrace();
    }
    System.out.println(conn);
    

（4）运行代码，看到结果，打印 dataSource 对象

![](https://images.gitbook.cn/d19573f0-96e5-11e8-a80f-e9b555a946c4)

除了使用 \ 元素为 Bean 的属性装配值和引用外，Spring 还提供了另外一种 bean 属性的装配方式：p 命名空间，该方式进一步简化配置代码。

### p 命名空间

p 命名空间可以简化 bean 的各种配置，直接通过代码来学习。

（1）在 User 实体类中添加 Car 属性

    
    
    public class User {
        private int id;
        private String name;
        private int age;
        private Car car;
        public User() {
            System.out.println("创建了User对象");
        }
        @Override
        public String toString() {
            return "User [id=" + id + ", name=" + name + ", age=" + age + ", car="
                    + car + "]";
        }
    }
    

（2）spring.xml 中创建 User bean 和 Car bean，并且通过 p 命名空间给属性赋值，同时完成依赖注入，注意需要引入 p 命名间

    
    
    <bean id="user" class="com.southwind.entity.User" p:id="1" p:name="张三" p:age="23" p:car-ref="car"></bean>
    <bean id="car" class="com.southwind.entity.Car" p:id="1" p:brand="宝马"></bean>
    

（3）测试类中获取 User 对象，并打印

    
    
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    User user = (User) applicationContext.getBean("user");
    System.out.println(user);
    

运行结果如下图所示，创建了 User 对象和 Car 对象，并且完成属性赋值，以及级联关系。

![](https://images.gitbook.cn/dae77840-96e5-11e8-a80f-e9b555a946c4)

### 总结

这一讲我们讲解了 Spring IoC 两种常规操作，分别是依赖注入与 p 命名空间，依赖注入是维护对象关联关系的机制。p
命名空间实际上是对配置文件的简化，以提高我们的开发效率。

[请单击这里下载源码](https://github.com/southwind9801/Spring-Ioc-2.git)

### 分享交流

我们为本课程付费读者创建了微信交流群，以方便更有针对性地讨论课程相关问题。入群方式请到第 5 篇末尾添加小助手的微信号，并注明「全家桶」。

阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

> 温馨提示：需购买才可入群哦，加小助手微信后需要截已购买的图来验证~
>
> [点击了解《Spring
> 全家桶》](https://gitbook.cn/gitchat/column/5d2daffbb81adb3aa8cab878?utm_source=springquan003)

