看到这里，恭喜你，结束了分布式存储模块的内容，愿此刻你已经把前面的章节都看透了，但是即便如此也还是不要松懈，接下来还有一个大模块的内容。因为大数据的技术基本都是计算与存储分离，各司其职，所以我们需要继续来看分布式计算模块的内容，让我们大数据面试的知识点更加完善。

第一篇写的是分布式计算界的中流砥柱，Spark。Spark 在实现上和 MapReduce
计算框架类似，但是它在内存的使用上更“贪婪”，也减少了数据磁盘持久化的频率，这使它成为了一个高效的大数据处理引擎。Spark
能解决大数据领域很多的问题，万金油一样的存在，离线处理、实时流处理、机器学习、交互式查询等等，所以 Spark
相关的内容在大数据面试中也占据了很大的比例，本专栏也会用比较多的篇幅来详细梳理 Spark 的内容，覆盖到尽可能多面试题。

**本篇面试内容划重点：shuffle，checkpoint，RDD**

### Spark 的重要概念

为了便于理解，讲概念之前我们结合下图一个最简单的 wordcount 的实例来说说 Spark 的数据处理流程。首先 Spark 从 HDFS 读
log.txt 文件，log.txt 存储在 HDFS 中被分成了三个块，Spark 会起了三个 Task 去读每个 Block 的数据，读到数据后
Spark 会按照算子的逻辑在 Task 内对每一条数据做相关操作（如图的 flatMap 和 map），如果遇到 shuffle 类算子（如图的
reduceByKey），会把数据打散，然后相同 key 的数据汇聚到同一个节点做聚合，另外下游 Stage 的计算会在上游 Stage 所有 Task
都完成之后。

