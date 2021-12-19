文件系统 HDFS，数据库 HBase，消息队列 Kafka，协调服务
Zookeeper，接下来要让大数据存储的生态圈变得更完整，不得不提的就是搜索引擎了，虽然说 ElasticSearch
是以搜索闻名的，但是说到底它还是个数据库，它底层的存储特性使它拥有了毫秒级的全文检索响应能力，很好地补充了大数据在文本检索这块的处理能力。 ES 的创始人
Shay Banon 很好地诠释了伟大的项目是来自于生活的，据说当初 ES
开发的最初目的是为了给他老婆做一个食谱搜索的功能，接下来我们就来看看这个“食谱搜索引擎”在面试官那里能问出些什么花样。

**本篇面试内容划重点：架构、读写、调优。**

### 架构与角色职责

![image.png](https://images.gitbook.cn/2020-06-11-070016.png) ElasticSearch
采用了 Raft 协议来解决共识（Consensus）问题，所以它的各个进程之间的交互逻辑包括选举等和使用 Zab 协议的 Zookeeper
是有点类似的，（毕竟这两种协议都来自于同一个“祖师爷” Paxos）。

  * **Master 节点：** Master 有且仅有一个，如果宕机会通过选举机制再选择出一个（后面有详解），它负责维护集群的状态，管理集群范围内的所有变更，例如增加、删除索引，或者增加、删除节点等。主节点并 **不需要涉及到文档级别的变更和搜索等操作** 。配置：node.master: true ，注意，这个配置是将节点配置为 Master Eligible Node，说明它有参加选举 Master 的权利。
  * **Data Node 节点：** 和 HDFS 的 DataNode 节点类似，数据节点，负责实际数据和其对应的倒排索引的存储，数据会分片存储在不同的节点中。数据的读写请求最终都会找到相应的 Data Node 节点负责。默认每一个节点都是数据节点（包括主节点），可以通过 node.data 属性进行设置。配置：node.data: true 。
  * **Coordinating Node 节点：** 协调节点主要负责接收客户端的请求，并将请求转发到请求的数据所在的数据节点，最后聚合各个数据节点返回的数据，一并返回给客户端。每个节点默认都是 Coordinating Node。
  * **Ingest Node: **这种节点支持对索引的文档做预处理的功能，预处理的操作可以由用户来自定义。这种节点不太常用理解，知道即可，因为预处理的操作一般会放在 logstash 中。所以这个节点不是集群必须的，有需要可以通过配置 node.ingest: true 来开启功能。

### 读写场景

![image.png](https://images.gitbook.cn/2020-06-11-070018.png)

#### 写数据

  1. 客户端选择任意一个 Coordinating Node 节点发送 Document 写入请求。
  2. ES 自动为每个 Document 分配一个 **全局唯一的 doc id** （也可以手动指定，但是要求是要全局唯一），同时也是根据 doc id 进行 hash 路由到对应的 primary shard 上。 
  3. Primary Shard 处理客户端的写请求, 然后将写完成后的 Document 同步到其对应的 Replica Shard 中， **增删改操作由 Primary Shard 处理, Replica Shard 只处理查询请求** 。
  4. Coordinating Node 如果监控到 Primary Shard 和所有 Replica Shard 都已经处理完这个 Document，就返回响应结果给客户端。

详细说下 **Data Node 处理数据的流程**

  1. 首先，Primary Shard 接收到客户端的写请求，会同时写 Buffer 和 Translog 两个地方。数据写入 Buffer ，写操作日志写入 Translog。（在 Buffer 里的数据客户端是搜索不到的）
  2. 如果 Buffer 快满了，或者到达默认阈值 1s，ES 就会将 Buffer 数据 refresh 到一个新的 segment file 中，此时的 Segment File 在 os cache（操作系统级别的一个内存缓存），不在磁盘中。 **这个过程就叫做 refresh** 。（refresh 过后， os cache 里的数据客户端可以搜索到）
  3. os cache 中的数据达到默认时间阈值 30min 或者 Translog 日志达到阈值会触发 flush 操作，将 Segment File 数据写入到磁盘中， **这个过程就叫做 flush** ，写入磁盘后 Translog 会被删除。

#### 写数据过程的优化方法

又是优化相关的内容，越是需要保持高性能的组件，对优化的要求越高，需要费劲心思地压榨它的性能。

  * **开启最佳压缩：** 对于打开了_source 字段的 index，可以使用压缩算法 DEFLATE，提高数据压缩率。
  * **bulk 批量写入：** 写入数据时尽量使用下面的 bulk 接口批量写入，提高写入效率。每个 bulk 请求的 doc 数量设定区间推荐为 1k~1w，具体可根据业务场景选取一个适当的数量。
  * **调整 translog 同步策略：** 默认情况下，translog 的持久化策略是，对于每个写入请求都做一次 flush，刷新 translog 数据到磁盘上。这种频繁的磁盘 IO 操作是严重影响写入性能的，若数据一致性要求不是那么高，可适当调整 translog 的刷盘周期。
  * **merge 并发控制：** 一个 index 由多个 shard 组成，一个 shard 由多个 segment 组成，segment 到达阈值会合并成一个大的 segment，这个过程被称为 merge。当节点配置的 cpu 核数较高时，merge 占用的资源可能会偏高，影响集群的性能，可以适当降低 merge 过程的并发度。
  * **调整 refresh 时间间隔：** 我们已知默认情况下，ES 每一秒会 refresh 一次，产生一个新的 segment，这样会导致产生大量 segment，从而 segment merge 较为频繁，系统资源消耗大。如果对数据的实时可见性要求较低，可以加大 index.refresh_interval 配置，降低系统开销。
  * **默认生成 _id：_** 当 Clinet 指定id 写入数据时，ES 会先判断这个 id 是否存在，不存在则写入，存在则先删后写。这样会消耗资源做判断，所以，在不需要通过 _id 做相关操作的场景中，可以配置默认生成_ id，能提高将近一倍效率。

#### 读（搜索）

![image.png](https://images.gitbook.cn/2020-06-11-070019.png)

  1. 客户端选择 **任意** 一个 Coordinating 节点发送请求。
  2. Coordinating Node 对 Document 进行路由，将请求 **转发** 到对应的 Data Node，此时会使用 **round-robin 随机轮询算法** ，在 primary shard 以及其所有 replica 中选择一个，让读请求 **负载均衡** 。
  3. 接收请求的 Data Node 返回 document 给 coordinate node
  4. coordinate node 返回 document 给客户端

**扩展一下搜索的步骤：**

  1. 读取 Document 时，如果是通过 Doc Id 来查询，Coordinating Node 会根据 Doc Id 进行 hash，判断出 Doc Id 被分配到了哪个 shard 上面去，从那个 shard 去查询。
  2. 如果需要对某字段进行 **全文检索** ，那么协调节点会将搜索请求转发到所有的 shard 对应的 primary shard 或 replica shard。
  3. **query phase：** 每个 shard 将自己搜索结果的 Doc Id 返回给协调节点，由协调节点进行 **数据的合并、排序、分页** 等操作。
  4. **fetch phase** ：协调节点根据 doc id 去各个节点上拉取实际的 document 数据，最终返回给客户端。

另外大家关心的倒排索引和一些底层优化的原理会在下一节详细说明。

**为什么 ES 说是准实时的搜索引擎？** 因为 refresh 过程默认是每隔 1 秒刷一次的，refresh
之后数据才能被真正地检索到，所以中间会有一个短暂的延迟即 refresh 过程。但是，es 也允许用户通过 api 执行 refresh
操作，让数据立马就可以被搜索到。** **Translog 能否 100%保证数据不丢失？** 默认情况下不能，因为 Translog
也是先写到内存中，然后再 fsync 到磁盘，默认 5 秒，即集群宕机最多可能丢 5s 的数据。但是你也可以选择在每次请求后都做一次 fsync
操作，落盘后再响应客户端，这样能保证数据不丢失，但是会带来一些性能的损失。这个取舍 es 交给用户来做决定。

#### 读数据过程的相关优化方法

  * **控制字段的存储选项**

在创建 Document 的时候，有以下的配置项，如果没有对应的需求可关闭功能，提升效率。

  * doc_values 用于控制列式存储，适用于聚合、排序、脚本等操作；
  * index 用于控制倒排索引；
  * field_names 用于 exists 查询，来确认某个 doc 里面有无一个字段存在。
  * **使用 routing**

对于数据量较大的 index，一般会配置多个 shard 来分摊压力。这种场景下，一个查询会同时 **搜索所有的 shard** ，然后再将各个 shard
的 **结果合并后**
，返回给用户。对于高并发的小查询场景，每个分片通常仅抓取极少量数据，此时查询过程中的调度开销远大于实际读取数据的开销，且查询速度取决于最慢的一个分片。 开启
routing 功能后，ES 会将 routing 相同的数据写入到同一个分片中（也可以是多个，由 index.routingpartitionsize
参数控制）。 **如果查询时指定 routing，那么 ES 只会查询 routing 指向的那个分片，可显著降低调度开销，提升查询效率** 。

  * **查询时，使用 query-bool-filter 组合取代普通 query**

默认情况下，ES 通过一定的算法计算返回的每条数据与查询语句的相关度。但对于非全文索引的使用场景，用户并不在意相关度，只是想 **精确的查找目标数据**
。query-bool-filter 可以不计算相关度，并缓存结果集，为后续相同查询提高效率。

  * **按需控制 index 的分片数和副本数**
  * 对于数据量较小（100GB 以下）的 index，查询会比写入频繁的场景，一般设置 3~5 个 shard，副本数设为 1 。
  * 对于数据量较大（100GB 以上）的 index：
    * 一般把 **单个 shard 的数据量** 控制在（20GB~50GB）
    * **让 index 压力分摊至多个节点** ：可通过 index.routing.allocation.totalshardsper_node 参数，强制限定一个节点上该 index 的 shard 数量，让 shard 尽量分配到不同节点上
    * 综合考虑整个 index 的 shard 数量，如果 shard 数量（不包括副本）超过 50 个，就很可能引发拒绝率上升的问题，此时可考虑把该 **index 拆分为多个独立的 index** ，分摊数据量，同时配合 routing 使用，降低每个查询需要访问的 shard 数量。

### 系统层面的调优

系统层面的优化主要是 Linux 为了配合 ES 使用可以做的一些配置调整，需要压榨性能就必须考虑到方方面面的内容。

  * **JVM** ：官方建议，-Xms 和-Xmx 设置为相同的值，避免在运行过程中再进行内存分配，同时，如果系统内存小于 64G，建议设置略小于机器内存的一半，剩余留给系统使用。 jvm heap 建议不要超过 32G，否则 jvm 会因为内存指针压缩导致内存浪费。
  * **交换分区** ：关闭交换分区，防止内存发生交换导致性能下降。
  * **文件句柄** ：Lucene 使用了大量的文件，同时 Elasticsearch 在节点和 HTTP 客户端之间进行通信也使用了大量的套接字，所有 ES 需要有足够的文件描述符，加大文件句柄至 65536。
  * **mmap：** Elasticsearch 对各种文件混合使用了 NioFs（ 注：非阻塞文件系统）和 MMapFs （ 注：内存映射文件系统）。确保配置的最大映射数量，以便有足够的虚拟内存可用于 mmapped 文件。
  * **磁盘**
  * **使用固态硬盘提高读写速度和稳定性** 。
  * **使用 RAID 0** 条带化存储，可以提升磁盘读写效率。
  * 挂载多块硬盘可提升读写操作效率。
  * 避免使用 NFS，减少网络的延迟对性能的影响。

