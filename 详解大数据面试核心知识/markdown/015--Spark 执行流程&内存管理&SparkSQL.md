Spark 的相关概念我们整理清楚了，接下来就是让 Spark
“跑起来”，让我们看看它是怎么运行的，包括它的任务提交和执行流程，执行过程中的内存管理机制，以及 SparkSQL 的执行和优化。掌握执行流程相关的内容对
Spark 任务的 debug
和优化有很大帮助，可以帮助我们迅速定位到哪个执行的环节出了问题，或者哪里出现了瓶颈，解决此类问题通常是大数据工程师的日常工作，所以这不仅仅是对知识点的考察，也是对工作能力的考察。

**本篇面试内容划重点：内存动态占用，Catalyst，优化方法**

### 任务执行流程

![image.png](https://images.gitbook.cn/2020-06-11-070046.png)

Spark 任务的提交和执行离不开和资源管理器的交互，此处抽象了资源管理器，没有区分 on Standalone 还是 on
Yarn，因为调度架构是类似的。（Yarn 相关任务处理流程后续会有专门章节说明）。

  1. Client 客户端提交运行 Spark Application 应用，应用中创建 SparkContext，由它负责与 ClusterManager（资源管理中心） 通信，进行 Executor 资源的申请。
  2. ClusterManager 在健康的 Worker 上启动 Executor，Executor 运行情况将随着心跳发送给 ClusterManager。
  3. Executor 的 ExecutorBackend 进程会向 SparkContext 注册。
  4. SparkContext 得知有哪些 Executor 后，将 Application 代码发送给 ExecutorBackend。
  5. SparkContext 开始构建成 DAG 图，然后 DAGScheduler 将 DAG 图根据 RDD 的依赖关系划分为多个 Stage，并提交给 TaskScheduler 对象。
  6. **TaskScheduler** 负责分配 Task 给相应的 executor，并将封装好的 TaskSet 提交给 Worker 上的 executor 运行。
  7. Task 在 Executor 上运行，运行完成后释放所有资源。

#### YARN-Client 与 YARN-Cluster 模式的对比

Spark On Yarn 是用得最多的部署方式，所以这里再简单提一下。Spark On Yarn 的运行模式下有一个
**ApplicationMaster** 进程，，（在上面的任务执行流程图中没有提到），它负责和 Yarn 的资源管理器
ResourceManager（即图中的 ClusterManager）通信请求资源，获取资源之后通知 Yarn 的执行节点
NodeManager（即图中的 Worker） 为其启动 Container。Client 和 Cluster 不同得关键就在于这个
ApplicationMaster。

  * YARN-Cluster 模式下， **Driver 运行在 ApplicationMaster 中** ，它负责向 YARN 申请资源，并监督作业的运行状况。当 Client 端提交了作业之后，由于 Driver 不在 Client 端，Client 的工作就已经完成了，此时完全可以 Kill 掉 Client，作业会继续在 YARN 上运行，因而 YARN-Cluster 模式不适合运行交互类型的作业， **作业一旦提交就和 Client 端没有关系了。**
  * YARN-Client 模式下， **Driver 运行在 Client 端** ，所以 Client 端此时要负责向 YARN 申请资源，并监督作业的运行状况，所以在整个作业运行过程中，它是不能被 Kill 掉的。Application Master 此时仅仅是做 **向 YARN 申请 Container 资源** 的工作。

### 内存管理机制

这里只讲 Spark.2+ 版本默认的统一内存管理方式下的内存分配情况。Spark 的内存分为堆内和堆外内存。 **__堆内内存的划分和管理？__**
**__![image.png](https://images.gitbook.cn/2020-06-11-070048.png)__****堆内内存分为以下四个部分**
**Execution 执行内存** ：主要用于存放 Shuffle、Join、Sort、Aggregation 等 **计算过程中** 的临时数据；
**Storage 存储内存** ：主要用于存储 Spark 的** Cache 数据**，全局变量、静态变量，例如 RDD 的缓存、unroll 数据；

**User Memory 用户内存** ：用于存储 **用户定义的数据结构** 和 **Spark 内部元数据** 比如 RDD 依赖等信息。

**Reserved Memory 预留内存** ：系统预留内存，防止 OOM，默认占 300M。

涉及到的相关配置参数 spark.executor.memory： **System Memory 系统总内存**
spark.memory.fraction：执行内存和存储内存共占 usableMemory 可用内存的比例。
spark.memory.storageFraction：存储内存占（执行内存+存储内存）总和的比例。

**内存的动态占用机制？** 统一内存管理的特点是存储内存和执行内存可以互相占用，但是要遵循以下规则

  * 双方的空间都不足时，则存储到硬盘，将内存中的 Block 存储到磁盘的策略是按照 LRU 规则进行的。
  * 执行内存被占用，可将被存储内存占用的部分转存到硬盘。
  * 存储内存被占用，只能等待释。(因为需要考虑 Shuffle 过程中的很多因素)

**堆外内存的划分和管理？** 默认堆外内存是关闭的，如果启用，那么 Executor
内将同时存在堆内和堆外内存，两者的使用互不影响，不存在堆内内存不够去借用堆外内存的情况。此时 Execution 内存是堆内 Execution 和堆外
Execution 内存之和，Storage 同理。另外，堆外内存只区分 Execution 内存和 Storage 内存，不存在 Reserved。

### 开发过程中的优化

内容总结自美团的《[Spark 性能优化指南](https://tech.meituan.com/2016/04/29/spark-tuning-
basic.html)》，文章写得非常全面，有兴趣可以去细看。 **代码优化注意事项**

  * 对于需要多次复用同一个 RDD 的场景，可以合理地使用持久化来避免多次计算，关于持久化方式 Cache/Persist/CheckPoint 的区别在前面的章节已经说过，不再赘述。
  * 尽量避免使用 shuffle 类算子，前面已经举了 repartition 和 coalesce 的例子，不过具体的使用选择还得看实际的计算场景，另外对于 rdd1 与 rdd2 的 join 场景，如果 rdd2 的数据量不大，可以将 rdd2 数据广播，然后在 rdd1 中遍历 rdd2 数据，根据关联字段进行数据拼接。
  * 使用 reduceByKey/aggregateByKey 替代 groupByKey，具体原因前面的内容也说过了，主要是 map 端的预聚合。
  * 数据库初始化或者其他初始化相关的工作在分区层面去完成，即使用 mapPartitions/foreachPartitions 而不是在 map/foreach 内部初始化。
  * 使用 Filter 之后，如果出现某些分区数据很少甚至没有数据而造成资源浪费，可以适当使用 coalesce 操作减少分区数。
  * 使用 repartitionAndSortWithinPartitions 替代 repartition 与 sort 类操作，用于需要重分区且排序的场景。

**数据倾斜问题解决方案？**

  * 使用 Hive ETL 预处理数据 (预处理过程还是会有数据倾斜问题)
  * 过滤少数导致倾斜的 key (只用于只有少数 key 数量大的情况)
  * 提高 shuffle 操作的并行度 (spark.sql.shuffle.partitions 缓解，其效果有限)
  * 两阶段聚合(局部聚合+全局聚合) (先打前缀然后再去掉，只用于聚合类操作，join 类不适用)
  * 将 reduce join 转为 map join (只用于大表和小表的情况，小表广播 Broadcast)
  * 采样倾斜 key 并分拆 join 操作 (用于少数 key 数据量大，加前缀，扩容)
  * 使用随机前缀和扩容 RDD 进行 join (大量 key，内存消耗大)
  * 多种方案组合使用

### SparkSQL

#### DataFrame

DataFrame 是基于 SparkSQL 构建的数据结构，DataFrame 和 RDD 类似，也是不可变的分布式弹性数据集，在此基础上
DataFrame 增加了数据的 schema 信息，也就是说 DataFrame 的每一列是有名称和类型的，类似于关系型数据库的表结构。SparkSQL
的执行本质上就是把 SQL 语句转化 Compute Func 算子并在 RDD 上进行相关的操作。

#### Catalyst 工作流程

Catalyst 优化器是 SparkSQL 的核心，我们直接来看看它的工作流程。
![image.png](https://images.gitbook.cn/2020-06-11-070050.png)

  1. **Parser**

即 SQL 解析，使用 antlr 生成解析器，解析过程是词法和语法解析。

  * 词法解析重点是分词，一个字符一个字符识别，关键字、标识符、字面量、运算符、分界符。
  * 语法解析会 **生成抽象语法树(AST)** ，并提供遍历树的接口。

  1. **Analyzer**

语义分析，绑定元数据(metaData)。上一个阶段构建了一颗语法树，但是系统并不知道表名，表字段，sum 函数等是什么。所以 Analyzer
阶段，就是将基本的元数据与语法树进行绑定。最重要的元数据信息主要包括两部分：

  * 表的 scheme 主要包括表的基本定义（列名、数据类型）、表的数据格式（Json、Text）、表的物理位置等，
  * 基本函数信息主要指类信息。

  1. **Optimizer**

生成逻辑执行计划和逻辑执行计划优化的过程，优化器是整个 Catalyst
的核心（基于规则），优化策略实际上就是对语法树进行一次遍历，模式匹配能够满足特定规则的节点，再进行相应的等价转换。比较常见的规则有：谓词下推（Predicate
Pushdown）、常量累加（Constant Folding）和列值裁剪（Column Pruning）。

  4. **Physical Plan**

逻辑执行计划是没办法真正执行的，只是逻辑上可行，Spark 也不知道如何去执行。比如 Join 只是一个抽象概念，代表两个表根据相同的 id
进行合并，然而具体怎么实现这个合并，逻辑执行计划并没有说明。逻辑执行计划转换为物理执行计划，将逻辑上可行的执行计划变为 Spark 可以真正执行的计划。

比如 Join 算子，Spark 根据不同场景为该算子制定了不同的算法策略，有 BroadcastHashJoin、ShuffleHashJoin 以及
SortMergeJoin 等（可以将 Join 理解为一个接口，BroadcastHashJoin
是其中一个具体实现），物理执行计划实际上就是在这些具体实现中挑选一个耗时最小的算法实现，这个过程涉及到基于代价优化策略

**基于代价与基于规则的优化**

  * **RBO（基于规则）**

上面已经提到了，RBO 的优化方式是基于一套已经定制好的规则来优化的，如果客户端提交的 SQL 满足优化的规则，那么就会对 SQL 做 RBO 优化。

  * 谓词下推：指在底层数据源进行扫描时就完成数据的过滤，而不是将所有数据读到 Spark 后再进行过滤的优化方法。
  * 列值裁剪：指裁剪去在 SQL 执行过程中无用的字段，减少操作的数据，避免资源浪费。
  * 常量累加：减少常量操作。
  * **CBO（基于代价）**

物理执行计划是一个树状结构，每一个节点都是都是有执行代价的，主要分为两部分

  * 该执行节点对数据集的影响，或者说该节点输出数据集的大小与分布（需要去采集）
  * 该执行节点操作算子的代价（相对固定，可用规则来描述）

在 SQL 执行之前会根据代价估算确定一种代价最小的方案来执行。

#### SparkSQL 其他的优化策略？

除了上述的 RBO 和 CBO 的 SparkSQL 执行优化策略，还有 Spark 还有如下的优化方法。

  * **内存缓存**

SparkSQL 实现了 cacheTable 用于将数据加载到内存进行缓存。cacheTable
相当于将数据表在分布式集群内存里的创建一个视图，将数据进行缓存，这样迭代和交互的查询就不需要再从磁盘读取数据，减少了 I/O 开销。

  * **列式存储**

cacheTable 还会将 **数据存储转换为列式存储，** 列式存储的优势在于 Spark SQL
只要读取用户指定的列，而不需要每次都读取所有的列，大大地减少了内存中需要缓存的数据量，减轻网络传输和磁盘 I/O 的压力。

  * **压缩**

列式存储由于会把数据类型相同的数据存储在一起，所以能够利用 **序列化和压缩** 减少内存的空间占用。Spark SQL
采用了一些压缩策略对内存列存储数据进行压缩，它支持 PassThrough，RunLengthEncoding，IntDelta 等多种压缩方式，对提高
SparkSQL 的执行效率有很大帮助。

