### 前言

MyBatis 框架支持数据的级联查询，当然它不会自动完成，需要开发者手动在 Mapper.xml
中进行映射配置，比如我们拿客户（Customer）和订单（Order）举例，每一个订单都有对应的客户，在程序中的体现是查询到一个 Order
对象之后，可以直接访问到对应的 Customer 对象，比如根据 ID 查询订单，输出其客户姓名，代码如下所示。

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream inputStream = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(inputStream);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            OrderRepository orderRepository = sqlSession.getMapper(OrderRepository.class);
            Order order = orderRepository.findById(1L);
            System.out.println(order.getCustomer().getName());
        }
    }
    

运行结果如下图所示。

![](https://images.gitbook.cn/f0355030-b342-11e9-84a8-2972098a9052)

可以看到 MyBatis 是通过两次 SQL 查询取到了客户姓名，为什么是两次呢？一是通过 ID 查询 t_order
表，取出相关数据，二是通过级联的外键查询 t_customer 表，从而取出“张三”。

但是这种方式会存在一个问题，当我们只需要获取订单信息的时候，同样会查询两张表，比如根据 ID 查询订单名称，代码如下所示。

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream inputStream = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(inputStream);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            OrderRepository orderRepository = sqlSession.getMapper(OrderRepository.class);
            Order order = orderRepository.findById(1L);
            System.out.println(order.getName());
        }
    }
    

运行结果如下图所示。

![](https://images.gitbook.cn/061c00b0-b343-11e9-84a8-2972098a9052)

通过结果我们可以看到，此时还是执行了两次 SQL 语句，但是订单名称通过 1 就可以取到，2
的执行完全没有必要，这就造成了资源浪费，是我们在实际开发中应该避免的，如何来解决这个问题呢？

解决方案就是这节课我们要学习的延迟加载。

延迟加载也叫惰性加载或者懒加载。使用延迟加载是如何提高程序运行效率的呢？对于持久层操作有一个原则，Java
程序与数据库交互频率越低越好，这点很好理解，Java 程序每一次和数据库进行交互，都需要先进行验证等操作，会消耗资源，因此实际开发中应该尽量减少 Java
程序与数据库的交互次数，以提高程序的整体运行效率，MyBatis 为我们提供了延迟加载机制，就可以很好地做到这一点。

我们还是通过客户（Customer）和订单（Order）的业务模型来理解延迟加载，这是一个典型的一对多关系模型，即一个客户（Customer）可对应多个订单（Order），但是一个订单（Order）只能对应一个客户（Customer），分别创建实体类
Customer 和 Order。

    
    
    public class Customer {
        private Long id;
        private String name;
        private List<Order> orders;
    }
    
    
    
    public class Order {
        private Long id;
        private String name;
        private Customer customer;
    }
    

按照这个模型在数据库建表，t_customer 为主表，t_order 为从表，t_order 表中的外键 cid 被 t_customer 的主键 id
约束，建表语句如下所示。

    
    
    CREATE TABLE `t_customer` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `name` varchar(11) DEFAULT NULL,
      PRIMARY KEY (`id`)
    )
    
    CREATE TABLE `t_order` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `name` varchar(11) DEFAULT NULL,
      `cid` int(11) DEFAULT NULL,
      PRIMARY KEY (`id`),
      KEY `cid` (`cid`),
      CONSTRAINT `t_order_ibfk_1` FOREIGN KEY (`cid`) REFERENCES `t_customer` (`id`)
    )
    

具体的业务是，当我们查询 Order 对象时，因为有级联关系，所以会将对应的 Customer 对象一并查询出来，这样就需要发送两条 SQL 语句，分别查询
t_order 表和 t_customer 表中的数据。

延迟加载的思路是：当我们查询 Order 的时候，如果没有调用 customer 属性，则只发送一条 SQL 语句查询 Order；如果需要调用
customer 属性，则发送两条 SQL 语句查询 Order 和 Customer。因此延迟加载可以看做是一种优化机制，根据具体的代码，自动选择发送的
SQL 语句条数。

