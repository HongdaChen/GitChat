### 前言

上一讲学习了 MyBatis 框架的基本原理和基本使用，这一讲继续深入 MyBatis 框架的学习，通过上一讲的学习我们知道 MyBatis
主要有两类配置文件：全局配置文件和 Mapper 配置文件，并且这两类配置文件的名字都可以自定义。

全局配置文件主要用来定义数据源信息和一些基本配置，如事务管理、打印 SQL 语句、开启二级缓存、设置延迟加载等。Mapper
配置文件的功能就比较单一，定义对应接口具体业务实现的 SQL 语句，同时需要注意的是 Mapper 配置文件一定要在全局配置文件中完成注册，否则
MyBatis 无法完成 Mapper 接口的实现。

MyBatis 是“半自动”的 ORM 框架，所有的 SQL 语句需要开发者自定义在 Mapper.xml 中，再由 MyBatis 框架完成 POJO 与
SQL 之间的映射关系，这一讲我们来学习 Mapper.xml 的常用属性。

实际开发中对数据库的查询可分为单表查询和多表关联查询，多表关联查询又包括一对一、一对多、多对多 3 种关系。

### 单表查询

首先来看一段完整的 Mapper.xml 代码，如下所示。

    
    
    <select id="findById" parameterType="java.lang.Integer" resultType="com.southwind.entity.Student">
        select * from t_student where id=#{id}
    </select>
    

这是在 Mapper.xml 中定义了一个通过 Id 查询 Student 对象的方法，目标表是 t_student，对应的实体类是
com.southwind.entity.Student，我们要做的是在 Mapper.xml 设置相关配置，然后由 MyBatis 自动完成查询，并生成
POJO。

可以看到关键属性有 id、parameterType、resultType。

其中 id 是对应接口的方法名，parameterType 定义参数的数据类型，resultType 定义查询结果的数据类型。

搞清楚了这几个属性的作用，你就掌握了 Mapper.xml 的使用。

### parameterType

parameterType 支持基本数据类型、包装类、String、多参数、POJO 等，我们分别来演示具体的使用。

（1）基本数据类型，通过 id 查询 Student。

接口定义如下所示。

    
    
    public Student findById(int id);
    

接口对应的 Mapper.xml 定义如下所示。

    
    
    <select id="findById" parameterType="int" resultType="com.southwind.entity.Student">
        select * from t_student where id=#{id}
    </select>
    

（2）包装类，通过 id 查询 Student。

接口定义如下所示。

    
    
    public Student findById(Integer id);
    

接口对应的 Mapper.xml 定义如下所示。

    
    
    <select id="findById" parameterType="java.lang.Integer" resultType="com.southwind.entity.Student">
        select * from t_student where id=#{id}
    </select>
    

（3）String 类型，通过 name 查询 Student。

接口定义如下所示。

    
    
    public Student findByName(String name);
    

接口对应的 Mapper.xml 定义如下所示。

    
    
    <select id="findByName" parameterType="java.lang.String" resultType="com.southwind.entity.Student">
        select * from t_student where name = #{name}
    </select>
    

（4）多个参数，通过 id 和 name 查询 Student。两个参数分别是 Integer 类型和 String 类型，类型不一致，此时
parameterType 可以省略，通过参数下标取出参数值。

接口定义如下所示。

    
    
    public Student findByIdAndName(Integer id,String name);
    

接口对应的 Mapper.xml 定义如下所示。

    
    
    <select id="findByIdAndName" resultType="com.southwind.entity.Student">
        select * from t_student where id = #{param1} and name = #{param2}
    </select>
    

这里需要注意的是通过 #{param1} 和 #{param2} 来映射两个参数，以此类推，如果有第 3 个参数，则使用 #{param3} 来映射。

（5）POJO，多个参数一个个写太麻烦了，这时候我们可以将参数列表进行封装，将封装对象作为 parameterType 的值。

接口定义如下所示。

    
    
    public Student findByStudent(Student student);
    

接口对应的 Mapper.xml 定义如下所示。

    
    
    <select id="findByStudent" parameterType="com.southwind.entity.Student" resultType="com.southwind.entity.Student">
        select * from t_student where id = #{id} and name = #{name}
    </select>
    

