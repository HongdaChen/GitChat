Elasticsearch 7版本发布已经有一段时间，按照官方的说法，他是迄今为止最快、最安全、最有弹性、最容易使用的 Elasticsearch
版本，下面我们看一下这个版本的重大变更及新特性，并分享我们的一些测试结论。

### 基于 Lucene 8.0

Elasticsearch 7 基于 Lucene 8.0.0，在索引的兼容性上，他可以直接加载 Elasticsearch
6.0以上的版本创建的索引，Elasticsearch 5.x 创建的索引需要 reindex 到 Elasticsearch 7.x

#### TOP-K 优化

Lucene 8.0.0做了大量的新能优化，主要亮点是 TOP-K 的查询优化。在之前的版本中，查询会计算所有命中的文档，但是用户经常查询 'a' ,
'the' 等词汇，这种词汇不会增加多少文档得分，但迫使查询过程为大量的文档进行打分。

因此，如果检索结果只需要返回 TOP-K 的结果，而非范围准确的命中数量，可以对此进行优化，Lucene 8 中引入了 WAND
算法来实现此特性。当检索结果小于指定的结果总数时，该优化不会生效。

在停止计算命中文档总数之后，查询 QPS 得到大幅提升，以下结果来自 [lucene
官方基准测试](http://people.apache.org/~mikemccand/lucenebench/)

Bool AND 查询，提升 2.3 倍左右。 ![enter image description
here](https://images.gitbook.cn/0db3ce90-b9a9-11e9-ad84-d52f8a9d7052)

Bool OR 查询，提升 2.5 倍左右。 ![enter image description
here](https://images.gitbook.cn/1e73fc00-b9a9-11e9-ad84-d52f8a9d7052)

Term 查询，提升 40 倍左右。 ![enter image description
here](https://images.gitbook.cn/2d448e70-b9a9-11e9-b261-53935f63522b)

在 Elasticsearch 7中，要在查询中返回 TOP-K 的结果，通过 track _total_ hits
参数来指定，默认值为10000，根据自己的需要设置返回前 K 个命中结果，或者设置为 true，返回全部命中结果数量。例如：

    
    
    curl -X GET "localhost:9200/twitter/_search" -H 'Content-Type: application/json' -d'
    {
        "track_total_hits": 100,
         "query": {
            "match" : {
                "message" : "Elasticsearch"
            }
         }
    }
    '
    

计算 TOP-K 的过程中需要评估文档的最大得分，这需要在索引过程中写入一些额外的信息。Lucene 将词典划分一个个的
block，并构建了一个跳跃表，在查询的时候跳过不匹配的文档，现在，索引过程中会为每个块中最高影响(impacts）的摘要添加到该跳表中，可以计算出该块可能产生的最大得分，如果该得分不具有竞争力，则可以跳过它。更多信息可以阅读[此处](https://www.elastic.co/blog/faster-
retrieval-of-top-hits-in-elasticsearch-with-block-max-wand)

#### 词典支持 off-heap 方式加载

Lucene8 现在支持以 mmap 的方式加载词典索引，倒排表和 Docvalues，节约常驻 JVM 内存，由于使用 mmap
方式使用对外内存，可能会被置换到磁盘上，因此可能引发从磁盘读取，所以查找词典会稍微慢一些，不过查找一个 term 只是查询过程的一小部分，这影响不大。

但是对查找主键（docid）影响很大，
这在更新索引时会用到。更多信息可以参考[此处](https://issues.apache.org/jira/browse/LUCENE-8635)

除此TOP-K 优化之外，Lucene8 对 DocValues 的随机访问性能也进行了优化，以及更快的自定义评分。关于
Lucene8的新特性和优化项的完整描述参阅[此处](http://lucene.apache.org/core/8_0_0/changes/Changes.html#v8.0.0.optimizations)

我们使用 7.1.1 与 6.8 版本进行对比测试，在默认配置下，并没有发现7.x版本常驻JVM内存有明显的降低。

> 测试过程中向 Elasticsearch 写入5TB 索引，然后通过 REST 接口查看 segment 内存占用9.3GB。平均1
> TB索引占用约2GB内存（内存占用和具体的业务数据相关）。

我们尝试修改 lucene 文件的加载方式，让tip（FST 保存在 tip
文件）也采用mmap方式加载，将索引的store类型设置成mmapfs，对比内存变化，Lucene segments占用内存大概有15%的降低。

### 增强弹性和稳定性

#### 新的集群协调子系统

Elasticsearch 6.x 及之前的版本使用名为 Zen Discovery 的集群协调子系统，这个系统已经比较成熟，但是存在一些缺点。Zen
Discovery 通过用户配置 discovery.zen.minimum _master_ nodes
来明确指定多少个符合主节点条件的节点可以形成法定数量，但是在集群扩容时，用户可能会忘记调整。其次， Zen Discovery的选主过程也有些慢。

Elasticsearch 7重新设计了集群协调子系统，移除了minimum _master_
nodes设置，由集群自己选择可以形成法定数量的节点。并且新的子系统可以在很短时间内完成选主。

要使用新的协调子系统，需要以下步骤（从6.x升级除外）：

**1\. 配置集群引导**

如果使用默认配置启动节点，他会自动查找本机的其他节点，并形成集群。如果在生产环境，要启动一个全新的集群时，必须指定集群初次选举中用到的具有主节点资格的节点，称为集群引导，这只在第一次形成集群时需要，例如下面的配置：

    
    
    cluster.initial_master_nodes:
      - master-a
      - master-b
      - master-c
    

注意：

  1. initial _master_ nodes配置只在集群首次启动时使用，后续将忽略此配置
  2. 不具备主节点资格的节点，以及新节点加入现有集群，无需配置 initial _master_ nodes
  3. 各个节点配置的initial _master_ nodes值应该相同

一定要小心的配置 initial _master_ nodes，否则集群可能脑裂。

**2\. 配置节点发现**

节点发现需要配置一些种子节点，这个概念和原来配置的 discovery.zen.ping.unicast.hosts 类似。例如：

    
    
    discovery.seed_hosts:
       - 192.168.1.10:9300
       - 192.168.1.11 
       - seeds.mydomain.com 
    

#### 真实的内存断路器

Elasticsearch 6.x 及之前版本的circuit breaker估算内存用量时与实际内存占用误差很大，因此circuit
breaker实际上很难发挥作用。Elasticsearch 7版本使用
jvmMemoryPoolMXBean提供的getUsage()获取内存使用情况，实现了比较准确的circuit
breaker，可以通过indices.breaker.total.use _real_ memory配置是否启用，默认为 true

如果说 ES 比较大的缺点，节点稳定性是其中之一，线上经常因为各种请求导致节点挂掉，例如深度聚合、以及巨大的 bulk
等，但是我们希望任何请求到达节点时可以失败，但是不能引起节点或集群异常。因此为了维护节点稳定性，Elasticsearch 6.x
之前需要自己去控制或过滤业务发起的请求，新版本的circuit breaker是一个很重要的更新。经过初步测试，新版本的circuit
breaker确实可以准确地根据内存用量去拒绝客户端请求，不过由于节点经常产生大量临时对象，JVM
占用情况可能会在短期内飙升，并在稍后恢复，因此需要为断路器设置较高一些的阈值。

#### 限制 aggregation bucket数量

Elasticsearch 6.x以前的版本中，对聚合结果bucket的数量是不限制的，这容易在深度聚合时产生
OOM。为了保护节点稳定性，7.x开始添加配置项search.max_buckets，默认值为10000。

#### 限制每个节点打开的分片数量

Elasticsearch 6.x 及之前的版本中，每个节点可以拥有的分片数量没有上线。新版本中，添加cluster.max _shards_
per_node设置，控制节点的分片总数，默认值为1000。

close 状态索引持有的分片不会被计算在内，但是unassigned状态的分片会被计算进去。判断分片数量是否超过阈值时是以集群当前总分片数与max
_shards_ per_node*节点数来比较：

    
    
    int maxShardsInCluster = maxShardsPerNode * nodeCount;
    if ((currentOpenShards + newShards) > maxShardsInCluster) {
    }
    

### 性能优化

#### 默认启用 ARS

自适应副本选择（ARS）在6.x 中实现，但默认关闭，现在他已默认开启。未启用 ARS
的情况下，查询请求在分片的多个副本之间轮询执行，但是可能某个节点的负载较大，ARS 会选择负载较小的节点来转发请求。这类似负载均衡器中智能路由的概念。

配置项：cluster.routing.use _adaptive_ replica_selection，默认true

#### 搜索空闲时跳过 refresh

以前的 refresh 策略为定期 refresh，默认1秒。过多的 refresh 会产生过多 lucene 分段，导致后期 merge
产生较大压力。在新的refresh策略中，某个 shard 在30秒内没有查询请求时，被标记为search idle，跳过周期的
refresh。当一个搜索请求到来时，会先对search idle的分片执行 refresh，保证数据对搜索的可见性。

如果设置了 refresh_interval，则此新策略不生效，将按照原来的周期性 refresh 执行。

配置项：index.search.idle.after，默认值 30s

#### 默认 1 个分片

Elasticsearch 遇到比较多的问题是集群分片太多，由于很多场景下索引是按周期生成的，默认的5个分片有些多，现在调整为1个。

#### 跨集群搜索（CCS）降低了请求往返次数

CCS 增加了 ccs _minimize_ roundtrips 模式，该模式降低了不必要的请求，降低了搜索延迟。

#### index.store.type 增加 hybridfs 类型

index.store.type 的默认类型为 fs，他会依据环境自动选择存储类型，目前所有受支持系统上的操作环境都是
hybridfs，但也有可能不同。Elasticsearch 6.x 及之前的版本会选择 mmap 类型。

hybridfs 类型是 niofs 和 mmapfs 的混合实现，它根据读取访问模式为每种类型的文件选择最佳的类型。目前，只有 term
dictionary, norms 和 doc values 文件使用 mmap 方式。所有其他文件都使用 Lucene NIOFSDirectory
打开。与 mmapfs 类似，hybridfs 类型需要确保您配置了正确的 vm.max _map_ count

### 使用更简便

#### 内置 JAVA

由于一些小白用户不知道 Elasticsearch 是一个 java 程序，新版本会将 JDK 一起打包。如果你设置了JAVA_HOME
环境变量，则使用你指定的 JDK。Elasticsearch 7打包的 JDK 为 OpenJDK 12.0.1

#### 索引生命周期管理( ILM) 做为正式功能

在日志等场景中，索引会按周期创建，写入，和删除，一般情况下使用 Elasticsearch 的团队会写一些脚本由 crontab
驱动来完成周期性的操作。Elasticsearch 从 6.6
开始内置了这些功能（beta），将索引的生命周期划分为：Hot、Warm、Cold、Delete。ILM 很大程度上降低了 Elasticsearch
的使用门槛，这个功能从 Elasticsearch 7开始称为正式特性。

#### SQL 转正

Elasticsearch 从 6.3 开始加入 SQL 接口，该特性在 7.x 成为正式功能。

#### 日志输出支持 json 格式

没啥好说的

#### java high-level REST client 已具备完整的 API

官方一直建议将 TransportClient 更换为 REST Client，现在，high-level REST client已具备完整的 API，使用
TransportClient 的客户端现在可以开始迁移。

### 功能变化

  * 移除了 type 的概念。从6.x 开始为移除 type 已经只支持一个 type，现在正式移除。
  * 集群名称和索引名称不允许包括冒号 :
  * `_all` `_uid` 两个元字段被删除
  * mappings 中的 `_default_` 被删除
  * 限制了 term query 中 terms 的最大值为 65536
  * `max_concurrent_shard_request` 语义更改，由原来的限制单个搜索请求并发分片请求的总数，变成每个结点最大并发分片请求数； 也就是说之前单个查询涉及n个分片，同时并行请求的数量不会超过该设置，不关心有多少节点参与到这一批查询；修改后，会将查询平摊到所有节点，每个节点查询的分片数不超过该值。
  * fielddata circuit breaker 默认配置由 60% 降低到 30%
  * 移除了 tribe node（部落节点），使用跨集群查询代替。
  * `index.unassigned.node_left.delayed_timeout` 不能设置成负数
  * 为了防止OOM，单个文档内嵌套 json 对象的数量被限制为10000个。可以通过索引设置`index.mapping.nested_objects.limit` 更改此默认限制。
  * 节点名默认为hostname, 之前为node.id的前8个字符
  * 去掉index thread pool, 单个文档 index 也使用 bulk thread pool，同时删除了`thread_pool.index.size`、 `thread_pool.index.queue_size` 两个配置
  * query_string 查询不再支持 `use_dismax`, `split_on_whitespace`, `all_fields` and `lowercase_expanded_terms`
  * 分片优先级去掉了 `_primary`, `_primary_first`, `_replica`, and `_replica_first`
  * 查询响应 hits.total 改成对象

### 索引性能的差异

在入库性能方面（写入速度），基于 Elasticsearch 7.1.1 进行写入压力测试，验证与 Elasticsearch
6.8版本的差异，通过3轮测试对比，在同样的环境下，Elasticsearch 7.1.1 入库性能降低约20%，这可能是受 TOP-
K优化的影响，他需要在写入过程中进行更多的计算。

### 参考资料

[Elasticsearch 7.0.0
released](https://www.elastic.co/cn/blog/elasticsearch-7-0-0-released)

[Breaking changes in
7.0](https://www.elastic.co/guide/en/elasticsearch/reference/current/breaking-
changes-7.1.html)

[Elasticsearch 7.0.0 Beta 1
Released](https://www.elastic.co/cn/blog/elasticsearch-7-0-0-beta1-released)

[What's new in Lucene 8](https://www.elastic.co/cn/blog/whats-new-in-lucene-8)

[Elasticsearch 集群协调迎来新时代](https://www.elastic.co/cn/blog/a-new-era-for-
cluster-coordination-in-elasticsearch)

[EFFICIENT TOP-K QUERY PROCESSING IN LUCENE
8](http://mocobeta.github.io/slides-html/search-tech-talk-1/search-tech-
talk-1.html)

[Lucene 8的Top-k查询处理优化（1）简介](https://medium.com/@mocobeta/lucene-8-%E3%81%AE-
top-k-%E3%82%AF%E3%82%A8%E3%83%AA%E3%83%97%E3%83%AD%E3%82%BB%E3%83%83%E3%82%B7%E3%83%B3%E3%82%B0%E6%9C%80%E9%81%A9%E5%8C%96-1-%E5%B0%8E%E5%85%A5%E7%B7%A8-5a6387076e8e)

