### 前言

前面的课程我们已经详细讲解了 MongoDB 数据库的安装及使用，实际开发中，我们可以通过 Spring Data 来简化对 MongoDB 数据库的管理。

Spring Data 是 Spring
提供的持久层产品，主要功能是为应用程序中的数据访问提供统一的开发模型，同时保留不同数据存储的特殊性，并且这套开发模式是基于 Spring
的。根据不同类型的数据存储类型又可分为 Spring Data JDBC、Spring Data JPA、Spring Data Redis、Spring
Data MongoDB 等，适用于关系型数据库和非关系型数据库。

使用 Spring Data 管理 MongoDB 数据库非常方便，Spring Data 可以理解为父级框架，具体管理 MongoDB 数据库的技术栈是
Spring Data MongoDB。

Spring Data 在数据持久层已经写好了常用功能，我们只需要定义一个接口去继承 Spring Data
提供的接口，就可以实现对数据库的操作，也可以自定义接口方法，甚至这些自定义方法都不需要我们手动去实现，Reposity 会自动完成实现，Reposity 是
Spring Data 的核心接口，泛型中的 T 表示实体类型，ID 表示实体类的标识符 id。

Reposity 是一个空的父接口，我们在开发中不会直接使用，最常用的是它的一个子接口 CrudReposity，该接口中定义了常规 CRUD
方法，如下所示。

  * `Optional <T> findById(ID id);`
  * `boolean existsById(ID id);`
  * `Iterable <T> findAll();`
  * `Iterable <T> findAllById(Iterable<ID> ids);`
  * `long count();`
  * `void deleteById(ID id);`
  * `void delete(T entity);`
  * `void deleteAll(Iterable<? extends T> entities);`
  * `void deleteAll();`

实际开发中我们只需要自定义接口，继承 CurdReposity 就可以使用了，不需要自己完成接口的实现。

下面通过一个例子来学习如何使用 Spring Data MongoDB 快速开发程序。

（1）创建 Student 实体类。

    
    
    @Document(collection="student_info")
    public class Student {
        @Id
        private String id;
        @Field
        private String name;
        @Field
        private int age;
        @Field(value="a_time")
        private Date addTime;
    }
    

（2）自定义 StudentReposity 接口，继承 CrudReposity，添加 @Reposity 注解。

    
    
    @Repository
    public interface StudentReposity extends CrudRepository<Student, String>{
    
    }
    

（3）spring.xml 中进行配置自动扫描，IoC 容器管理 Reposity 接口。

    
    
    <!-- Spring 连接 MongoDB 客户端配置 -->
    <mongo:mongo-client host="127.0.0.1" port="12345" id="mongo"/>
    
    <!-- 配置 MongoDB 目标数据库 -->
    <mongo:db-factory dbname="testdb" mongo-ref="mongo" />
    
    <!-- 配置 MongoTemplate -->
    <bean id="mongoTemplate" class="org.springframework.data.mongodb.core.MongoTemplate">
      <constructor-arg name="mongoDbFactory" ref="mongoDbFactory"/>
    </bean>
    
    <!-- 扫描 Reposity 接口 -->
    <mongo:repositories base-package="com.southwind.repository"></mongo:repositories>
    

（4）从 IoC 容器中获取 StudentRepostiy 实例，调用其方法完成对数据库的操作。

  * 查询总记录数

    
    
    long count = studentReposity.count();
    System.out.println(count);
    