![image.png](https://images.gitbook.cn/2020-06-11-070006.png)

  * **RDD** 即弹性分布式数据集，Spark 中最基本的数据抽象，如图 Spark 从 HDFS 读取数据会将数据转化为 RDD，在 Spark 通过算子（fllatMap/map 等）计算后会把一个 RDD 转化为另一个 RDD 而不是直接在原有 RDD 上更新。
  * **Partition** 即分区，它是 Spark 中最小的执行单元。RDD 数据分布在各个节点，每一个节点的数据集即可理解为一个分区。如图 Partition1-Partition3。
  * **Compute Func （RDD 算子）** ，Spark 中的 RDD 计算是以分区为单位的，每个 RDD 都会实现 compute 函数来对数据做相关计算。算子可以分为以下几类。
  * **Transformation 转换算子** ：延迟计算，不会触发作业的提交。
    * Value：处理的数据类型为 value 类型。(map/union/mapPartitions/filter/distinct...)
    * Key-Value：处理的数据类型为 KV 类型。(reduceByKey/groupByKey/join/cogroup...)
  * **Action 执行算子** ：触发 SparkContext 提交 Job 作业。（foreach/saveAsTextFile/collect/count...）
  * **Lineage** ：血缘依赖分两种 Narrow Dependencies （窄依赖）与 Wide Dependencies （宽依赖），用来保证数据容错时的高效性。
  * Narrow Dependencies 指父 RDD 与子 RDD 的分区之间是一对一或者多对一的关系，不会产生 Shuffle。如图 RDD1 和 RDD2。
  * Wide Dependencies 指父 RDD 与子 RDD 的分区之间是一对多的关系，一个父 RDD 的分区数据会打散分布到下游各个分区中，过程会产生 Shuffle。如图 RDD2 和 RDD3。
  * **Task：** 在 Executor 进程中实际执行任务的工作单元，一个 Task 包含一个分区数据在一个 Stage 中的整个计算流程。如图 Task1/Task2/Task3。同一个 Stage 之中的 Task **计算逻辑相同，数据不同。**
  * **Stage** ：如图 Stage1 和 Stage2，宽依赖是两个 Stage 的分界点，Stage 在 Driver 端划分，然后按照顺序在 Executor 端执行，下一个 stage 需要等上一个执行完并从上一个 stage 获取数据。
  * **DAG：** 有向无环图，是一组顶点和边的组合。顶点代表了计算过程中的各个 RDD， 边代表了上下游 RDD 的依赖关系。如上图，整体就是一个 DAG。

**作为 Spark 的核心，RDD 有哪些特性？** **Partition**
，数据集的基本组成单位，分布在各个服务器上，每个分区都会被一个计算任务处理，决定并行计算的粒度。 **Partitioner** ，对于 kv 型的
RDD（图中 RDD2），spark 实现了分区函数。它可以决定 shuffle 过程的分区方式和输出的分区数量。 **Compute Func**
，Spark 中的 RDD 的计算是以分区为单位的，每个 RDD 都会实现 compute 函数来对数据做相关计算，比如图中的
flatmap/map/reduceByKey 都是 RDD 的实现的算子。 **Dependency** ，RDD 是不可变的，只能通过算子生成新的
RDD，同时 spark 会保存 RDD 之间的依赖关系。在部分分区数据丢失时，可以通过依赖关系重新计算丢失的分区数据，而不是对 RDD
的所有分区进行重新计算。 **PreferredLocation** ，Spark
秉承“移动数据不如移动计算”的理念，在进行任务调度的时候，会尽可能地将计算任务分配到其所要处理数据块的存储位置。

**RDD 的容错机制机制？** 没错，又是无处不在的容错问题，RDD 的容错从 Lineage 和 CheckPoint 两个角度来看。

  * 依赖关系 Dependencies 构成了 RDD 的血缘 Lineage ，父 RDD 通过算子计算得到子 RDD，Dependencies 会存储上下游的 RDD 相关元数据信息，在存在有 RDD 异常时，Spark 可以根据血缘来重构异常的 RDD。
  * CheckPoint 可以解决 DAG 中 Lineage 过长或者宽依赖父 RDD 多导致重构 RDD 开销大的问题，它主要是可以把 RDD 写入磁盘，相当于将依赖关系截断后落盘，如果后面的 RDD 出现故障只需要从 CheckPoint 落盘的位置开始重构 RDD 即可，减小了资源消耗。

### Shuffle

Shuffle 这块内容在面试中很重要，是考核对 Spark
原理理解的一个重要知识点，所以我们用放大镜把上图中的“shuffle”这块内容放大，得到了下面这张图。
![image.png](https://images.gitbook.cn/2020-06-11-070008.png) Shuffle
是为了满足让不同分区的数据能够聚合在一起计算的需求，而对数据打散重新分区组合的操作。Shuffle 过程分为 shuffle write 和 shuffle
read 两个阶段，（这里只讲 Sorted-Based Shuffle）。

  * **Write 阶段** 会把 Mapper 中每个 MapTask 所有的输出数据排好序，然后写到一个 Data 文件中，同时还会写一份 index 文件用于记录 **Data 文件内部分区数据的元数据** （即记录哪一段数据是输出给哪个 ReduceTask 的），所以 Mapper 中的每一个 MapTask 会产生两个文件 。
  * **Read 阶段** 首先 Reducer 会找 Driver 去获取父 Stage 中每个 MapTask 输出的位置信息，跟据位置信息获取并解析 Index 文件，会 **根据 Index 文件记录的信息来获取所需要读取的 Data 文件中属于自己那一部分的数据** 。

Mapper 端的排序分为两个部分：内部排序和分区排序，每个分区内部的数据是有序的，分区之间也是有序的，如图 Data 文件中有三个分区，分别着对应下游的
ReduceTask，分区排序的目的是让 ReduceTask 获取数据更加高效。

**Sorted-Based Shuffle 过程 Mapper 端生成的文件个数与什么有关？** 一个 core 即一个 Task
即一个分区即一个并发，一个 Task 会生成两个文件，Data 文件和 Index 文件。与 Reduce 端无关，Reduce
只负责读属于自己的那一部分数据。 **Sorted-Based Shuffle 解决的问题和存在的缺陷？** 相比老版本的 HashShuffle，减少了
Mapper 端 ShuffleWriter 所产生的文件数量，降低了资源消耗。

存在的缺陷是它会强制要求数据在 Mapper 端必须进行排序，无论最终结果是否需要排序，排序会消耗额外的资源导致速度有所减慢，但是 Spark
在排序方面也做了一些优化来弥补这个缺陷，比如 Tungsten 来优化排序算法。

### RDD Compute Func

RDD 的算子非常多，每一个都拿来详细讲不太现实，所以这里选择了几组比较有代表性的，希望大家能够基于这几组算子发散开来，更好地掌握算子底层的实现。

#### coalesce 和 repartition 的区别

解决问题最好的办法就是直接看看 coalesce 和 repartition 的源码 `def ``repartition``(numPartitions:
``Int``)(``implicit ``ord: ``Ordering``[``T``] = ``null``): RDD[``T``] =
withScope {` `coalesce(numPartitions``, ``shuffle = ``true``)` `}`

`def ``coalesce``(numPartitions: ``Int, ``shuffle: ``Boolean ``= ``false,`
`partitionCoalescer: Option[PartitionCoalescer] = Option.``_empty_``)`
`(``implicit ``ord: ``Ordering``[``T``] = ``null``)` `: RDD[``T``] = withScope
{` `......` `}`

由上述源码的传参可知，repartition 与 coalesce 的 shuffle 参数设置为 true
时是完全一样的。所以关于这两个算子选择的问题就转变成了需不需要 Shuffle 的问题了。repartition 算子一定会产生 Shuffle；而
coalesce 算子默认不产生 Shuffle（可配置）。

下面看个实例，例子中 Spark 任务的配置为 4 个并行度即：--executor-cores 2 --num-executors 2

  1. **上游 Partition 数量 > 下游 Partition 数量**

  * **repartition（shuffle）**
    * 每个 Task 负责读取上游的一个分区，一个分区读取结束后才读取下一个分区。如图中实线箭头所示，任务开始时每个 Task 对应一个分区来拉取数据；当某个 Task 完成了一个分区的数据读取后，才会开始读另一个分区的数据，即如图中虚线箭头所示。
    * Task 默认 Hash 分区，将数据求 Hash 值后写入下游的两个 Partition 中。

![image.png](https://images.gitbook.cn/2020-06-11-070010.png)

  * **coalesce**
    * coalesce 默认没有 Shuffle，RDD 之间是窄依赖关系，所以 Task 和下游 Partition 一一对应，每个 Task 只写一个 Partition，当并行度小于下游分区数的场景下（如图），则存在 Executor 的 Task 空跑的情况。造成了资源浪费。
    * Task 读数据同样是一个分区一个分区读，但是由于下游分区数量只有两个，整个计算的过程为窄依赖关系，所以一次只会读两个上游的分区写到下游。

![image.png](https://images.gitbook.cn/2020-06-11-070011.png)
上面的两种方式实际上都各有各的缺点，repartition 会带来 Shuffle，那么就意味着需要进行额外的网络 IO；coalesce 虽然没有
Shuffle，但是会存在 Executor 空跑，资源浪费的情况。那么如何做选择，这其实是在两者找平衡点的一个过程，可参考下述方法。

  * Spark 的并行度（即图中 Task 的数量）大于下游 Partition 的数量则可以用 coalesce，不会造成 Executor 的空跑。
  * 如果上下游的分区数量差距非常大，比如上游 1000 个分区，下游 1 个分区，此时如果用 coalesce 那么只会有一个 Task 在处理数据，势必会让任务处理变得非常慢。用 repartition 则可以动用所有 Task 来处理数据，最终通过 Shuffle 将数据写入一个分区中，虽然有 shuffle 但是和一个 Task 处理任务比起来效率还是更高的。

  1. **上游 Partition 数量 < 下游 Partition 数量**

不经过 shuffle，是无法增加 RDD 的分区数的，因为窄依赖关系的分区数据计算流程一定是在同一台服务器完成的，一个分区通过 RDD 的
Transformation
算子计算后生成对应的另一个分区，如果多了一个分区那么数据就需要通过网络传输才能到达这个多出来的分区，违背了窄依赖的计算原则。所以上游 Partition
数量 < 下游 Partition 数量的场景，只能通过 shuffle 计算实现（repartition）。
**![image.png](https://images.gitbook.cn/2020-06-11-070013.png)**

#### groupByKey 和 reduceByKey 的区别

按照和上面一样的套路，我们来看看这两个算子的源码。 def reduceByKey(func: (V, V) => V, numPartitions:
Int): RDD[(K, V)] = self.withScope { reduceByKey(new
HashPartitioner(numPartitions), func) }

def groupByKey(partitioner: Partitioner): RDD[(K, Iterable[V])] =
self.withScope { ...... }

如源码所示，reduceByKey 可以接收一个 func 函数作为参数，这个函数会作用到每个分区的数据上，即分区内部的数据先进行一轮计算，然后才进行
shuffle 将数据写入下游分区，再将这个函数作用到下游的分区上，这样做的目的是减少 shuffle 的数据量，减轻负担。
![image.png](https://images.gitbook.cn/2020-06-11-070014.png) groupByKey
不接收函数，Shuffle 过程所有的数据都会参加，从上游拉去全量数据根据 Key
进行分组写入下游分区，这样会消耗比较多的资源，数据传输会导致任务处理的延迟。
![image.png](https://images.gitbook.cn/2020-06-11-070015.png)

**Cache、** **Persist、CheckPoint 的区别？** **先来说说 Cache 和 Persist ，**
首先如果用了上面的方式看了源码就能够发现，Persist 的 MEMORY_ONLY 级别的存储等于 Cache，Persist
其他的配置只是存储的方式不同，作用和原理是和 Cache 类似的，所以这两个的特性放在一块说。CheckPoint 和这两个差别就比较大了。

  * Cache、Persist 是转化类算子，和其他算子一样，触发的时机是在对应分区的上游算子计算完成之后。
  * Cache、Persist 会把 RDD 缓存到指定位置，这个操作不会改变 Lineage 血缘的依赖关系，且 Job 执行完成之后，缓存的数据会被清除。
  * Cache、Persist 一般应用于需要访问重复数据的应用（如迭代型算法和交互式应用）缓存可以运行得更快。
  * CheckPoint 是在 Job 结束后再另外启动专门的 Job 去完成的。所以，CheckPoint 时 RDD 会被计算两次。
  * CheckPoint 执行完毕后，会产生 CheckPointRDD，此时 lineage 血缘关系已经改变了，容错会从 CheckPointRDD 开始。
  * CheckPoint 将 RDD 持久化到 HDFS ，会被永久保存，可以给其他的 Driver 使用

