数据湖是近几年来比较热门的一个话题，把这块的内容也加入这个专栏的目的想传达的一个信息，程序员需要对新技术框架有敏锐感知能力，也需要保持一颗好奇心，因为技术肯定是会一直迭代发展的，这一点毫无疑问，所以我们也需要不停地迭代自己的知识，跟上时代的步伐。另外，如果你的简历上有一个新技术的实践经验，这也可以是你的一个加分项。本篇是基于数据湖的具体实现之一
Apache Hudi 来讲解相关内容。

**本篇面试内容划重点：表类型（COW、MOR）、查询类型（视图）。**

Apache Hudi 依赖于 HDFS 做底层的存储，所以可存储的数据规模是巨大的，同时基于以下两个原语，Hudi 可以将流批一体的存储问题。

  * **Update/Delete 记录** ：Hudi 支持更新/删除记录，使用文件/记录级别索引，同时对写操作提供事务保证。查询可获取最新提交的快照来产生结果。
  * **Change Streams** ：支持增量获取表中所有更新/插入/删除的记录，从指定时间点开始进行增量查询，可以实现类似 Kafka 的增量消费机制。

### Hudi 的重要概念

#### **Timeline 时间线**

Hudi 内部对每个表都维护了一个 Timeline，这个 Timeline 是由一组作用在某个表上的 Instant（时刻）对象组成。Instant
表示在某个时间点对表进行操作的，从而达到某一个状态的表示，所以 Instant 包含 Instant Action，Instant Time 和
Instant State 这三个内容，它们的含义如下所示：

  * **Instant Action** ：对 Hudi 表执行的操作类型，目前包括 COMMITS、CLEANS、DELTA_COMMIT、COMPACTION、ROLLBACK、SAVEPOINT 这 6 种操作类型。
  * **Instant Time** ：表示一个时间戳，这个时间戳必须是按照 Instant Action 开始执行的时间顺序单调递增的。
  * **Instant State** ：表示在指定的时间点（Instant Time）对 Hudi 表执行操作（Instant Action）后，表所处的状态，包括
    * **REQUESTED** 已调度但未初始化
    * **INFLIGHT** 当前正在执行
    * **COMPLETED** 操作执行完成

#### **索引**

Hudi 会通过记录 Key 与分区 Path 组成 Hoodie Key，即 Record Key + Partition Path，通过将 Hoodie
Key 映射到前面提到的文件 ID，具体其实是映射到 file_group/file_id，这就是 Hudi
的索引。一旦记录的第一个版本被写入文件中，对应的 Hoodie Key 就不会再改变了。

Hudi 有三种索引类型，HBASE、INMEMORY、BLOOM（GLOBAL_BLOOM）：

  * HBase 类似的索引数据外部索引，需要依赖于 HBase 环境，所以还需要配置 hbaseZkQuorum/hbaseZkPort/hbaseZkZnodeParent/hbaseTableName 等信息。
  * INMEMORY 索引顾名思义，索引存储在内存中，高效但是没有持久化。
  * BLOOM 和 GLOBAL_BLOOM，这两种索引不用依赖于外部的环境，索引信息直接存储在 Parquet 数据文件的页脚中。区别是 BLOOM 分区生效，GLOBAL_BLOOM 全局生效。

#### Hudi 表类型

Hudi 具有两种类型的表。

**1\. Copy-On-Write 表**

Copy-On-Write 不是 Hudi 发明的，为了便于理解先简单解释一下什么是 Copy-On-Write？Copy-On-Write
是一种较为普遍和通用的存储优化策略，它在 Linux 和 JDK 中都有使用，可以翻译为 **写时拷贝** 。在 Wiki 中的定义如下：

>
> 如果有多个调用者（callers）同时请求相同资源（如内存或磁盘上的数据存储），他们会共同获取相同的指针指向相同的资源，直到某个调用者试图修改资源的内容时，系统才会真正复制一份专用副本（private
> copy）给该调用者，而其他调用者所见到的最初的资源仍然保持不变。这过程对其他的调用者都是透明的（transparently）。

简单来说就是在程序运行阶段，所有线程（A、B、C）共享同一个内存数据，直到其中的一个线程 B 需要对数据进行修改时，那么系统将会复制一份当前的数据，此后线程
B 所做的数据修改都是在复制的数据段上进行的。这个过程对所有的线程都是透明的，也就是说 AC 两个线程使用的还是原来的内存数据，当线程 B
将数据更改后，系统将数据指针移向修改后的数据段上，这样在所有线程都不知情的情况下完成了数据的更新操作。

