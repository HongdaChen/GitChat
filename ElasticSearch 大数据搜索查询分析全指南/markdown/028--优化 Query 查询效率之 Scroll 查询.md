如果涉及大量数据查询的话，一次性取回所有数据显得非常不可靠，一方面 ES 可能会被长时间占用，另一方面在网络连接方面也要一直保持连接状态。

以查询电商商品为例，如果当用户查看商品时候，将所有数据都返回，这将会使得用户等待时间比较长，那么这么体验是非常糟糕的。在处理传统这方面的需求可以通过自定义逻辑实现分页查询，到数据库中分批取数据。ES
同样也支持分页查询。

### 分页查询

**给定需求：**

> 使用分页查询方法查询电子商务订单的订单 ID 与下单日期。

_source 字段里面包括整个文档的所有字段，但是有时候并不是所有的字段都需要，因此使用 includes 过滤。

from 定义了偏移量为多少，size 表示要返回多少数据。下面的查询语句表示从 0 开始查，返回 10 条数据。

    
    
    GET /kibana_sample_data_ecommerce/_search
    {
      "_source": {
        "includes": [
          "order_id",
          "order_date"
        ]
      },
      "from": 0,
      "size": 10
    }
    

查询结果：

    
    
    {
      "took" : 3,
      "timed_out" : false,
      "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
      },
      "hits" : {
        "total" : {
          "value" : 4675,
          "relation" : "eq"
        },
        "max_score" : 1.0,
        "hits" : [
          {
            "_index" : "kibana_sample_data_ecommerce",
            "_type" : "_doc",
            "_id" : "P6ijfm4BzbJ5b0BFXskv",
            "_score" : 1.0,
            "_source" : {
              "order_date" : "2019-12-02T09:28:48+00:00",
              "order_id" : 584677
            }
          },
          ...................
        ]
      }
    }
    

这里会有个疑问，from 最大能设置为多少，才能够把所有数据都查询完。结合前面的课时介绍的关键字 value_count
即可，统计出有多个数据之后就很容易计算出 from 偏移量最大是多少。

    
    
    POST /kibana_sample_data_ecommerce/_search
    {
      "size": 0, 
      "aggs": {
        "count": {
          "value_count": {
            "field": "_id"
          }
        }
      }
    }
    

这种分页查询方法是最常见的，也是很容易理解的，但是它却有个缺点：一是需要首先统计总共有多少个数据，其次是当偏移量越大的时候查询明显变低。因此这种方式只适合数据量不大的情况下，如果数据量很大的情况下这种查询方式就不好了，ES
提供一种 Scroll 查询方式能够解决大数据量环境下的查询。

### Scroll 查询

Scroll 查询与传统数据库中的游标查询方式类似，滚动查询能够应用到分页查询需求中去，Scroll
查询并不是针对用户的实时请求，而是能够处理更大的数据，将一个索引的内容重新索引为具有不同配置的新索引。

使用 Scroll 滚动查询的方式是指定一个 srcoll 参数，并指定游标 scroll_id 的有效时间：

    
    
    POST /class/_search?scroll=10m
    {
      "size": 1
    }
    

从建立 Scroll 查询之后，新的索引已经被建立，一个 Scroll 查询将会返回所有的查询内容，这个查询内容此刻会生成一个快照信息，这个快照过期时间是
10 分钟。在这个时间内有文档被增删改查，那么查询的结果是不会更新的，还是之前的文档未被修改之前的内容。这样看起来数据更像是被缓存起来了。我们来验证一下。

    
    
    {
      "_scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFw4WbG1DSEg0OHlTQ3FGRC1kNUtkRlFOZw==",
      "took" : 67,
      "timed_out" : false,
      "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
      },
      "hits" : {
        "total" : {
          "value" : 8,
          "relation" : "eq"
        },
        "max_score" : 1.0,
        "hits" : [
          {
            "_index" : "class",
            "_type" : "_doc",
            "_id" : "DqksZm8BOMhQf7h0chZh",
            "_score" : 1.0,
            "_source" : {
              "name" : "xiaohong",
              "sex" : "female",
              "age" : 16
            }
          }
        ]
      }
    }
    

