### 前言

上节课我们学习了 MyBatis 延迟加载，可以有效减少 Java
程序与数据库的交互次数，从而提高程序的运行效率，但是延迟加载的功能并不全面，它只能在做级联查询的时候提高效率，如果现在的需求就是单表查询，那么延迟加载就无法满足需求了。不用担心，MyBatis
同样为我们提供了这种业务场景下的解决方案，即缓存。

使用缓存的作用也是减少 Java 应用程序与数据库的交互次数，从而提升程序的运行效率。比如第一次查询出某个对象之后，MyBatis
会自动将其存入缓存，当下一次查询同一个对象时，就可以直接从缓存中获取，不必再次访问数据库了，如下图所示。

![enter image description
here](https://images.gitbook.cn/0925eaa0-b68b-11e9-9778-7bbd052235e6)

MyBatis 有两种缓存：一级缓存和二级缓存。

MyBatis 自带一级缓存，并且是无法关闭的，一直存在，一级缓存的数据存储在 SqlSession 中，即它的作用域是同一个
SqlSession，当使用同一个 SqlSession 对象执行查询的时候，第一次的执行结果会自动存入 SqlSession
缓存，第二次查询时可以直接从缓存中获取。

但是如果是两个 SqlSession 查询两次同样的 SQL，一级缓存不会生效，需要访问两次数据库。

同时需要注意，为了保证数据的一致性，如果 SqlSession 执行了增加、删除，修改操作，MyBatis 会自动清空 SqlSession
缓存中存储的数据。

一级缓存不需要进行任何配置，可以直接使用。

MyBatis 二级缓存是比一级缓存作用域更大的缓存机制，它是 Mapper 级别的，只要是同一个 Mapper，无论使用多少个 SqlSession
来操作，数据都是共享的，多个不同的 SqlSession 可以共用二级缓存。

MyBatis 二级缓存默认是关闭的，需要使用时可以通过配置手动开启。

下面我们通过代码来学习如何使用 MyBatis 缓存。

首先来演示一级缓存，以查询 Classes 对象为例。

### 一级缓存

1\. Classes 实体类

    
    
    public class Classes {
        private Long id;
        private String name;
        private List<Student> students;
    }
    

2\. ClassesReposirory 接口

    
    
    public interface ClassesRepository {
        public Classes findById(Long id);
    }
    

3\. ClassesReposirory.xml

    
    
    <mapper namespace="com.southwind.repository.ClassesRepository">
    
        <select id="findById" parameterType="java.lang.Long" resultType="com.southwind.entity.Classes">
            select * from classes where id = #{id}
        </select>
    
    </mapper>
    

4\. config.xml

    
    
    <configuration>
    
        <mappers>
            <mapper resource="com/southwind/repository/ClassesReposirory.xml"/>
        </mappers>
    
    </configuration>
    

5\. Test 测试类

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream inputStream = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(inputStream);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            ClassesRepository classesRepository = sqlSession.getMapper(ClassesRepository.class);
            Classes classes = classesRepository.findById(2L);
            System.out.println(classes);
            Classes classes2 = classesRepository.findById(2L);
            System.out.println(classes2);
        }
    }
    

结果如下图所示。

![WX20190617-170626@2x](https://images.gitbook.cn/8ed4f300-b13e-11e9-8d9f-5d806c870c68)

可以看到结果，执行了一次 SQL 语句，查询出两个对象，第一个对象是通过 SQL 查询的，并保存到缓存中，第二个对象是直接从缓存中获取的。

我们说过一级缓存是 SqlSession 级别的，所以 SqlSession 一旦关闭，缓存也就不复存在了，修改代码，再次测试。

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream inputStream = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(inputStream);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            ClassesRepository classesRepository = sqlSession.getMapper(ClassesRepository.class);
            Classes classes = classesRepository.findById(2L);
            System.out.println(classes);
            sqlSession.close();
            sqlSession = sqlSessionFactory.openSession();
            classesRepository = sqlSession.getMapper(ClassesRepository.class);
            Classes classes2 = classesRepository.findById(2L);
            System.out.println(classes2);
            sqlSession.close();
        }
    }
    

结果如下图所示。

