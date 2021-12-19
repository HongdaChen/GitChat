### 1 Elasticsearch Cache 机制

#### 1.1 Cache 类型

Elasticsearch 内部包含三个类型的读缓冲，分别为 **Node Query Cache** 、 **Shard Request Cache**
以及 **Fielddata Cache** 。

**1\. Node Query Cache**

Elasticsearch 集群中的每个节点包含一个 Node Query Cache，由该节点的所有 shard 共享。该 Cache 采用 LRU
算法，Node Query Cache 只缓存 filter 的查询结果。

**2\. Shard Request Cache**

Shard Request Cache 缓存每个分片上的查询结果跟 Node Query Cache 一样，同样采用 LRU 算法。默认情况下，Shard
Request Cache 只会缓存设置了 `size=0` 的查询对应的结果，并不会缓存 hits，但是会缓存命中总数，aggregations，and
suggestions。

有一点需要注意的是，Shard Request Cache 把整个查询 JSON 串作为缓存的 key，如果 JSON
对象的顺序发生了变化，也不会在缓存中命中。所以在业务代码中要保证生成的 JSON 是一致的，目前大部分 JSON 开发库都支持 canonical 模式。

**3\. Fielddata Cache**

Elasticsearch 从 2.0 开始，默认在非 text 字段开启 `doc_values`，基于 `doc_values`
做排序和聚合，可以极大降低节点的内存消耗，减少节点 OOM 的概率，性能上损失却不多。

5.0 开始，text 字段默认关闭了 Fielddata 功能，由于 text 字段是经过分词的，在其上进行排序和聚合通常得不到预期的结果。所以我们建议
Fielddata Cache 应当只用于 global ordinals。

#### 1.2 Cache 失效

不同的 Cache 失效策略不同，下面分别介绍：

**1\. Node Query Cache**

Node Query Cache 在 segment 级缓存命中的结果，当 segment 被合并后，缓存会失效。

**2\. Shard Request Cache**

每次 shard 数据发生变化后，在分片 refresh 时，Shard Request Cache 会失效，如果 shard
对应的数据频繁发生变化，该缓存的效率会很差。

**3\. Fielddata Cache**

Fielddata Cache 失效机制和 Node Query Cache 失效机制完全相同，当 segment 被合并后，才会失效。

#### 1.3 手动清除 Cache

Elasticsearch 提供手动清除 Cache 的接口：

    
    
    POST /myindex/_cache/clear?query=true      
    POST /myindex/_cache/clear?request=true    
    POST /myindex/_cache/clear?fielddata=true   
    

Cache 对查询性能很重要，不建议在生产环境中进行手动清除
Cache。这些接口一般在进行性能压测时使用，在每轮测试开始前清除缓存，减少缓存对测试准确性的影响。

### 2 Cache 大小设置

#### 2.1 关键参数

下面几个参数可以控制各个类型的 Cache 占用的内存大小。

  * `indices.queries.cache.size`：控制 Node Query Cache 占用的内存大小，默认值为堆内存的10%。
  * `index.queries.cache.enabled`：索引级别的设置，是否启用 query cache，默认启用。
  * `indices.requests.cache.size`：控制 Shard Request Cache 占用的内存大小，默认为堆内存的 1%。
  * `indices.fielddata.cache.size`：控制 Fielddata Cache 占用的内存，默认值为unbounded。

#### 2.2 Cache 效率分析

要想合理调整上面提到的几个参数，首先要了解当前集群的 Cache 使用情况，Elasticsearch 提供了多个接口来获取当前集群中每个节点的 Cache
使用情况。

**cat api**

    
    
    # curl -sXGET 'http://localhost:9200/_cat/nodes?v&h=name,queryCacheMemory,queryCacheEvictions,requestCacheMemory,requestCacheHitCount,request_cache.miss_count'
    

得到如下结果，可以获取每个节点的 Cache 使用情况：

    
    
    name queryCacheMemory queryCacheEvictions requestCacheMemory requestCacheHitCount request_cache.miss_count
    test01 1.6gb 52009098 15.9mb 1469672533 205589258
    test02 1.6gb 52196513 12.2mb 2052084507 288623357
    

**nodes_stats**

    
    
    curl -sXGET 'http://localhost:9200/_nodes/stats/indices?pretty'
    

从结果中可以分别找到 Query Cache、Request Cache、Fielddata 相关统计信息

    
    
    ...
    "query_cache" : {
         "memory_size_in_bytes" : 1736567488,
         "total_count" : 14600775788,
         "hit_count" : 9429016073,
         "miss_count" : 5171759715,
         "cache_size" : 292327,
         "cache_count" : 52298914,
         "evictions" : 52006587
    }
    ...
    "fielddata" : {
         "memory_size_in_bytes" : 186953184,
         "evictions" : 0
    }
    ...
    "request_cache" : {
         "memory_size_in_bytes" : 16369709,
         "evictions" : 307303,
         "hit_count" : 1469518738,
         "miss_count" : 205558017
    }
    ...
    

#### 2.3 设置 Cache 大小

在收集了集群中节点的 Cache 内存占用大小、命中次数、驱逐次数后，就可以根据收集的数据计算出命中率和驱逐比例。

过低的命中率和过高的驱逐比例说明对应 Cache 设置的过小。合理的调整对应的参数，使命中率和驱逐比例处于期望的范围。但是增大 Cache 要考虑到对 GC
的压力。

### 总结

Elasticsearch 并不会缓存每一个查询结果，他只缓存特定的查询方式，如果你增大了 Cache 大小，一定要关注 JVM
的使用率是否在合理的范围，我们建议保持在 60% 以下比较安全，同时关注 GC 指标是否存在异常。

下一节我们介绍一下 **如何使用熔断器（breaker）来保护 Elasticsearch 节点的内存使用率** 。

* * *

### 交流与答疑

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《高可用 Elasticsearch 集群 21 讲》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「244」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

