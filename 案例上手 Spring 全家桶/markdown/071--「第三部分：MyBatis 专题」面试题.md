第三部分（第 21 ~ 29 篇）这部分内容将为大家详细讲解主流的 ORMapping 框架
MyBatis，包括常用模块的使用和底层实现原理，作为持久层的实现方案，MyBatis 在实际项目开发中会与 Spring MVC 整合使用。

（1）简单谈谈你对 MyBatis 的理解？

**点击查看答案**

  * Mybatis 是一个半自动化的 ORM（对象关系映射）框架，它内部封装了JDBC，开发时只需要关注 SQL 语句本身，不需要花费精力去处理加载驱动、创建连接、创建 Statement 等繁杂的过程。程序员直接编写原生态 SQL，可以严格控制 SQL 执行性能，灵活度高。
  * MyBatis 可以使用 XML 或注解来配置和映射原生信息，将 POJO 映射成数据库中的记录，避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。
  * 通过 XML 文件或注解的方式将要执行的各种 Statement 配置起来，并通过 POJO 和 Statement 中 SQL 的动态参数进行映射生成最终执行的 SQL 语句，最后由 MyBatis 框架执行 SQL 并将结果映射为 POJO 并返回。

    

（2）MyBatis 接口绑定的优点是什么？

**点击查看答案**

接口映射就是在 MyBatis 中任意定义接口，然后把接口里面的方法和 SQL 语句绑定，开发者直接调用接口方法就即可，这样比起原生 SqlSession
提供的方法，开发者有了更加灵活的选择和设置。

    

（3）实现 MyBatis 接口绑定分别有哪几种方式?

**点击查看答案**

接口绑定有两种实现方式，一种是通过注解绑定，就是在接口的方法上面添加 @Select、@Update 等注解里面包含 SQL 语句来绑定；另外一种是通过在
XML 文件中编写 SQL 语句来绑定，这种情况下要指定 XML 映射文件里的 namespace 必须为接口的全路径名。

    

（4）MyBatis 如何实现一对一关联关系？

**点击查看答案**

  * 有联合查询和嵌套查询，联合查询是几个表进行关联查询，只查询一次，通过在 resultMap 中配置 association节点配置一对一的映射就可以完成。
  * 嵌套查询是先查一个表，根据该表查询结果的外键 id，去另外一个表里面查询数据，通过 association 配置进行关联，但另外一个表的查询需要通过 select 属性进行配置。

    

（5）MyBatis 如何实现一对多关联关系？

**点击查看答案**

  * 有联合查询和嵌套查询，联合查询是几个表进行关联查询，只查询一次，通过在 resultMap 中配置 collection 节点配置一对多的就可以完成。
  * 嵌套查询是先查一个表，根据该表查询结果的外键 id，去另外一个表里面查询数据，通过配置 collection 进行关联，但另外一个表的查询需要通过 select 属性进行配置。

    

（6）说说 MyBatis 动态 SQL 的具体使用步骤?

**点击查看答案**

MyBatis 的动态 SQL 一般是通过 if 节点来实现，通过 OGNL 语法来实现，但是如果要写完整，必须配合 where、trim 节点，where
节点用来判断是否包含其他子节点，有的话就插入 where，否则不插入，trim 节点是用来判断如果动态语句是以 and 或 or 开始，那么会自动把这个
and 或者 or 去掉。

    

（7）MyBatis 与 Hibernate 的区别是什么？

**点击查看答案**

  * Mybatis 和 Hibernate 不同，它不完全是一个 ORM 框架，因为 MyBatis 需要程序员自己编写 SQL 语句。
  * MyBatis 直接编写原生 SQL，可以严格控制 SQL 执行性能，灵活度高，非常适合对关系数据模型要求不高的软件开发，因为这类软件需求变化频繁，一但需求变化要求迅速输出成果。但是灵活的前提是 MyBatis 无法做到数据库无关性，如果需要实现支持多种数据库的应用，则需要自定义多套 SQL 映射文件，工作量大。
  * Hibernate 对象/关系映射能力强，数据库无关性好，对于关系模型要求高的应用，用 Hibernate 开发可以节省很多代码，提高效率。

    

（8）MyBatis 如何实现模糊查询?

**点击查看答案**

    
    
    <select id="findAllByKeyWord" parameterType="java.lang.String" resultType="com.southwind.entity.Product">
      select * from easybuy_product where name like "%"#{keyWord}"%"
    </select>
    

    

（9）Nginx 反向代理实现高并发的具体步骤是什么？

**点击查看答案**

  * 一个主进程，多个工作进程，每个工作进程可以处理多个请求，每进来一个 request，会有一个 worker 进程去处理。但不是全程的处理，处理到可能发生阻塞的地方，比如向上游（后端）服务器转发 request，并等待请求返回。那么这个处理的 worker 继续处理其他请求，而一旦上游服务器返回了，就会触发这个事件，worker 才会来接手，这个 request 才会接着往下走。
  * 由于 Web Server 的工作性质决定了每个 request 的大部份生命都是在网络传输中，实际上花费在 Server 机器上的时间片不多，这是用几个进程就可以实现高并发的秘密所在。

    

（10）Nginx 搭建 Tomcat 集群的核心配置应该怎么写？

**点击查看答案**

    
    
    #tomcat集群
    upstream  myapp {   #tomcat集群名称 
        server    localhost:8080;   #tomcat1配置
        server    localhost:8181;   #tomcat2配置
    }
    
    server {
        listen       9090;
        server_name  localhost;
    
        #charset koi8-r;
    
        #access_log  logs/host.access.log  main;
    
        location / {
            #root   html;
            #index  index.html index.htm;
            proxy_pass http://myapp;
            proxy_redirect default;
        }
     }
    

    

