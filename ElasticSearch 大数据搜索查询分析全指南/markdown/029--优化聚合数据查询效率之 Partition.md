上节课时中介绍了对于 query 查询可以使用 Scroll 来进行优化，聚合数据有时候也会产生大量的数据，本课时将会介绍一下如何优化对聚合数据的查询。

**给定需求：**

> 同一个人在不同城市消费的均值。

对于这个需求要弄明白，保证同一个人就是对这个人进行一次聚合，然后再根据不同城市进行聚合，这种需求就会产生大量的聚合数据，因为人比较多。

下面的语句就是先用 customer_id 作为聚合 key，然后用 geoip.city_name 作为第二次聚合 key。再对
taxless_total_price 取均值。

    
    
    GET /kibana_sample_data_ecommerce/_search
    {
      "size": 0, 
     "aggs": {
       "users": {
         "terms": {
           "field": "customer_id",
           "size": 10
         },"aggs": {
           "city": {
             "terms": {
               "field": "geoip.city_name",
               "size": 10
             },"aggs": {
               "avg": {
                 "avg": {
                   "field": "taxless_total_price"
                 }
               }
             }
           }
         }
       }
     }
    }
    

查询结果如下：

![image-20200513202516016](https://images.gitbook.cn/b91dc370-9596-11ea-9fd5-332242a3cf46)

不难注意到，用户数我设置 size 为 10，城市 size 也是 10，所以最多会返回 100 条聚合数据。但是如何把全部数据都取回来呢？尽量把 size
设置到最大比如说 999999。但是这种作为肯定是最糟糕的，因为你不知道到底有多少用户，如果很多的，这条 ES 语句将会消耗大量的资源，甚至会宕机。

### Partition 查询

为了分批取回所有的聚合数据，可以使用 Partition，分批取首先要知道会有多少条数据、分多少批能取回来，因为我是使用 customer_id 作为
key，所以使用 value_cout 会得到一个比较准备的值，如果 key 不是唯一的：

    
    
    GET /kibana_sample_data_ecommerce/_search
    {
      "size": 0, 
     "aggs": {
       "users": {
         "cardinality": {
           "field": "customer_id"
         }
         }
     }
    }
    

返回结果如下，从返回结果来看共有 46 个用户，那么我们把 batch size 设置为 10，可以把数据分成 5 部分：

![image-20200513203405677](https://images.gitbook.cn/e34f27b0-9596-11ea-
bfb1-9da6a82f9268)

在 users terms 下面 添加 include，num_partitions 是指把数据分成多少分，设置成 5 后，就是把总的数据分成了 5
份，一共有 46 个用户，如果返回所有数据，那么第一层桶将会返回 46 个，现在分批后，只会返回 10 个以内。这个时候 city_name
就可以设置得大一点，返回所有的 city。Partition 是指取第几个部分。

    
    
    GET /kibana_sample_data_ecommerce/_search
    {
      "size": 0,
      "aggs": {
        "users": {
          "terms": {
            "include": {
              "partition": 0,
              "num_partitions": 5
            },
            "field": "customer_id",
            "size": 100
          },
          "aggs": {
            "city": {
              "terms": {
                "field": "geoip.city_name",
                "size": 100000
              },
              "aggs": {
                "avg": {
                  "avg": {
                    "field": "taxless_total_price"
                  }
                }
              }
            }
          }
        }
      }
    }
    

因为 cardinality 统计个数，返回的是一个近似值，所以就可以在返回数据的基础上，再适当增大一点，保证返回的近似 count 大于真实的
count。

不难看出它是有一定缺点的，一个就是 count 返回的不一定准确，自己也不一定能够估计正确，导致 num_partitions
设置得过大；另外一个就是内层的嵌套也可能会有大量的数据，同样会造成数据量过大而导致效率变慢。但是优点也是明显的，就是我们可以用多线程来取数据，只要设置好
Partition 就以用多个线程把所有数据取回来。所以对与 Partition 的选用，前提是你一定要估计好数据量，这样才能尽可能地高效取回所有数据。

### Composite 查询

如果你不知道数据量到底有多少，但是又希望能够正确地取回所有数据，这个时候你可以用 Composite Query。

Composite 相当于 SQL 语句查询中的 `groupby user,city`。同时对两个键进行聚合。

    
    
    GET /kibana_sample_data_ecommerce/_search
    {
      "size": 0,
      "aggs": {
        "users": {
          "composite": {
            "size": 10,
            "sources": [
              {
                "user": {
                  "terms": {
                    "field": "customer_id"
                  }
                }
              },
              {
                "city": {
                  "terms": {
                    "field": "geoip.city_name"
                  }
                }
              }
            ]
          },
          "aggs": {
            "avg": {
              "avg": {
                "field": "taxless_total_price"
              }
            }
          }
        }
      }
    }
    

返回结果如下，可以看到 key 不是嵌套的了，而是在一起了。因为 size 设置为 10 只返回了 10 条数据，另外多了一个 after_key
键，有了这个 after_key 我们就可以对聚合数据进行翻页查询，下一次查询指定 after_key 就可以接着上次查询返回数据。

    
    
    aggregations" : {
        "users" : {
          "after_key" : {
            "user" : "19",
            "city" : "Abu Dhabi"
          },
          "buckets" : [
            {
              "key" : {
                "user" : "10",
                "city" : "Istanbul"
              },
              "doc_count" : 59,
              "avg" : {
                "value" : 66.89790783898304
              }
            },
            ................
    

配置好 after key 后，再次查询，就可以接着上次查询继续翻页查询。

![image-20200513205900921](https://images.gitbook.cn/25e757a0-9597-11ea-898d-b30a8b52ea7c)

优点是不需要估计数据量大小了，但是缺点也明显不能够多线程，只能翻页一页页地查，如果数据量很大，查询时间会比较久。

### 总结

本课时主要介绍如何对聚合数据进行优化查询、多线程的 Partition，以及简单的 Composite 查询，可以根据具体场景选择更适合的。