接下来我们通过代码来实现延迟加载。

创建 OrderRepository，如下所示。

    
    
    public interface OrderRepository {
        public Order findById(Long id);
    }
    

对应的 OrderRepository.xml 如下所示。

    
    
    <mapper namespace="com.southwind.repository.OrderRepository">
    
        <resultMap type="com.southwind.entity.Order" id="orderMap">
            <id property="id" column="id"/>
            <result property="name" column="name"/>
            <association property="customer" javaType="com.southwind.entity.Customer" select="com.southwind.repository.CustomerRepository.findById" column="cid"></association>
        </resultMap>
    
        <select id="findById" parameterType="java.lang.Long" resultMap="orderMap">
            select * from t_order where id = #{id}
        </select>
    
    
    </mapper>
    

创建 CustomerRepository，如下所示。

    
    
    public interface CustomerRepository {
        public Customer findById(Long id);
    }
    

对应的 CustomerRepository.xml 如下所示。

    
    
    <mapper namespace="com.southwind.repository.CustomerRepository">
    
        <select id="findById" parameterType="java.lang.Long" resultType="com.southwind.entity.Customer">
            select * from t_customer where id = #{id}
        </select>
    
    </mapper>
    

通过 id 查询 Order 对象，并输出 name 值。

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream inputStream = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(inputStream);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            OrderRepository orderRepository = sqlSession.getMapper(OrderRepository.class);
            Order order = orderRepository.findById(1L);
            System.out.println(order.getName());
        }
    }
    

结果如下图所示。

![WX20190723-162453@2x](https://images.gitbook.cn/ae9b7ad0-b13c-11e9-8d9f-5d806c870c68)

可以看到，执行了两条 SQL，分别查询出 Order 对象和级联的 Customer 对象，但是此时我们只需要输出 Order 的 name，没有必要去查询
Customer 对象，开启延迟加载，可解决这个问题。

在 config.xml 中开启延迟加载。

    
    
    <configuration>
    
        <!--设置 settings -->
        <settings>
            <!--打印 SQL -->
            <setting name="logImpl" value="STDOUT_LOGGING"/>
            <!--打开延迟加载的开关 -->
            <setting name="lazyLoadingEnabled" value="true" />
        </settings>
    
    </configuration>
    

再次查询 Order，输出 name，可以看到只打印了一条 SQL
语句，相比于上一次操作，本次操作就少了一次与数据库的交互，在同样可以完成需求的情况下，提高了程序的效率。

![WX20190723-162626@2x](https://images.gitbook.cn/d73cf040-b13c-11e9-aadf-b32faf6b5e19)

查询 Order，输出级联 Customer 对象的 name 值。

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream inputStream = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(inputStream);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            OrderRepository orderRepository = sqlSession.getMapper(OrderRepository.class);
            Order order = orderRepository.findById(1L);
            System.out.println(order.getCustomer().getName());
        }
    }
    

结果如下图所示。

![WX20190723-162739@2x](https://images.gitbook.cn/e84a2c40-b13c-11e9-83fd-07d7da80e621)

可以看到执行了两条 SQL，此时的业务需求必须去查询 Customer 对象才能获取其 name 值，用到了 Order 对象所级联的 Customer
对象，按需加载，所以在执行查询 Order 的 SQL 的同时，也需要执行查询 Customer 的 SQL。

### 总结

本节课我们讲解了 MyBatis 框架的延迟加载机制，是实际开发中使用频率较高的一个功能，正确地使用延迟加载，可以有效减少 Java Application
与数据库的交互频次，从而提高整个系统的运行效率，延迟加载适用于多表关联查询的业务场景。

[请点击这里查看源码](https://github.com/southwind9801/gcmybatis.git)

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