![](https://images.gitbook.cn/b3cc8d90-9adc-11e8-8cbe-ad3f3badcc18)

  * 根据 id 查询数据

![](https://images.gitbook.cn/cac438e0-9adc-11e8-831e-0180aea56660)

    
    
    Student student = studentReposity.findById("5b62c6f55a41f60d93ffe6c2").get();
    System.out.println(student);
    

![](https://images.gitbook.cn/e5b92980-9adc-11e8-b37c-dd4feba3837e)

  * 查询全部数据

    
    
    Iterator<Student> iterator = studentReposity.findAll().iterator();
    while(iterator.hasNext()) {
        Student Student = iterator.next();
        System.out.println(Student);
    }
    

![](https://images.gitbook.cn/074d8190-9add-11e8-a178-519d5b470954)

  * 查询 id 集合对应的数据

    
    
    List<String> ids = new ArrayList<String>();
    ids.add("5b62c6f55a41f60d93ffe6c2");
    ids.add("5b62c6f55a41f60d93ffe6c3");
    ids.add("5b62c6f55a41f60d93ffe6c4");
    Iterator<Student> iterator = studentReposity.findAllById(ids).iterator();
    while(iterator.hasNext()) {
        Student Student = iterator.next();
        System.out.println(Student);
    }
    

![](https://images.gitbook.cn/1bc14120-9add-11e8-831e-0180aea56660)

  * 根据 id 查询数据是否存在

    
    
    boolean flag = studentReposity.existsById("5b62c6f55a41f60d93ffe6c2");
    System.out.println(flag);
    

![](https://images.gitbook.cn/299969d0-9add-11e8-a178-519d5b470954)

  * 根据 id 删除数据

    
    
    studentReposity.deleteById("5b62c6f55a41f60d93ffe6c2");
    

![](https://images.gitbook.cn/3a306460-9add-11e8-a178-519d5b470954)

记录张三 1 删除成功。

  * 删除一组数据

    
    
    Student Student = studentReposity.findById("5b62c6f55a41f60d93ffe6c1").get();
    Student Student2 = studentReposity.findById("5b62c6f55a41f60d93ffe6c3").get();
    Student Student3 = studentReposity.findById("5b62c6f55a41f60d93ffe6c4").get();
    List<Student> list = new ArrayList<Student>();
    list.add(Student);
    list.add(Student2);
    list.add(Student3);
    studentReposity.deleteAll(list);
    

删除之前：

![](https://images.gitbook.cn/4ae76a10-9add-11e8-a178-519d5b470954)

删除之后：

![](https://images.gitbook.cn/5b17daa0-9add-11e8-b37c-dd4feba3837e)

  * 删除全部数据

    
    
    studentReposity.deleteAll();
    

![](https://images.gitbook.cn/6bfc0ad0-9add-11e8-b37c-dd4feba3837e)

使用 CrudReposity 接口定义好的方法操作数据库非常方便，同时我们也可以根据需求自定义方法，并且不需要实现，Reposity
会自动实现这些自定义方法，但是使用时需要注意命名规范。

  * 根据 name 查询数据

自定义方法。

    
    
    @Repository
    public interface StudentReposity extends CrudRepository<Student, String>{
        public List<Student> findByName(String name);
    }
    

直接调用。

    
    
    List<Student> list = studentReposity.findByName("张三1");
    for (Student Student : list) {
        System.out.println(Student);
    }
    

数据库记录如下。

![](https://images.gitbook.cn/8ed36a30-9add-11e8-8cbe-ad3f3badcc18)

查询结果。

![](https://images.gitbook.cn/98695550-9add-11e8-a178-519d5b470954)

  * 根据 name 和 age 查询数据

自定义方法。

    
    
    @Repository
    public interface StudentReposity extends CrudRepository<Student, String>{
        public List<Student> findByNameAndAge(String name,int age);
    }
    

直接调用。

    
    
    List<Student> list = studentReposity.findByNameAndAge("张三1",16);
    for (Student Student : list) {
        System.out.println(Student);
    }
    

数据库记录如下。

![](https://images.gitbook.cn/acb40910-9add-11e8-b37c-dd4feba3837e)

查询结果。

![](https://images.gitbook.cn/b6e87e20-9add-11e8-8cbe-ad3f3badcc18)

  * 查询全部数据并排序

自定义方法。

    
    
    @Repository
    public interface StudentReposity extends CrudRepository<Student, String>{
        public List<Student> findAll(Sort sort);
    }
    

直接调用，Direction.DESC 表示降序排列。

    
    
    List<Student> list = studentReposity.findAll(
                  Sort.by(Direction.DESC,"age"));
    for (Student Student : list) {
        System.out.println(Student);
    }
    

查询结果。

![](https://images.gitbook.cn/cac729a0-9add-11e8-a178-519d5b470954)

若要升序排列，将 Direction.DESC 替换为 Direction.ASC 即可。

### 总结

本节课我们讲解了 Spring Data 框架管理 MongoDB 数据库的另外一种方式——Reposity，它其实就是一套 Spring Data
帮我们封装的底层接口，使得开发者在完成数据的 CRUD 操作时无需自己写接口，自己写命令，直接调用封装好的接口方法即可，非常方便快捷。

### 源码

[请点击这里查看源码](https://github.com/southwind9801/springdata_repository.git)

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。
>
> 温馨提示：需购买才可入群哦，加小助手微信后需要截已购买的图来验证~