首先获取 id 为 5 的文档信息：

    
    
    GET class/_doc/5
    

查询结果如下：

    
    
    {
      "_index" : "class",
      "_type" : "_doc",
      "_id" : "5",
      "_version" : 2,
      "_seq_no" : 35,
      "_primary_term" : 6,
      "found" : true,
      "_source" : {
        "name" : "lanlan deng",
        "sex" : "man",
        "age" : 16
      }
    }
    

我做了更新年龄的操作，把年龄改为 19 岁：

    
    
    POST class/_update/5
    {
      "doc": {
        "age": 19
      }
    }
    

查询结果如下：

    
    
    {
      "_index" : "class",
      "_type" : "_doc",
      "_id" : "5",
      "_version" : 3,
      "result" : "updated",
      "_shards" : {
        "total" : 2,
        "successful" : 2,
        "failed" : 0
      },
      "_seq_no" : 40,
      "_primary_term" : 20
    }
    

最后拿着之前做 Scroll 查询生成的 scroll_id 查询：

    
    
    POST /_search/scroll 
    {
        "scroll" : "10m", 
        "scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFw4WbG1DSEg0OHlTQ3FGRC1kNUtkRlFOZw==" 
    }
    

查询结果显示年龄依旧是 16 岁：

    
    
    {
      "_scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFw4WbG1DSEg0OHlTQ3FGRC1kNUtkRlFOZw==",
      "took" : 22,
      "timed_out" : false,
      "terminated_early" : true,
      "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
      },
      "hits" : {
        "total" : {
          "value" : 8,
          "relation" : "eq"
        },
        "max_score" : 1.0,
        "hits" : [
          {
            "_index" : "class",
            "_type" : "_doc",
            "_id" : "5",
            "_score" : 1.0,
            "_source" : {
              "name" : "lanlan deng",
              "sex" : "man",
              "age" : 16
            }
          }
        ]
      }
    }
    

但是如果采用 from 分页查询的话：

    
    
    POST /class/_search
    {
      "size": 1,
      "from": 7
    }
    

结果是被更新了。

    
    
    {
      "took" : 2,
      "timed_out" : false,
      "_shards" : {
        "total" : 1,
        "successful" : 1,
        "skipped" : 0,
        "failed" : 0
      },
      "hits" : {
        "total" : {
          "value" : 8,
          "relation" : "eq"
        },
        "max_score" : 1.0,
        "hits" : [
          {
            "_index" : "class",
            "_type" : "_doc",
            "_id" : "5",
            "_score" : 1.0,
            "_source" : {
              "name" : "lanlan deng",
              "sex" : "man",
              "age" : 19
            }
          }
        ]
      }
    }
    

### slice 查询

前面说到如果创建一个 Scroll
查询，将会返回所有的文档，并重新建立一个新的索引，如果数据非常的庞大想要获取所有的数据，这个过程显得非常耗时，有没有一种可以并发的方式获取数据？

Scroll 为了解决这个问题，提供了片段查询，也就是说对所有的文档做成多个片段，然后对不同的片段做 Scroll 查询，每个片段都是独立的可并行的。max
指的是最大的片段个数，id 是片段的 id。这样的话就把所有的 class 数据分成两个片段，如下查询所示：

    
    
    GET /class/_search?scroll=1m
    {
        "slice": {
            "id": 0, 
            "max": 2 
        }
    }
    GET /class/_search?scroll=1m
    {
    
        "slice": {
            "id": 1,
            "max": 2
        }
    }
    

### 小结

from 分页查询可以应对实时查询的需求，当时文档被修改后能够立马查出来，但是缺点是如果文档太大查询速度会变慢。

当对数据实时性要求不高时，使用 Scroll 查询大批量数据，数据将会被快照缓存下来，拿着初始 Scroll 查询生成的 scroll_id
去缓存中查询数据，查询速度非常快，缺点是当文档被修改时候容易出现脏数据。Scorll 查询支持片段查询，但是片段的个数也要注意，最好不要超过 ES
分片的个数，不然的话第一查询可能比较慢，虽然之后的查询会变快，但是也要避免这种做法，防止内存被耗尽。

