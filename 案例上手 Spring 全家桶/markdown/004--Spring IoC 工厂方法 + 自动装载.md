### 前言

这一讲我们继续来学习 IoC 的两个常用知识点：IoC 通过工厂方法创建对象、IoC 自动装载（autowire）。

### IoC 通过工厂方法创建对象

之前说过 IoC 是典型的工厂模式，下面我们就来学习如何使用工厂模式来创建 bean，IoC 通过工厂模式创建 bean 有两种方式：

  * 静态工厂方法
  * 实例工厂方法

按照惯例，我们还是通过代码来带大家去学习工厂方法，先来学习静态工厂方法。

（1）创建 Car 实体类

    
    
    public class Car {
        private int num;
        private String brand;
        public Car(int num, String brand) {
            super();
            this.num = num;
            this.brand = brand;
        }
    }
    

（2）创建静态工厂类、静态工厂方法

    
    
    public class StaticCarFactory {
        private static Map<Integer,Car> cars;
        static{
            cars = new HashMap<Integer,Car>();
            cars.put(1, new Car(1,"奥迪"));
            cars.put(2, new Car(2,"奥拓"));
        }
        public static Car getCar(int num){
            return cars.get(id);
        }
    }
    

（3）在 spring.xml 中配置静态工厂

    
    
    <!-- 配置静态工厂创建 car 对象 -->
    <bean id="car1" class="com.southwind.entity.StaticCarFactory" factory-method="getCar">
       <constructor-arg value="1"></constructor-arg>
    </bean>
    

  * factory-method 指向静态方法；
  * constructor-arg 的 value 属性为调用静态方法所传的参数。

（4）在测试类中直接获取 car1 对象。

    
    
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    Car car = (Car) applicationContext.getBean("car1");
    System.out.println(car);
    

运行结果如下图所示。