![WX20190617-170226@2x](https://images.gitbook.cn/a093cbc0-b13e-11e9-9446-7bea49a7ddfe)

可以看到，执行了两次 SQL，在关闭 SqlSession、一级缓存失效的情况下，可以启用二级缓存，实现提升效率的需求。

### 二级缓存

MyBatis 可以使用自带的二级缓存，也可以使用第三方的 ehcache 二级缓存。

先来使用 MyBatis 自带的二级缓存，具体步骤如下所示。

1\. config.xml 中配置开启二级缓存

    
    
    <configuration>
    
        <!-- 设置 settings -->
        <settings>
            <!-- 打印 SQL -->
            <setting name="logImpl" value="STDOUT_LOGGING"/>
            <!-- 开启二级缓存 -->
            <setting name="cacheEnabled" value="true"/>
        </settings>
    
    </configuration>
    

2\. ClassesReposirory.xml 中配置二级缓存

    
    
    <mapper namespace="com.southwind.repository.ClassesRepository">
    
        <cache></cache>
    
        <select id="findById" parameterType="java.lang.Long" resultType="com.southwind.entity.Classes">
            select * from classes where id = #{id}
        </select>
    
    </mapper>
    

3\. Classes 实体类实现 Serializable 接口

    
    
    public class Classes implements Serializable {
        private long id;
        private String name;
        private List<Student> students;
    }
    

4\. 测试

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream inputStream = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(inputStream);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            ClassesRepository classesRepository = sqlSession.getMapper(ClassesRepository.class);
            Classes classes = classesRepository.findById(2L);
            System.out.println(classes);
            sqlSession.close();
            sqlSession = sqlSessionFactory.openSession();
            classesRepository = sqlSession.getMapper(ClassesRepository.class);
            Classes classes2 = classesRepository.findById(2L);
            System.out.println(classes2);
            sqlSession.close();
        }
    }
    

结果如下图所示。

![WX20190617-170516@2x](https://images.gitbook.cn/d8f033a0-b13e-11e9-83fd-07d7da80e621)

可以看到，执行了一次 SQL，查询出两个对象，二级缓存生效，接下来我们学习如何使用 ehcache 二级缓存，具体步骤如下所示。

1\. pom.xml 添加 ehcache 相关依赖

    
    
    <dependencies>
      <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis-ehcache</artifactId>
        <version>1.0.0</version>
      </dependency>
    
      <dependency>
        <groupId>net.sf.ehcache</groupId>
        <artifactId>ehcache-core</artifactId>
        <version>2.4.3</version>
      </dependency>
    </dependencies>
    

2\. 在 resources 路径下创建 ehcache.xml

    
    
    <ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="../config/ehcache.xsd">
    <diskStore/>
        <defaultCache
            maxElementsInMemory="1000"
            maxElementsOnDisk="10000000"
            eternal="false"
            overflowToDisk="false"
            timeToIdleSeconds="120"
            timeToLiveSeconds="120"
            diskExpiryThreadIntervalSeconds="120"
            memoryStoreEvictionPolicy="LRU">
        </defaultCache>
    </ehcache>
    

3\. config.xml 中配置开启二级缓存

    
    
    <configuration>
    
        <!-- 设置settings -->
        <settings>
            <!-- 打印SQL -->
            <setting name="logImpl" value="STDOUT_LOGGING"/>
            <!-- 开启二级缓存 -->
            <setting name="cacheEnabled" value="true"/>
        </settings>
    
    </configuration>
    

4\. ClassesReposirory.xml 中配置二级缓存

    
    
    <mapper namespace="com.southwind.repository.ClassesReposirory"> 
        <!-- 开启二级缓存 -->
        <cache type="org.mybatis.caches.ehcache.EhcacheCache" >
            <!-- 缓存创建以后，最后一次访问缓存的时间至失效的时间间隔 -->
            <property name="timeToIdleSeconds" value="3600"/>
            <!-- 缓存自创建时间起至失效的时间间隔-->
            <property name="timeToLiveSeconds" value="3600"/>
            <!-- 缓存回收策略，LRU 移除近期最少使用的对象 -->
            <property name="memoryStoreEvictionPolicy" value="LRU"/>
        </cache>
    
        <select id="findById" parameterType="java.lang.Long" resultType="com.southwind.entity.Classes">
            select * from classes where id = #{id}
        </select>
    
    </mapper>
    

5\. Classes 实体类不需要实现 Serializable 接口

    
    
    public class Classes {
        private long id;
        private String name;
        private List<Student> students;
    }
    

6\. 测试

    
    
    public class Test {
        public static void main(String[] args) {
            InputStream inputStream = Test.class.getClassLoader().getResourceAsStream("config.xml");
            SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBuilder.build(inputStream);
            SqlSession sqlSession = sqlSessionFactory.openSession();
            ClassesRepository classesRepository = sqlSession.getMapper(ClassesRepository.class);
            Classes classes = classesRepository.findById(2L);
            System.out.println(classes);
            sqlSession.close();
            sqlSession = sqlSessionFactory.openSession();
            classesRepository = sqlSession.getMapper(ClassesRepository.class);
            Classes classes2 = classesRepository.findById(2L);
            System.out.println(classes2);
            sqlSession.close();
        }
    }
    

结果如下图所示。

![WX20190617-171012@2x](https://images.gitbook.cn/05d1b930-b13e-11e9-8d9f-5d806c870c68)

同样执行一次 SQL，查询出两个对象，ehcache 二级缓存生效。

### 总结

本节课我们讲解了 MyBatis 框架的缓存机制，MyBatis 的缓存分两种：一级缓存和二级缓存，一级缓存是 SqlSession 级别的，二级缓存是
Mapper 级别的，使用时我们需要注意这两种缓存的区别。缓存机制跟延迟加载功能类型，都是通过减少 Java Application
与数据库的交互频次，从而提高系统的运行效率。

[请点击这里查看源码](https://github.com/southwind9801/gcmybatis.git)