Hudi 的 Copy-On-Write 表和上述的原理相似，它使用 Parquet 格式存储， **更新时会保存多个版本** ，并且在写的过程中通过异步的
Merge 来实现重写（Rewrite）数据文件（Base 文件）。所以 Hudi
每次写数据都会产生新的列文件副本。在新的副本上进行写入操作，这种方式带来的问题就是在每次数据写入时，都需要重写整列数据文件，哪怕只有一个字节的数据写入（也就是所谓的写放大）。

每次新数据的写入，都会基于当前的数据文件产生一个带有提交时间戳的新副本文件，新数据会插入到当前的新副本文件中，直到整个操作没有完成前，所有的查询操作都不会看到这个新的文件副本。在新文件数据写入提交后，新的查询将会获取到新的数据。因此，查询不受任何写入失败/部分写入的影响，仅运行在已提交数据上。另外，COW
没有合并的说法，就是直接存储的数据文件和 Rewrite。

**来看看下图的官方例子** ：每次执行 INSERT 或 UPDATE 操作，都会在 Timeline 上生成一个的
COMMIT，同时对应着一个文件分片（File Slice）。如果是 INSERT 操作则生成文件分组的第一个新的文件分片，如果是 UPDATE
操作则会生成一个新版本的文件分片。写入过程中可以进行查询，如果查询 COMMIT 为 10:10 之前的数据，则会首先查询 Timeline 上最新的
COMMIT，通过过滤掉只会小于 10:10 的数据查询出来，即把文件 ID 为 1、2、3 且版本为 10:05 的文件分别查询出来。

![image.png](https://images.gitbook.cn/ecb23950-1124-11eb-a727-b74ce73d9726)

**COW 表存在的问题：**

  * 数据延迟较高：只能够保证数据的最终一致性，不能够保证数据的实时一致性。
  * io 请求较高：每次数据的更新写入需要重写整个 Parquet 文件。
  * 写放大：每次写入数据，哪怕只有一字节，那么都需要重写整个文件。

**2\. Merge-On-Read 表**

MOR 可以称为 **读时合并** ，这个可以说是 Hudi 特有的属性了，读时合并指的是合并实时数据和历史数据，提供一个最新的读视图。

从底层来说，MOR 在列存 Parquet
的基础上增加了基于行的增量日志文件（Avro）写入功能，也就是说在当前数据文件更新前将对应的数据先写入到日志文件中。然后再通过一定的策略进行
COMPACTION 操作来将增量文件（Avro）合并到 Base 文件（Parquet）上。

总结一下也就是说：使用列式和行式文件格式混合的方式来存储数据，列式文件格式比如 Parquet，行式文件格式比如
Avro。更新时写入到增量（Delta）文件中，之后通过同步或异步的 COMPACTION 操作，生成新版本的列式格式文件。

来看一下官方给的例子，如下图，每个文件分组都对应一个增量日志文件（Delta Log File）。COMPACTION
操作在后台定时执行，会把对应的增量日志文件合并到文件分组的 Base 文件中，生成新版本的 Base 文件。对于查询 10:10 之后的数据的 Read
Optimized Query，只能查询到 10:05 及其之前的数据，看不到之后的数据，查询结果只包含版本为 10:05、文件 ID 为 1、2、3
的文件；但是 Snapshot Query 是可以查询到 10:05 之后的数据的。

![image.png](https://images.gitbook.cn/01567ec0-1125-11eb-86c5-33053ff1297d)

#### **Hudi 查询类型**

**1. 快照查询 Snapshot Query **

只能查询到给定 COMMIT 或 **COMPACTION 后** 的最新快照数据。

  * Copy-On-Write 表，Snapshot Query 能够查询到：已经存在的列式格式文件（Parquet 文件）；
  * Merge-On-Read 表，Snapshot Query 能够查询到：通过合并已存在的 Base 文件和增量日志文件得到的数据（Parquet 文件 + Avro 文件）。

**2\. 读取优化查询 Read Optimized Query**

  * Copy-On-Write 表，和快照查询一样。
  * Merge-On-Read 表，只能查询到给定的 COMMIT/COMPACTION 之前所限定范围的最新数据。即只能读到列式格式中的最新数据。这种方式不需要合并行和列数据（通俗地说就是上面的快照查询去掉了“行”的部分，只返回“列”的部分），所以拥有比快照查询更好的性能，但是实时性方面会打折扣，因为少读了未合并的数据。

**3\. 增量查询 Incremental Query**

只能查询到最新写入 Hudi 表的数据，也就是给定的 COMMIT/COMPACTION 之后的最新数据。

  * Copy-On-Write 表：在给定的开始，结束即时时间范围内，对最新的基本文件执行查询（称为增量查询窗口），同时仅使用 Hudi 指定的列提取在此窗口中写入的记录。
  * Merge-On-Read 表，查询是在增量查询窗口中对最新的文件片执行的，具体取决于窗口本身，读取基本块或日志块中读取记录的组合。