![](https://images.gitbook.cn/eb768390-96e5-11e8-9f54-b3cc9167c22b)

> [点击这里了解《Spring
> 全家桶》](https://gitbook.cn/gitchat/column/5d2daffbb81adb3aa8cab878?utm_source=springquan004)

#### 实例工厂方法

（1）创建实例工厂类、工厂方法

    
    
    public class InstanceCarFactory {
        private Map<Integer,Car> cars;
        public InstanceCarFactory() {
            cars = new HashMap<Integer,Car>();
            cars.put(1, new Car(1,"奥迪"));
            cars.put(2, new Car(2,"奥拓"));
        }
        public Car getCar(int num){
            return cars.get(num);
        }
    }
    

（2）在 spring.xml 中配置 bean

    
    
    <!-- 配置实例工厂对象 -->
    <bean id="carFactory" class="com.southwind.entity.InstanceCarFactory"></bean>
    <!-- 通过实例工厂对象创建 car 对象 -->
    <bean id="car2" factory-bean="carFactory" factory-method="getCar">
        <constructor-arg value="2"></constructor-arg>
    </bean> 
    

（3）在测试类中直接获取 car2 对象

    
    
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    Car car2 = (Car) applicationContext.getBean("car2");
    System.out.println(car2);
    

运行结果如下图所示。

![](https://images.gitbook.cn/f52c52c0-96e5-11e8-ab3d-7b3c8b8e2dff)

对比两种方式的区别，静态工厂方法的方式创建 car 对象，不需要实例化工厂对象，因为静态工厂的静态方法，不需要创建对象即可调用。所以 spring.xml
只需要配置一个 Car bean，而不需要配置工厂 bean。

实例工厂方法创建 car 对象，必须先实例化工厂对象，因为调用的是非静态方法，必须通过对象调用，不能直接通过类来调用，所以 spring.xml
中需要先配置工厂 bean，再配置 Car bean。

其实这里考察的就是静态方法和非静态方法的调用。

### IoC 自动装载（autowire）

我们通过前面的学习，掌握了 IoC 创建对象的方式，以及 DI 完成依赖注入的方式，通过配置 property 的 ref 属性，可将 bean
进行依赖注入。

同时 Spring 还提供了另外一种更加简便的方式：自动装载，不需要手动配置 property，IoC 容器会自动选择 bean 完成依赖注入。

自动装载有两种方式：

  * byName，通过属性名自动装载；
  * byType，通过属性对应的数据类型自动装载。

通过代码来学习。

（1）创建 Person 实体类

    
    
    public class Person {
        private int id;
        private String name;
        private Car car;
    }
    

（2）在 spring.xml 中配置 Car bean 和 Person bean，并通过自动装载进行依赖注入。

创建 person 对象时，没有在 property 中配置 car 属性，因此 IoC 容器会自动进行装载，autowire="byName"
表示通过匹配属性名的方式去装载对应的 bean，Person 实体类中有 car 属性，因此就将 id="car" 的 bean 注入到 person 中。

> 注意：通过 property 标签手动进行 car 的注入优先级更高，若两种方式同时配置，以 property 的配置为准。
    
    
    <bean id="person" class="com.southwind.entity.Person" autowire="byName"> 
       <property name="id" value="1"></property>
       <property name="name" value="张三"></property>
    </bean>
    <bean id="car" class="com.southwind.entity.StaticCarFactory" factory-method="getCar">
       <constructor-arg value="2"></constructor-arg>
    </bean>
    

（3）测试类中获取 person 对象

    
    
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    Person person = (Person) applicationContext.getBean("person");
    System.out.println(person);
    

运行结果如下图所示。

![](https://images.gitbook.cn/fec8cd90-96e5-11e8-ab3d-7b3c8b8e2dff)

同理，byType 即通过属性的数据类型来配置。

（1）spring.xml 配置如下

    
    
    <bean id="person" class="com.southwind.entity.Person" autowire="byType"> 
       <property name="id" value="1"></property>
       <property name="name" value="张三"></property>
    </bean>
    <bean id="car" class="com.southwind.entity.StaticCarFactory" factory-method="getCar">
       <constructor-arg value="1"></constructor-arg>
    </bean>
    <bean id="car2" class="com.southwind.entity.StaticCarFactory" factory-method="getCar">
       <constructor-arg value="2"></constructor-arg>
    </bean>
    

（2）测试类中获取 person 对象

    
    
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
    Person person = (Person) applicationContext.getBean("person");
    System.out.println(person);
    

运行结果如下图所示。

![](https://images.gitbook.cn/0bc98200-96e6-11e8-9e0c-8bfb55c56242)

可以看到控制台直接打印异常信息，这是因为使用了 byType 进行自动装载，但是 spring.xml 中配置了两个 Car bean，IoC
容器不知道应该将哪一个 bean 装载到 person 对象中，所以抛出异常。使用 byType 进行自动装载时，spring.xml
中只能配置一个被装载的 bean。

（3）现修改 spring.xml 配置，只保留一个 bean，如下所示：

    
    
    <bean id="person" class="com.southwind.entity.Person" autowire="byType"> 
        <property name="id" value="1"></property>
        <property name="name" value="张三"></property>
    </bean>
    <bean id="car" class="com.southwind.entity.StaticCarFactory" factory-method="getCar">
        <constructor-arg value="1"></constructor-arg>
    </bean>
    

（4）测试类中获取 person 对象，运行结果如下图所示：

![](https://images.gitbook.cn/155ad940-96e6-11e8-9f54-b3cc9167c22b)

### 总结

本讲我们讲解了 Spring IoC 的工厂方法与自动装载，工厂方式是 Spring
框架对工厂模式的实现。自动装载是对上一讲内容的依赖注入优化，在维护对象关联关系的基础上，进一步简化配置。

### 分享交流

我们为本课程付费读者创建了微信交流群，以方便更有针对性地讨论课程相关问题。入群方式请到第 5 篇末尾添加小助手的微信号。

阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

[请单击这里下载源码](https://github.com/southwind9801/Spring-Ioc-3.git)

> [点击这里了解《Spring
> 全家桶》](https://gitbook.cn/gitchat/column/5d2daffbb81adb3aa8cab878?utm_source=springquan004)