与多个参数的不同之处在于，这里是通过 #{属性名} 来映射参数对象的具体属性值的。

### resultType

resultType 的使用与 parameterType 基本一致，如下所示。

（1）基本数据类型，统计 Student 的总记录数。

接口定义如下所示。

    
    
    public int count();
    

接口对应的 Mapper.xml 定义如下所示。

    
    
    <select id="count" resultType="int">
        select count(*) from t_student
    </select>
    

（2）包装类，统计 Student 的总记录数。

接口定义如下所示。

    
    
    public Integer count();
    

接口对应的 Mapper.xml 定义如下所示。

    
    
    <select id="count" resultType="java.lang.Integer">
        select count(*) from t_student
    </select>
    

（3）String 类型，根据 id 查询 name。

接口定义如下所示。

    
    
    public String findNameById(Integer id);
    

接口对应的 Mapper.xml 定义如下所示。

    
    
    <select id="findNameById" parameterType="java.lang.Integer" resultType="java.lang.String">
        select name from t_student where id = #{id}
    </select>
    

（4）POJO，如通过 id 查询 Student，上面已经介绍过了，这里就不再重复了。

### 多表关联查询

实际开发中不可能只是对单表进行操作，一定会涉及到多表关联查询，数据表之间的关系有三种：一对一关系、一对多关系、多对多关系，实际开发中最常用的是一对多关系和多对多关系，我们分别来介绍
MyBatis 框架如何实现这两种关系。

（1）一对多

上面我们演示的是 Student 单表查询，如果是多表关联查询，比如查询 Student 同时级联对应的 Classes，如何处理呢？使用
resultType 无法完成，我们以通过 id 查询 Student 来举例。

创建数据表

    
    
    CREATE TABLE `t_classes` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `name` varchar(11) DEFAULT NULL,
      PRIMARY KEY (`id`)
    )
    
    CREATE TABLE `t_student` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `name` varchar(11) DEFAULT NULL,
      `cid` int(11) DEFAULT NULL,
      PRIMARY KEY (`id`),
      KEY `cid` (`cid`),
      CONSTRAINT `t_student_ibfk_1` FOREIGN KEY (`cid`) REFERENCES `t_classes` (`id`)
    ) 
    

SQL：

    
    
    select s.id sid,s.name sname,c.id cid,c.name cname from t_student s,t_classes c where s.cid = c.id and s.id = 1;
    

查询结果：

