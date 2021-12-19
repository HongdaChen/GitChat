第四部分（第 30 ~ 34 篇）这部分内容将为大家详细讲解非关系型数据库 MongoDB 的安装及使用，以及 Spring 全家桶的整合方案 Spring
Data MongoDB 的使用，同时完成本套课程的第 2 个项目案例，使用 Spring MVC + layui + Spring Data
MongoDB 实现权限管理系统。

（1）谈谈你对 MongoDB 的理解？

**点击查看答案**

作为主流的非关系型数据库（NoSQL）产品，MongoDB 很好的实现了面向对象的思想，在 MongoDB 中每一条记录都是一个 Document
对象。MongoDB 最大的优势在于所有的数据持久操作都无需开发人员手动编写 SQL 语句，直接调用方法就可以轻松实现 CRUD 操作。

    

（2）MongoDB 有哪些特点？

**点击查看答案**

  * 高性能、易使用，存储数据非常方便
  * 面向集合存储，易存储对象类型的数据
  * 支持动态查询
  * 支持完全索引，包含内部对象
  * 支持复制和故障恢复
  * 使用高效的二进制数据存储，包括大型对象（如视频等）
  * 自动处理碎片，以支持云计算层次的扩展性
  * 支持 Python、PHP、Ruby、Java、C、C#、JavaScript、Perl 及 C++ 语言的驱动程序，社区中也提供了对 Erlang 及 .NET 等平台的驱动程序
  * 文件存储格式为 BSON（一种 JSON 的扩展） 

    

（3）MongoDB 都有哪些主要功能？

**点击查看答案**

  * 面向集合的存储：适合存储对象及 JSON 形式的数据。 
  * 动态查询：MongoDB 支持丰富的查询表达式，查询指令使用 JSON 形式的标记，可轻易查询文档中内嵌的对象及数组。 
  * 完整的索引支持：包括文档内嵌对象及数组，MongoDB 的查询优化器会分析查询表达式，并生成一个高效的查询计划。 
  * 查询监视：MongoDB 包含一个监视工具用于分析数据库操作的性能。 
  * 复制及自动故障转移：MongoDB 数据库支持服务器之间的数据复制，支持主从模式及服务器之间的相互复制，复制的主要目标是提供冗余及自动故障转移。 
  * 高效的传统存储方式：支持二进制数据及大型对象（如照片或图片）。
  * 自动分片以支持云级别的伸缩性：自动分片功能支持水平的数据库集群，可动态添加额外的机器。 

    

（4）说说你知道的 MongoDB 适用场景。

**点击查看答案**

  * 网站数据：MongoDB 非常适合实时的插入、更新与查询，并具备网站实时数据存储所需的复制及高度伸缩性。 
  * 缓存：由于性能很高，MongoDB 也适合作为信息基础设施的缓存层。在系统重启之后，由 MongoDB 搭建的持久化缓存层可以避免下层的数据源过载。 
  * 大尺寸、低价值的数据：使用传统的关系型数据库存储一些数据时可能会比较昂贵，在此之前，很多时候程序员往往会选择传统的文件进行存储。 
  * 高伸缩性的场景：MongoDB 非常适合由数十或数百台服务器组成的数据库，MongoDB 的路线图中已经包含对 MapReduce 引擎的内置支持。 
  * 用于对象及 JSON 数据的存储：MongoDB 的 BSON 数据格式非常适合文档化格式的存储及查询。

    

（5）关闭 MongoDB 服务的命令是？

**点击查看答案**

    
    
    use admin
    db.shutdownServer()
    

    

（6）下列哪个选项是 MongoDB 创建数据库的命令？（单选题）

A. use testdb

B. create testdb

C. use database testdb

D. create database testdb

**点击查看答案**

A

    

（7）谈谈你对 Spring Data JPA 的理解？

**点击查看答案**

  * JPA（Java Persistence API）是 Sun 官方提出的 Java 持久化规范，它为 Java 开发人员提供了一种对象 / 关联映射工具来管理 Java 应用中的关系数据。它的出现主要是为了简化现有的持久化开发工作和整合 ORM 技术，结束现在 Hibernate、TopLink、JDO 等 ORM 框架各自为营的局面。JPA 是在充分吸收了现有的 Hibernate、TopLink、JDO 等 ORM 框架的基础上发展而来的，具有易于使用、伸缩性强等优点。
  * Spring Data JPA 是 Spring 基于 ORM 框架、JPA 规范的基础上封装的一套 JPA 应用框架，可以让开发者用极简的代码即可实现对数据的访问和操作。它提供了包括增、删、改、查等在内的常用功能，且易于扩展，学习并使用 Spring Data JPA 可以极大提高开发效率。Spring Data JPA 其实就是 Spring 基于 Hibernate 之上构建的 JPA 使用解决方案，方便在 Spring Boot 项目中使用 JPA 技术。

    

（8）Spring Data JPA 删除多条记录并返回的代码是？

**点击查看答案**

    
    
    List<Student> students =  mongoTemplate.findAllAndRemove(
      Query.query(new Criteria("age").is(19)), 
      Student.class);
    

    

（9）谈谈 Spring Data JPA 的底层实现。

**点击查看答案**

Spring Data JPA 底层默认使用 Hibernate 框架实现，Spring Data JPA 的首个接口就是
Repository，它是一个标记接口，开发者自定义接口只需继承这个接口，就可以直接使用 Spring Data JPA
了，同时可以按照一定的命名规范来自定义方法，并且 Spring Data JPA 会自动实现这些方法，开发者直接调用即可。

    

（10）Spring AOP 的原理是什么？都有哪些具体的应用场景？

**点击查看答案**

  * AOP：Aspect Oriented Programming 面向切面编程，用来封装横切关注点，具体可以在下面的场景中使用：Authentication 权限、Caching 缓存、Context passing 内容传递、Error handling 错误处理 Lazy loading 懒加载、Debugging 调试、logging、tracing、profiling and monitoring 记录跟踪优化校准、Performance optimization 性能优化、Persistence 持久化、Resource pooling 资源池、Synchronization 同步、Transactions 事务。
  * 原理：AOP 是面向切面编程，是通过动态代理的方式为程序添加统一功能，集中解决一些公共问题。
  * 优点：1）各个步骤之间的良好隔离性使得耦合性大大降低；2）源代码无关性，扩展功能的同时不对源码进行修改操作。

    

