### 前言

对于任何一个 Java 开发人员，Spring 的大名一定如雷贯耳，在行业中可谓是无人不知、无人不晓，说它是 Java 领域第一框架毫不为过。

![enter image description
here](https://images.gitbook.cn/971d5a50-a6dc-11e9-883b-23c2459fead5)

（图片来自 Spring 官网）

Spring 概念诞生于 2002 年，创始人 Rod Jahnson 在其著作《Expert One-on-One J2EE Design and
Development》中第一次提出了 Spring 的核心思想，于 2003 年正式发布第一个版本 Spring Framework 0.9。

经过十几年的优化迭代，Spring Framework 已经从最初的取代 EJB 框架逐步发展为一套完整的生态，最新的版本是 5.X，支持现代 Java
开发的各个技术领域，家族两大核心成员 Spring Boot 和 Spring Cloud 更是当下 Java 领域最为热门的技术栈。

毋庸置疑，Spring 已经成为 Java 开发的行业标准，无论你是初级程序员还是架构师，只要是做 Java 开发的，工作中或多或少一定会接触到
Spring 相关技术栈。

我们所说的 Spring 全家桶各个模块都是基于 Spring Framework 衍生而来，通常所说的 Spring 框架一般泛指 Spring
Framework，它包含 IoC 控制反转、DI 依赖注入、AOP 面向切面编程、Context 上下文、bean 管理、Spring Web MVC
等众多功能模块，其他的 Spring 家族成员都需要依赖 Spring Framework。

可以简单理解 Spring Framework
是一个设计层面的框架，通过分层思想来实现组件之间的解耦合，开发者可以根据需求选择不同的组件，并且可以非常方便的进行集成，Spring Framework
的这一特性使得企业级项目开发变得更加简单方便。

Spring 的两大核心机制是 IoC（控制反转）和 AOP（面向切面编程），对于初学者来讲，搞清楚这两个核心机制就掌握了 Spring
的基本应用。这两大核心机制也是 Java 设计模式的典型代表，其中 IoC 是工厂模式，AOP 是代理模式。

> [点击这里了解《Spring
> 全家桶》](https://gitbook.cn/gitchat/column/5d2daffbb81adb3aa8cab878?utm_source=springquan002)

### 什么是 IoC 和 AOP

下面来详细了解 IoC，IoC 是 Spring 框架的灵魂，非常重要，理解了 IoC 才能真正掌握 Spring 框架的使用。

**IoC 也叫控制反转** ，首先从字面意思理解，什么叫控制反转？反转的是什么？

在传统的程序开发中，需要获取对象时，通常由开发者来手动创建实例化对象，但是在 Spring 框架中创建对象的工作不再由开发者完成，而是交给 IoC
容器来创建，我们直接获取即可，整个流程完成反转，因此是控制反转。

举个例子：超市购物。

  * **传统方式** ：你去超市买东西，需要自己拿着袋子去超市购买商品，然后自己把袋子提回来。
  * **IoC 容器** ：你只需要把袋子放在家门口，袋子里面会自动装满你需要的商品，直接取出来用就可以了。

我们通过创建一个 Student 对象的例子来对比两种方式的区别。

#### 传统方式

（1）创建 Student 类

    
    
    public class Student {
        private int id;
        private String name;
        private int age;
    }
    

（2）测试方法中调用构造函数创建对象

    
    
    Student student = new Student();
    

#### IoC 容器

##### **实现步骤**

  * 在 pom.xml 中添加 Spring 依赖
  * 创建配置文件，可以自定义文件名 spring.xml
  * 在 spring.xml 中配置 bean 标签，IoC 容器通过加载 bean 标签来创建对象
  * 调用 API 获取 IoC 创建的对象

IoC 容器可以调用无参构造或者有参构造来创建对象，我们先来看无参构造的方式。

##### **无参构造**

    
    
    <!-- 配置 student 对象-->
    <bean id="stu" class="com.southwind.entity.Student"</bean>
    

配置一个 bean 标签：

  * id，对象名
  * class，对象的模板类

接下来调用 API 获取对象，Spring 提供了两种方式来获取对象：id 或者运行时类。

（1）通过 id 获取对象

    
    
    //1.加载 spring.xml 配置文件
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    //2.通过 id 值获取对象
    Student stu = (Student) applicationContext.getBean("stu");
    System.out.println(stu);
    

第一步：加载 spring.xml 配置文件，生成 ApplicationContext 对象。

第二步：调用 ApplicationContext 的 getBean 方法获取对象，参数为配置文件中的 id 值。程序在加载 spring.xml 时创建
stu 对象，通过反射机制调用无参构造函数，所有要求交给 IoC 容器管理的类必须有无参构造函数。

运行结果如下图所示。

![](https://images.gitbook.cn/512e7870-96e4-11e8-9f54-b3cc9167c22b)

可以看到，此时 stu 对象的属性全部为空，因为调用无参构造只会创建对象而不会进行赋值，如何赋值呢？只需要在 spring.xml
中进行相关配置即可，如下所示。

    
    
    <!-- 配置 student 对象 -->
    <bean id="stu" class="com.southwind.entity.Student">
        <property name="id" value="1"></property>
        <property name="name" value="张三"></property>
        <property name="age" value="23"></property>
    </bean>
    

添加 property 标签：name 对应属性名，value 是属性的值。若包含特殊字符，比如 name=""，使用 \\]]> 进行配置，如下所示。

    
    
    <!-- 配置 student 对象 -->
    <bean id="stu" class="com.southwind.entity.Student">
       <property name="id" value="1"></property>
       <property name="name">
           <value><![CDATA[<张三>]]></value>
       </property>
       <property name="age" value="23"></property>
    </bean>
    

运行结果如下图所示。

![](https://images.gitbook.cn/5fcf9670-96e4-11e8-9e0c-8bfb55c56242)

Spring 通过调用每个属性的 setter 方法来完成属性的赋值，因此实体类必须有 setter 方法，否则加载时报错，getter 方法可省略。

（2）通过运行时类获取对象

    
    
    //1.加载 spring.xml 配置文件
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    //2.通过运行时类获取对象
    Student stu = applicationContext.getBean(Student.class);
    System.out.println(stu);
    

此方法有一个弊端，当 spring.xml 中配置两个 Student 的 bean 时程序会抛出异常，因为此时两个 bean 都是由 Student
类生成的，IoC 容器无法将两个 bean 都返回，必须指定一个唯一的 bean。

    
    
    <bean id="stu" class="com.hzit.entity.Student">
       <property name="id" value="1"></property>
       <property name="name">
           <value><![CDATA[<张三>]]></value>
       </property>
            <property name="age" value="23"></property>
    </bean>
    <bean id="stu2" class="com.hzit.entity.Student">
       <property name="id" value="1"></property>
       <property name="name" value="李四"></property>
       <property name="age" value="23"></property>
    </bean>
    

异常信息如下图所示。

![](https://images.gitbook.cn/6764cf90-96e4-11e8-9f54-b3cc9167c22b)

以上是 IoC 容器通过无参构造创建对象的方式，同时 IoC 容器也可以调用有参构造来创建对象。

##### **有参构造**

（1）在实体类中创建有参构造

    
    
    public Student(int id, String name, int age) {
            super();
            this.id = id;
            this.name = name;
            this.age = age;
    }
    

（2）spring.xml 中进行配置

    
    
    <!-- 通过有参构造函数创建对象 -->
    <bean id="stu3" class="com.hzit.entity.Student">
       <constructor-arg name="id" value="3"></constructor-arg>
       <constructor-arg name="name" value="小明"></constructor-arg>
       <constructor-arg name="age" value="22"></constructor-arg>
    </bean>
    

（3）此时 IoC 容器会根据 constructor-arg 标签去加载对应的有参构造函数，创建对象并完成属性赋值。name
的值需要与有参构造的形参名对应，value 是对应的值。除了使用 name 对应参数外，还可以通过下标 index 对应，如下所示。

    
    
    <!-- 通过有参构造函数创建对象 -->
    <bean id="stu3" class="com.hzit.entity.Student">
       <constructor-arg index="0" value="3"></constructor-arg>
       <constructor-arg index="1" value="小明"></constructor-arg>
       <constructor-arg index="2" value="22"></constructor-arg>
    </bean>
    

以上是 IoC 容器通过有参构造创建对象的方式，获取对象同样有两种方式可以选择：id 和运行时类。

##### **如果 IoC 容器管理多个对象，并且对象之间有级联关系，如何实现？**

（1）创建 Classes 类

    
    
    public class Classes {
        private int id;
        private String name;
    }
    

（2）在 Student 类中添加 Classes 属性

    
    
    public class Student {
        private int id;
        private String name;
        private int age;
        private Classes classes;
    }
    

（3）spring.xml 中配置 classes 对象，然后将该对象赋值给 stu 对象

    
    
    <!-- 创建 classes 对象 -->
    <bean id="classes" class="com.hzit.entity.Classes">
       <property name="id" value="1"></property>
       <property name="name" value="Java班"></property>
    </bean>
    <!-- 创建 stu 对象 -->
    <bean id="stu" class="com.hzit.entity.Student">
       <property name="id" value="1"></property>
       <property name="name">
           <value><![CDATA[<张三>]]></value>
       </property>
       <property name="age" value="23"></property>
       <!-- 将 classes 对象赋给 stu 对象 -->
       <property name="classes" ref="classes"></property>
    </bean>
    

再次获取 Student 对象，结果如下图所示。

![](https://images.gitbook.cn/0d91b450-96e5-11e8-ab3d-7b3c8b8e2dff)

在 spring.xml 中，通过 ref 属性将其他 bean 赋给当前 bean 对象，这种方式叫做依赖注入（DI），是 Spring
非常重要的机制，DI 是将不同对象进行关联的一种方式，是 IoC 的具体实现方式，通常 DI 和 IoC 是紧密结合在一起的，因此一般说的 IoC 包括
DI。

##### **如果是集合属性如何依赖注入？**

（1）Classes 类中添加 List\ 属性。

    
    
    public class Classes {
        private int id;
        private String name;
        private List<Student> students;
    }
    

（2）spring.xml 中配置 2 个 student 对象、1 个 classes 对象，并将 2 个 student 对象注入到 classes
对象中。

    
    
    <!-- 配置 classes 对象 -->
    <bean id="classes" class="com.hzit.entity.Classes">
       <property name="id" value="1"></property>
       <property name="name" value="Java班"></property>
       <property name="students">
           <!-- 注入 student 对象 -->
           <list>
               <ref bean="stu"/>
               <ref bean="stu2"/>
           </list>
       </property>
    </bean>
    <bean id="stu" class="com.hzit.entity.Student">
       <property name="id" value="1"></property>
       <property name="name">
            <value><![CDATA[<张三>]]></value>
       </property>
       <property name="age" value="23"></property>
    </bean>
    <bean id="stu2" class="com.hzit.entity.Student">
       <property name="id" value="2"></property>
       <property name="name" value="李四"></property>
       <property name="age" value="23"></property>
    </bean>
    

运行结果如下图所示。

![](https://images.gitbook.cn/1d27e0b0-96e5-11e8-ab3d-7b3c8b8e2dff)

集合属性通过 list 标签和 ref 标签完成注入，ref 的 bean 属性指向需要注入的 bean 对象。

### 总结

这一讲我们讲解了 Spring IoC 的基本概念以及如何使用，IoC 是 Spring 的核心，这很重要。使用 Spring
开发项目时，控制层、业务层、DAO 层都是通过 IoC 来完成依赖注入的。

### 分享交流

我们为本课程付费读者创建了微信交流群，以方便更有针对性地讨论课程相关问题。入群方式请到第 5 篇末尾添加小助手的微信号，并注明「全家桶」。

阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

> 温馨提示：需购买才可入群哦，加小助手微信后需要截已购买的图来验证~

[请单击这里下载源码](https://github.com/southwind9801/Spring-Ioc-1.git)

> [点击这里了解《Spring
> 全家桶》](https://gitbook.cn/gitchat/column/5d2daffbb81adb3aa8cab878?utm_source=springquan002)

