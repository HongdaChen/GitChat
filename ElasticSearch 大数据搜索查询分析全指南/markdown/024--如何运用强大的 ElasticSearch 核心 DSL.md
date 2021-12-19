### DSL 介绍

Query DSL（Domain Specific Language）又叫做查询表达式，是一种丰富的、灵活的、强大的全文本与结构化查询语言，基于 JSON
格式对数据进行检索。前面我们介绍了简单的 ES 查询，但是对于更复杂的需求使用 URL 查询显得有些困难，ES 提供了 DSL 查询，使用 DSL
表达式基本上满足各种复杂的业务需求，DSL 可以与数据库提供的 SQL 进行类比，有了 SQL
各种查询都可以在数据库中实现，而不用把查询计算逻辑放到业务代码中去。

ES 的查询有两种查询与过滤，ES 的查询涉及到一个重要的概念就是相关性分数，这个分数在 ES 查询结果中 _score 字段中体现出来。像你在
Google
或者百度中查询资料时候，往往是以一种推荐的方式返回查询结果给你，并且有顺序的，排在前面的往往是与你查询问题最接近的内容，这里面就涉及到一个相关性分数，查询的每个结果都有一个相关性分数。

另外还有一种查询方式叫做过滤，过滤的查询结果中是不带有相关性分数的，因为有的时候我们只关注有或者没有，显而易见，不带有相关性分数的少了一层计算相关性分数的过程，那么自然计算速度会比较快。

第一个 DSL 查询，查询索引为 class 的所有数据。

    
    
    GET class/_search
    {
      "query": {
        "match_all": {}
      }
    }
    

![image-20200208155610134](https://images.gitbook.cn/2020-04-07-063102.png)

#### **字段含义**

查询结果如下，我们来稍微看一个 ES 返回的查询数据的结构。首先第一眼查看 _score 字段，发现每个字段的值都为
1，那是因为我们查询所有的结果而且没有设置条件，自然每个结果都是我们要的，所以就为 1。

  * **took** 字段代表我们查询这次结果所花费的时候，单位是毫秒，这个字段经常可以用来作为查询 ES 最大连接时长参数设置的一个参考字段。像这次查询花费了 3 毫秒 那么如果我使用 Java 或者 Python 作为查询 ES 的开发语言，就可以把最大连接时长设置为 2 秒钟，如果超过 2 秒就是 time out，当然你可以根据具体需求设置大一点。
  * **timed_out** 字段是指请求是否超时。
  * **_shards** 字段显示查询中参与的分片信息，成功多少分片失败多少分片等。
  * **hits** 字段表示命中数据的意思，一共查询了多少个文档在 total 的 value 中可以看到是 7。在 hits 字段里面还有一个 hits 数组，这个数组就是我们查询到的数据。有 _index 索引信息，_type 是 _doc，还有一个 随机的 _id，表示相关系数的 _score 字段。
  * **_source** 字段里面包围的就是文档信息数据，像 name、sex、age。

    
    
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
          "value" : 7,
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
          .......
          ]
      }
    

在上一篇中我们使用 URI 演示 Term 与 Phrase 查，但是显得比较奇怪而且理解起来也比较费力，在 DSL 将大不相同，书写与理解起来会简单很多。

### Phrase 查询

**给定需求：** 查询班级中名称中包含 deng 的记录。

    
    
    GET class/_search
    {
      "query": {
        "match": {
          "name": "deng"
        }
      }
    }
    

