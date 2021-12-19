### 运算原理

首先以词频统计的案例，来描述一下 MapReduce
的运算原理与一些基本的概念。这里输入的数据是一些英文的文章，它有很多行组成，而每一行又包含很多单词，每个单词之间由空格隔开；现在需要使用 MapReduce
来统计每个单词的出现次数。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201018062221.png)

这里输入的案例数据比较少，只有三行，分别是 Deer Bear River、Car Car River、Deer Car Bear。

当数据被上传到 HDFS 的时候，会被自动拆分（以 128M 为标准）为 Block 存储，MapReduce 在执行前，需要一个 **Splitting
阶段** 来确定 Map 数量，默认情况下与 Block 数量保持一致，即 Splitting 阶段不做任何处理，直接沿用 Block
数量，然后直接在下一个阶段将计算任务移动到每个 Block 上即可。但 Splitting 真正存在的意义在于——自定义 Map
数量，如果需要更多的并发度，则还需要对存储在 HDFS 上的 Block 进行拆分，如果更少的并发，则对 Block 进行合并。

这里的 Splitting 使用默认情况，假设文件在被存储到 HDFS 时，被拆分了 3 个 Block，每个 Block 分别存储了一行数据；那这里
Splitting 不做任何处理，即 3 个 Split。

之后每一个 Split 数据块上便会启动一个 Map 任务，进入到 **Map 阶段** ，进行部分结果的运算。词频统计这里，每个 Map
任务会将每一行数据按照空格进行拆分，拆分成单词后，将每个单词标记数量为 1，即（Word，1）。这里要注意的是，MapReduce 程序中，Map
的输出结果都是以元组的形式，使用（Key，Value）对来保存。

Map 计算完成之后，这些部分结果就要进行汇总，需要将单词相同的数据，分发到同一个节点进行累加。这个阶段就是 **Shuffle（洗牌）阶段**
。Shuffle 阶段会将 Map 阶段的结果（Key，Value），对 Key 进行哈希取模，首先将 Key 转换为数字，之后除以 Reduce
节点的个数取余数，这样就保证 Key 值相同的数据会被分发到一个 Reduce 节点中。如何确定 Reduce
节点个数呢？可以在程序运行前手动指定，默认情况下为 Map 个数的 80%。

相同的单词被汇总到一个节点之后， **Reduce 阶段** 会对 Key 相同的数据进行操作，这里词频统计比较简单，直接将 Value
累加即可。计算得到最终结果，依然以（Key，Value）对来保存；因为每个 Reduce
的计算结果就已经是最终的准确结果了，只不过是最终结果的一部分数据，所以它在存储的时候不需要再进行汇总，而是每个 Reduce
任务会将结果存储为最终结果文件的一个 Block 块。那这样就意味着 Reduce 个数越多，生成的小文件可能也就越多。

整个 MapReduce 提交之后，是一个 Job，用于完成一个计算任务。而 Job 在分布式计算时，又会被拆分为 Map Task、Reduce
Task，用于并发计算。Shuffle 是在 Map 和 Reduce 端共同完成的，不存在 Shuffle Task。

### 执行过程

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201018072301.png)

在对词频统计的过程有了认识之后，来梳理下 MapReduce 的执行过程。首先文件上传到 HDFS 中，会被拆分为 Block
块进行存储，MapReduce 在执行前，会根据配置来进一步细分或者合并 Block 块来增加或减少 Map 的数量，但默认沿用 Block 数量，所以
Split 阶段不做任何操作。Split 处理完文件块后，每个文件块上会分发一个 Map 任务做部分结果的处理，Map
的处理结果为（Key，Value）对，之后因为要将 Key 相同的数据分发到同一个节点，所以 Map 节点会对结果进行哈希取模，生成与 Reduce
节点个数相同的文件存储到硬盘中。

Map 阶段运行结束后，Reduce 节点会主动从各个 Map 节点上拉取属于自己的数据，然后在 Reduce 进行合并，合并后对 Key
相同的数据进行处理，最终处理结果会保存到 HDFS 中，每个 Reduce 会生成最终结果文件的一个 Block 块。

在整个过程中，因为 Shuffle 最为复杂，它由 Map 和 Reduce 共同来完成，所以对 Shuffle 做一个详细的讲解。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201020032839.png)

首先在 Map 端，Map 任务边进行计算边将中间结果写入专用内存缓冲区 Buffer（默认 100M），在缓冲区中对数据同时进行 Partition 和
Sort（先按“key 的 hashcode 对 reduce 任务数量进行取余计算”，对数据进行分区，得到的分区数与 reduce
任务数量相同，分区内再对 key 进行排序），当缓冲区 Buffer 的数据量达到阈值（默认
80%）时，将数据溢写（Spill）到磁盘的一个临时文件中，因为在缓冲区中经过处理，所以这个文件内的数据已经完成分区和排序。

在整个 Map 数据处理阶段，会生成大量的临时文件，这些文件都是小文件，于是需要对这些临时文件合并（Merge）为一个 Map
输出文件，文件内数据同样先分区后排序。

Map 阶段结束之后，Reduce 任务从不同的 Map 节点中的 Map 输出文件中主动抓取（Fetch）属于自己的分区数据，先写入
Buffer，数据量达到阈值后，溢写到磁盘的一个临时文件中。从所有 Map 节点抓取完数据后，Reduce
节点上会生成大量的临时文件，于是要将多个临时文件合并为一个 Reduce 输入文件，文件内数据按 key 排序。之后再对这个文件进行 Reduce
处理，得到最终结果。

因为整个 Shuffle 过程中，会涉及到与磁盘的大量交互和网络传输，所以效率很低；而且普遍的说法是 MapReduce 基于磁盘，其实也是指
MapReduce 在设计上（主要是 Shuffle 阶段）与磁盘交互频繁，效率被大多开发者所诟病。但其实不管是 MapReduce 还是
Spark，Shuffle 都是一个很重的过程，在调优阶段会尽量减少和避免 Shuffle 来提高运行效率。

### 运行模式

MapReduce 的运行模式在 Yarn 这一节中已经讲过，快速回顾一下。

MapReduce 在 hadoop 1.x 中，运行时使用 JobTracker/TaskTracker 模式，JobTracker
为主节点，负责为作业分配 TaskTracker 上的资源，调度任务在 TaskTracker 上运行，若任务失败，指定新 TaskTracker
重新运行。TaskTracker 节点为从节点，负责执行任务，向 JobTracker
发送进度报告。但因为这种架构，单点故障率太高，而且对资源的分配过于粗犷，通用性较差，所以也促使了 hadoop2.x 中的 Yarn 出现。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201020034557.png)

Hadoop 2.x 使用的模式为 YARN 模式，Reource Manager 为客户端提交的任务分配 Node Manager 资源，Node
Manager 将资源封装为 Container，Reource Manager 调度任务到 Container 上执行；程序首先在 Container
中运行 Application Master，解析任务后，向 Resource Manager 为 Task 申请相应的 Container
资源，之后便将任务调度到 Container 中执行，执行过程中 Task 任务实时向 Application Master
汇报；任务结束，Application Master 向 Resource Manager 申请释放资源。Resource Manager
随之清理作业使用的所有 Container。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201020035122.png)

