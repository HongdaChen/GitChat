前面的章节主要介绍 ES 的基本概念，以及如何使用 ES+Kibana 进行业务数据的分析挖掘。本章节将涉及到如何使用 ES
对数据进行增删改查，传统数据库第一步就是创建数据库，然后创建表，再基于表对数据进行增删改查，ES 也是如此。前面也对传统数据库与 ES
做了简单的对比，数据库 == ES 实例，数据库表 == ES 索引，数据库 schema==mapping。

7.0 之前的 ES 是这么一种格式， **索引/类型** ，如果把索引看做是数据库，那么类型就是数据库表，你可以这么定义
**class/student/1** ，表示班级索引、学生类型、学号为 1。现在 7.0 之后类型统一为 **_doc** 。

![image-20191230215848181](https://images.gitbook.cn/2020-04-07-063345.png)

**Index 操作**

我们现在创建一个班级索引，并插入一条学生的数据。值得注意的是我框选出来的，首先是 type 我改成 **_doc** ，否则将无法创建索引。其次是 id
号为 1，版本号 _version 也是 1，result 是 created。如果我再执行一次这个命令：

![image-20191230220329726](https://images.gitbook.cn/2020-04-07-063348.png)

是的，版本号加 1，另外显示 result 为 **updated** 。也就是说，在使用 PUT
作为创建索引添加数据的时候，会检查是否存在相同的文档，如果存在则删除，并且创建新的文档，另外版本号加一。

![image-20191230220657755](https://images.gitbook.cn/2020-04-07-063350.png)

我想要像传统数据库那样，指定 id 为主键后，再插入数据如果有相同的 id 则无法插入，明显 index 操作无法满足这种需求。

**Create 操作**

PUT 提供了另外一个 REST API 来解决这个问题——create。直接执行将会报错，像下图那样。如果想要创建唯一个文档，可以考虑使用 create
命令。

    
    
    PUT class/_create/1
    {
      "name":"xiaoming",
      "sex":"man",
      "age":16
    }
    

![image-20191230221455223](https://images.gitbook.cn/2020-04-07-063354.png)

在传统数据库中，可以设置自增的 id，在 ES 中虽然不能够设置自增 id，但是可以设置自动生成 id，只需要使用 POST 接口。虽然没有指定
id，但是系统自动生成了唯一的 id。像 ES 中存数据，要有个顺序，首先是指定索引，索引指定后指定 Type，然后才是 id，id 不指定也行，但是要使用
POST 接口（Index->Type->id）。

    
    
    POST class/_doc
    {
      "name":"xiaohong",
      "sex":"female",
      "age":16
    }
    

![image-20200102201548932](https://images.gitbook.cn/2020-04-07-063356.png)

当我们创建好一个文档后，返回的数据如下，从返回的数据可以了解到，索引 _index 是 class，类型 _type 是 _doc，_id 是
DqksZm8BOMhQf7h0chZh。另外，可以看到分片信息，一共有两个主分片。

    
    
    {
      "_index" : "class",
      "_type" : "_doc",
      "_id" : "DqksZm8BOMhQf7h0chZh",
      "_version" : 1,
      "result" : "created",
      "_shards" : {
        "total" : 2,
        "successful" : 2,
        "failed" : 0
      },
      "_seq_no" : 3,
      "_primary_term" : 3
    }
    

我们在创建 calss 的时候，虽然没有配置分片信息，但是已经系统给定默认的配置，我们也可以自定义配置方法。我配置了 2
个主分片，并为每个分片配置一个副本。

    
    
    PUT /class1
    {
      "settings": {
        "number_of_replicas": 1,
        "number_of_shards": 2
      }
    }
    

![image-20200102202701063](https://images.gitbook.cn/2020-04-07-063400.png)

我在索引 class1 中创建一个文档，可以看到分片信息。

![image-20200102203212542](https://images.gitbook.cn/2020-04-07-063402.png)

我们知道了在 create
过程中可以配置分片信息，那么如果随着数据量的增大，我们觉得两个主分片不能够很好地工作，想要把主分片扩展一下呢？不幸的是，如果我再次为索引 class1
配置主分片个数为 3
的时候，这个时候已经提示错误，无法进行扩展，这也就是说在创建索引之前，自己一定要业务数据的规模规划好，分片数。因此创建完之后将无法更改。

![image-20200102203813524](https://images.gitbook.cn/2020-04-07-063403.png)

**Update 操作**

如果想要更新某个文档的数据，比如说把小明的年龄改成 18 岁，如何做呢？首先你可以选择使用索引 index
更改，但是前面我们说过，索引操作将会删除已经存在的数据，并且 version 加一。如下图，我将 ID 为 1 的小明年龄改成
18，然后再查下小明信息的时候，发现文档中只有年龄信息、姓名、性别信息全都被删除了。因此很明显 Index 操作不适合这个需求，ES 提供了 Update
接口。Update 接口要更新某个文档首先要求的是这个文档必须存在，要不然将会更新失败。

    
    
    PUT class/_doc/1
    {
      "age":18
    }
    GET class/_doc/1
    

![image-20200102205140970](https://images.gitbook.cn/2020-04-07-063404.png)

首先我重新把小明的信息存回去，然后使用 _update 接口只更改年龄，然后查看文档 id 为 1 的小明信息，发现年龄已经变成 18 岁，并且
version 版本增加 1。

    
    
    PUT class/_doc/1
    {
      "name":"xiaoming",
      "sex":"man",
      "age":16
    }
    POST class/_update/1
    {
      "doc": {
        "age": 19
      }
    }
    GET class/_doc/1
    

![image-20200102205548105](https://images.gitbook.cn/2020-04-07-063406.png)

**GET 操作**

GET 是查询的一个接口，上面的介绍中也有展示。使用 GET 命令，指定索引/类型/ID 就可以查询到相应的文档。我们存的数据都被包裹在 _source
字段中。

    
    
    GET class/_doc/1
    

![image-20200102210838648](https://images.gitbook.cn/2020-04-07-063407.png)

**Bulk 批量操作**

对 ES 的操作都是基于 Rest API，当需要批量插入文档的时候，一条一条的插入明显对网络 IO 不友好，ES 提供了一个 Bulk
API，能够支持批量插入数据，减少网络 IO。

Bulk 支持以下四种操作：

  * Index
  * create
  * update
  * delete

如下使用 Bulk API
可以一次请求，进行删除、索引、创建、更新操作，不仅仅如此还可以对不同的索引进行操作。需要注意的是数据格式要在同一行，否则将会报错。

    
    
    PUT _bulk
    {"delete":{"_index":"class","_id":"1"}}
    { "index":{"_index":"class","_id":"1"}}
    {"name":"xiaoming", "sex":"man","age":16}
    {"create":{"_index":"class","_id":"2"}}
    {"name":"xiaohong","sex":"female", "age":18}
    {"update":{"_id":"1","_index":"class"}}
    {"doc":{"age":20}}
    

![image-20200103212810178](https://images.gitbook.cn/2020-04-07-063410.png)

**Mget 批量读**

根据 id 查询文档，一次访问只能查询一条记录，使用 Mget API 可以批量根据 id 查询文档。如下我使用 Mget 查询索引为 class 的 id
为 1 与 2 的两个文档。

    
    
    GET _mget
    {
      "docs": [
        {
          "_index": "class",
          "_id": 1
        },
        {
          "_index": "class",
          "_id": "2"
        }
      ]
    }
    

![image-20200103215019967](https://images.gitbook.cn/2020-04-07-063411.png)

**小结：** 通过本课时已经了解如何使用 ES 进行增删改，以及简单的查询，以及为了优化性能介绍了批量操作 Bulk，以及批量读 Mget 命令。