![image-20200208162933983](https://images.gitbook.cn/2020-04-07-063103.png)

从查询结果上来看，把姓 deng 的都找到了，并且返回结果的先后顺序是根据 _score 的大小进行排序。

我们注意到查询的结果包含名称“ziqi 3 Deng”的，以及“ziqi 2 deng”。使用 match 查询，ES
会把查询的字符串，先转小写，然后分词，索引到 ES 中，因此实际上就是把我们的查询条件“ziqi
deng”，分解成两个词，“ziqi”、“deng”，然后到 ES 中查询对比，这个时候就查询到了上面这些结果。这也不难理解，在使用 Google
与百度查询的时候，我们自己输入的条件，也是这样处理的，可以简单地认为是关键词查询。如下我使用百度查询，可以看到红色字是与我的问题匹配的，并且是词匹配。

![image-20200208170718780](https://images.gitbook.cn/2020-04-07-63104.png)

然而我们不仅仅满足于此，想要精确到个人，把叫 ziqi deng 的人查出来。

### Term 查询

**给定需求：** 查询班级中 ziqi deng 的记录。

    
    
    GET class/_search
    {
      "query": {
        "match": {
          "name": "ziqi deng"
        }
      }
    }
    

![image-20200208163317604](https://images.gitbook.cn/2020-04-07-063104.png)

从查询结果中可以看到，返回了好多条数据，ziqi deng、ziqi 2 deng 等，这明显不是我们的需求。

这个时候就需要使用 Term 精确查询，就能把结果查询出来。

    
    
    GET class/_search
    {
      "query": {
        "term": {
          "name.keyword": "ziqi deng"
        }
      }
    }
    

![image-20200208164013397](https://images.gitbook.cn/2020-04-07-063105.png)

细心一点的可以看到，这次把 match 改为 term 之外，另外加了一个 keyword 字段。这里将会详细地解释 name 与 name.keyword
查询到底有什么区别呢？

#### **Text 字段与 keyword 字段**

ES 在存储字符串字段的时候会自动生成两个类型的字段，第一种是 Text 字段，第二种就是 Keyword 字段。这两个字段的区别就是 Text
字段会被分词等其他预处理，Keyword 则就是不会做处理，直接存起来，用来索引。

找到索引 class 的字段类型的 mapping 信息，以 name 字段为例，可以看到 type 为 text 下面有多了一个 keyword。这是
ES 在创建 class 索引时候，自动添加上的。也就是说一个字符串字段，在默认情况下，会以两种方式索引起来，一种是 Text，另一种就是 Keyword。

    
    
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
    

![image-20200209170500905](https://images.gitbook.cn/2020-04-07-063106.png)

比如存储名称为 ziqi deng 这个字符串的时候：

1\. Text 字段会先转小写然后分词，也就是说在 ES 中 是以 ziqi、deng 两个单词存储起来，而 Term 查询表示精确查询，如果不加
keyword，就会到 ES 中查询“ziqi deng”，把“ziqi deng”作为一个单词去查询，然而 ES
中并没有存储这样的单词，所以是查询不到的。

    
    
    GET class/_search
    {
      "query": {
        "term": {
          "name": "ziqi deng"
        }
      }
    }
    

![image-20200208164827275](https://images.gitbook.cn/2020-04-07-63107.png)

2\. 怎么解决这个问题呢？Keyword 类型是没经过处理然后索引下来的，所以这里使用这个字段的 Keyword 类型就能够解决。

这个地方需求你仔细地体会，因为只要使用了 term 查询，那么你给的查询条件就会作为一个整体 到 ES 中查询，你拿着“ziqi deng”这个单词，到
ES 词库中查询自然查询不到这样的单词，然而如果你使用 keyword 类型，那么就会把这个单词跟原始存储的“ziqi
deng”，对比查询，这个时候自然能查询到了。

keyword 仅仅是对字符串数据字段来说的，如果是时间或者数字类型的字段就不需要。

**给定需求：** 查询年龄为 16 的学生信息。

实际上对于非字符串字段，你使用 match 查询也行，也能查出正确的数据，但是 match 会多了很多预处理过程，没有 term 查询快。因此建议使用
Term 查询。

    
    
    GET class/_search
    {
      "query": {
        "term": {
          "age":16
        }
      }
    }
    

![image-20200208171032244](https://images.gitbook.cn/2020-04-07-063107.png)

### 范围查询

范围查询也是经常遇到的需求，时间、数字等。

**给定需求：** 查询年龄在 10 岁到 16 岁之间的学生。

使用 range 关键字，这里的操作符主要有四个：gt 大于、gte 大于等于、lt 小于、lte 小于等于。

    
    
    GET class/_search
    {
      "query": {
        "range":{
          "age":{
            "gte": 10,
            "lte": 16
          }
        }
      }
    }
    

![image-20200208171840411](https://images.gitbook.cn/2020-04-07-063109.png)

时间返回查询同样也是一个重要的查询方式，由于这个样例数据中没有包含时间字段所以简单地给出查询方式，时间戳查询与时间字符串查询，不难看出时间字符串查询需要指定时区信息，“+08:00”表示东八时区，在这里我建议在开发过程中使用时间戳查询，不要使用时间字符串查询，因为时间戳查询不涉及时区的问题。在工作中各个国家的都会产生
ES
数据，我在查询过程中不可能根据不同的国家设置好时区后再查询，最简单的方式就是使用时间戳，另外在开发过程中，时间戳的获取要比具体格式时间的字符串获取，要简单太多。

    
    
    GET _search
    {
      "query": {
        "range": {
          "@timestamp": {
            "gte": 1557676800000,
            "lte": 1557680400000,
            "format":"epoch_millis"
          }
        }
      }
    }
    GET _search
    {
      "query": {
        "range":{
          "@timestamp":{
            "gte": "2020-02-10 18:30:00",
            "lte": "2020-05-14",
            "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd",
            "time_zone": "+08:00"
          }
        }
      }
    }
    

### 小结

本篇详细介绍了 DSL 基本使用方法，但是重点需要理解的还是 Term 查询与 Match 查询：Term 查询不会对查询条件进行分词预处理，但是
match 查询就会。选择使用 Term 还是 Match，这要取决于你是否想要对查询条件做处理。选择好了后，你还要抉择去哪去对比索引，是去 Text 还是
Keyword。