![](https://images.gitbook.cn/963a8d30-acf7-11e9-a760-01c165706a91)

实体类 Student：

    
    
    public class Student {
        private Integer id;
        private String name;
        private Classes classes;
    }
    

Classes：

    
    
    public class Classes {
        private Integer id;
        private String name;
        private List<Student> students;
    }
    

MyBatis 会自动将结果与实体类进行映射，将字段的值赋给对应的属性，若字段名与属性名一致，完成赋值，那么问题来了。

![](https://images.gitbook.cn/bbb9b400-acf7-11e9-a760-01c165706a91)

如图，id、name 属性可以对应字段 sid 和 sname，classes 属性没有对应的字段，准确的讲，classes 属性需要对应的对象为
cid，cname 封装起来的对象，此时需要使用 resultMap 来完成映射。

StudentRepository：

    
    
    public Student findById(Integer id);
    

修改 StudentRepository.xml，使用 association 标签配置 classes 级联，因为一个 Student 只能对应一个
Classes。

    
    
    <resultMap type="com.southwind.entity.Student" id="studentMap">
        <id property="id" column="sid"/>
        <result property="name" column="sname"/>
        <!-- 映射 classes 属性 -->
        <association property="classes" javaType="com.southwind.entity.Classes">
            <id property="id" column="cid"/>
            <result property="name" column="cname"/>
        </association>
    </resultMap>
    
    <select id="findById" parameterType="java.lang.Integer" resultMap="studentMap">
        select s.id sid,s.name sname,c.id cid,c.name from t_student s,t_classes c where s.cid = c.id and s.id = #{id}; 
    </select>
    

同理，反过来查询 Classes，将级联的所有 Student 一并查询。

ClassesRepository：

    
    
    public Classes findById(Integer id);
    

ClassesRepository.xml 中使用 collection 标签配置 students 级联，因为一个 Classes 可以对应多个
Student。

    
    
    <resultMap type="com.southwind.entity.Classes" id="classesMap">
        <id property="id" column="cid"/>
        <result property="name" column="cname"/>
        <!-- 映射 students 属性 -->
        <collection property="students" ofType="com.southwind.entity.Student">
            <id property="id" column="sid"/>
            <result property="name" column="sname"/>
        </collection>
    </resultMap>
    
    <select id="findById" parameterType="java.lang.Integer" resultMap="classesMap">
        select c.id cid,c.name cname,s.id sid,s.name sname from t_classes c,t_student s where c.id = s.cid and c.id = #{id};
    </select>
    

需要注意的是 association 标签，通过设置 javaType 属性，映射实体类，collection 标签，通过设置 ofType
属性映射实体类。

（2）多对多

多对多其实是双向的一对多关系，我们用学生选课的具体场景来模拟，一个学生可以选择多门课程，同理一门课程也可以被多个学生选择，即一个 Student
可以对应多个 Course，一个 Course 也可以对应多个 Student，所以双方都是用 collection 标签设置级联，创建数据表。

    
    
    CREATE TABLE `t_student` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `name` varchar(11) DEFAULT NULL,
      PRIMARY KEY (`id`)
    )
    
    CREATE TABLE `t_course` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `name` varchar(11) DEFAULT NULL,
      PRIMARY KEY (`id`)
    )
    
    CREATE TABLE `student_course` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `sid` int(11) DEFAULT NULL,
      `cid` int(11) DEFAULT NULL,
      PRIMARY KEY (`id`),
      KEY `sid` (`sid`),
      KEY `cid` (`cid`),
      CONSTRAINT `student_course_ibfk_1` FOREIGN KEY (`sid`) REFERENCES `t_student` (`id`),
      CONSTRAINT `student_course_ibfk_2` FOREIGN KEY (`cid`) REFERENCES `t_course` (`id`)
    )
    

创建实体类 Student。

    
    
    public class Student {
        private Integer id;
        private String name;
        private List<Course> courses;
    }
    

创建实体类 Course。

    
    
    public class Course {
        private Integer id;
        private String name;
        private List<Student> students;
    }
    

StudentRepository：

    
    
    public Student findById(Integer id);
    

StudentRepository.xml：

    
    
    <resultMap type="com.southwind.entity.Student" id="studentMap">
        <id property="id" column="sid"/>
        <result property="name" column="sname"/>
        <!-- 映射 course 属性 -->
        <collection property="courses" ofType="com.southwind.entity.Course">
            <id property="id" column="cid"/>
            <result property="name" column="cname"/>
        </collection>
    </resultMap>
    
    <select id="findById" parameterType="java.lang.Integer" resultMap="studentMap">
        select s.id sid,s.name sname,c.id cid,c.name cname from t_student s,t_course c,student_course sc where s.id = sc.sid and c.id = sc.cid and s.id = #{id};
    </select>
    

CourseRepository：

    
    
    public Course findById(Integer id);
    

CourseRepository.xml：

    
    
    <resultMap type="com.southwind.entity.Course" id="courseMap">
        <id property="id" column="cid"/>
        <result property="name" column="cname"/>
        <!-- 映射 students 属性 -->
        <collection property="students" ofType="com.southwind.entity.Student">
            <id property="id" column="sid"/>
            <result property="name" column="sname"/>
        </collection>
    </resultMap>
    
    <select id="findById" parameterType="java.lang.Integer" resultMap="courseMap">
        select s.id sid,s.name sname,c.id cid,c.name cname from t_student s,t_course c,student_course sc where s.id = sc.sid and c.id = sc.cid and c.id = #{id};
    </select>
    

### 总结

本节课我们讲解了 MyBatis 框架中 Mapper.xml 的具体使用，可以说 Mapper 是 MyBatis 框架的核心机制，Mapper.xml
就是我们使用 MyBatis 的重中之重，MyBatis 具体的业务逻辑包括 POJO 与 SQL 的映射关系都是在 Mapper.xml
中进行配置的，是我们学习的重点。

[请单击这里下载源码](https://github.com/southwind9801/gcmybatis.git)

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

