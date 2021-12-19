### 基本概念对比

关系型数据库，像最常见的 Mysql , SqlServer 都是属于关系型数据库，关系型数据中有这几种概念：数据库、表、行、列、Schema，还有 SQL
查询语句。

以学生数据库为例，回忆一下数据库过程：

  1. 建学生数据库
  2. 见一个学生表
  3. 配置学生字段，如名称 varchar、性别 char 等
  4. 一行代表一个学生，不同的列代表这个学生不同的属性

ES 也与关系型数据库如出一辙，只是叫法不同，在 ES 中建一个学生数据库步骤如下：

在 ES 中启动一个 ES 实例，这个实例就相当于数据库，表在 ES 中被称为索引 Index，行称为文档 Document，列称为字段 Field
，Schema 被称为 Mapping ，数据库中查询语句 SQL 在 ES 中有相应的 DSL 查询语句。

因此建学生索引：

  1. 配置 ES，启动一个 ES 实例
  2. 新建一个学生索引
  3. 不需要配置字段属性，ES 会自动识别
  4. 一个 JSON 字符串代表一个学生，JSON 字符串中有学生属性字段 Field

具体对应关系如下表：

RDBMS | ES  
---|---  
Table | Index(Type)  
Row | Document  
Column | Field  
Schema | Mapping  
SQL | DSL  
  
### 创建 Table/Index 对比

传统数据创建方式如下，使用简单的 SQL 语句就能够轻松创建，但是前提是要定义好表结构以及数据类型。

    
    
    CREATE TABLE Student(
        name varchar(20),
        sex char(2),
        age int
    );
    

ES 基于 Http 协议的，增删改查的接口都基于 Http ,因此它需要使用 ES 的 Rest API 才能够创建，只需要将 JSON
格式的学生数据利用 Rest API PUT 给 ES 即可自动创建，并且可以选择性的更改 ES 自动创建好的数据字段的类型
，后面的课时中有详细的介绍如何对 ES 进行增删改查。

下面的这个语法，是基于 ES 的可视化界面 Kibana 来做的，使用 PUT 的方式，索引是 student，操作是 _create
创建。这样的话就会自动创建好索引，以及数据字段的类型，比如 name 是 text 类型，age 是 数字类型。

    
    
    PUT student/_create/1
    {
      "name":"xiaoming",
      "sex":"male",
      "age":18
    }
    

### Row/Document 对比

传统数据库中，row 就是一行数据，每行数据是一条记录。查询结果如下：

    
    
    name     | sex  |age 
    xiaoming | male | 18
    

但是在 ES 中数据记录的方式是 documet ，插入数据是将数据看做 documet 然后以 JSON 的格式插入进行，因此查询返回的结果也是
JSON。

    
    
    {
      "took" : 10,
      "timed_out" : false,
      "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
      },
      "hits" : {
        "total" : {
          "value" : 1,
          "relation" : "eq"
        },
        "max_score" : 1.0,
        "hits" : [
          {
            "_index" : "student",
            "_type" : "_doc",
            "_id" : "1",
            "_score" : 1.0,
            "_source" : {
              "name" : "xiaoming",
              "sex" : "male",
              "age" : 18
            }
          }
        ]
      }
    }
    

### Column/ Field 对比

传统数据的话每列代表一个属性，name 列都是都是姓名并且类型是 varchar(20)。

ES 根据字段类型自动创建，会给 name 字段自动定义为 String 类型。

![image-20200219143357472](https://images.gitbook.cn/2020-04-02-025912.png)

**Schema /Mapping 对比**

传统数据库 Schema 说明了表之间的联系结构，字段关系主外键等。

![Figure-1-Student-Database-Schema-The-structure-of-student-database-is-
relational-where](https://images.gitbook.cn/2020-04-02-25913.png)

而 ES 是没有这么多复杂的关系，不存在主外键，表与表之间相互联系，因此 ES 是将一个对象的 JSON 实体直接存到 ES 中，通过 mapping
来查看具体结构。

    
    
    {
      "mapping": {
        "properties": {
          "age": {
            "type": "long"
          },
          "name": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          },
          "sex": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          }
        }
      }
    }
    

### SQL/DSL 对比

两者都是一种语法，SQL 是针对传统数据库的，DSL 是针对 ES 的。在 “创建 Table/Index 对比”
这个部分已经能够很好的说明了两者的区别与联系，语法不通，但是应用场景非常的相似，像增删改查，分组等，两者都具备这样的功能。

### 小结

本节我们明白了两者之间的异同，就能够很好的理解ES各个概念，本课时对传统数据库与 ES
做了一个粗略的对比，只通过本课时想要把握两者的具体与联系是比较困难的，想要了解更多的 ES 特性请多关注后面的实战课时。

