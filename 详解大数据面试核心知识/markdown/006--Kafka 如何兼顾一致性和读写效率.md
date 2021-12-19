完成了 HBase 的相关内容的梳理，我们把目光转到同样在流数据处理领域应用非常广泛的分布式组件 Kafka，作为一个消息队列，它与分布式数据库 HBase
不同，它的主要功能不是数据的存储，而是在各个系统之间起一个缓冲的作用，言简意赅又不失准确地总结一下就是 **解系统耦合** 和 **流量削峰**
。本篇按照大数据组件学习的方法，从基础讲起，逐层递进，深入底层，掌握原理，助力面试。

**本篇面试内容划重点：一致性与可用性、ISR、存储结构、快的原因。** **

### 进程的职责与交互逻辑？

![image.png](https://images.gitbook.cn/2020-06-11-070109.png)

按照惯例，我们举个有意思的例子，Kafka 的数据是以日志的形式顺序存储的，所以整个 Kafka
集群看起来就像是图书馆，这家图书馆有多家分店（Broker），图书馆管理员（producer）同一本书会买三本（replica）分发到各分店里，这样能保证一家店的书损坏了顾客可以去另一家店继续看书，另外图书馆还有一个图书检索网站（Zookeeper）记录了图书的信息和顾客的信息（元数据），顾客（consumer）可以通过网站找到自己要读的书的位置然后去指定的分店读书。

  * **Broker：** Kafka 的处理节点（即一台物理服务器），一个节点就是一个 Broker，一个 Kafka 集群由一个或多个 broker 组成。
  * **Topic / Partition：** Topic 是 **逻辑上的概念** ，用于 Kafka 对消息进行主题归类，发送到集群的每一条消息都要指定一个 Topic。每个 Topic 包含一个或多个 Partition 分区，一个 Partition 对应一个文件夹，这个文件夹下存储 Partition 的数据和索引文件， **每个 partition 内部有序，partition 之间无序** 。另外为了保证高可用，数据会根据配置冗余存储多份至不同的 Partition。 **Partition 可以提高消费端的并发度。**
  * **Consumer / ConsumerGroup：** Consumer 是消费者，负责从 Broker 读取消息，每个 consumer 属于一个特定的 consumer group，一条数据可以发送到不同的 Consumer group，但对于 **同一条数据一个 Consumer group 只能由其中的某一个 Consumer 消费一次** 。
  * **Producer：** Producer 是数据生产者，负责写入数据到 Broker ，在客户端构建 ProducerRecord(topic, partition, key, value) 对象并序列化，根据参数确定写入分区，然后将相同分区数据作为一个批次，内部线程负责把这些数据按批次发送到相应的 broker 。Broker 写入数据成功则会返回一个 RecordMetaData 对象，包含了 **Topic 、 Partition 、Offset 信息** 。若写入失败，则生产者会收到错误信息，并重新发送，失败次数达到阈值则抛出异常。

### Partition 副本数据的一致性保证？

前面提到了为了保证可用性，Kafka 的分区是多副本的，可以在创建分区时通过 replication-factor 参数 **指定该分区的副本数** ，
**某一副本丢失并不会造成实际数据的丢失** ，从其他副本获取数据即可。但同时引出了另外一个问题，各个副本之间的数据如何保证一致性？

#### 副本类型和属性

首先，分区的副本根据角色的不同可分为：Leader 副本、 Follower 副本

  * **Leader 副本** ：Leader 负责与 Producer 和 Consumer 交互，即 **数据的读写。**
  * **Follower 副本** ： **被动地备份 leader 副本中的数据** ，不与 Client 端做任何交互。

另外，不得不提的是 ISR，即（in-sync Replica）副本同步队列：它包含了 Leader 副本和所有与 Leader 副本保持同步的
Follower 副本。那么如何判断 Follower 副本与 Leader 是同步的？ **Leader 副本和 Follower 副本**
有两个重要的属性值，如图 LEO 日志末端位移和 HW 水位线。

  * LEO（log end offset)：记录 **日志的末端位移值** ，即数据写到的最新的位置。
  * HW（high watermark）：取最小的 LEO 作为 HW，即** Committed 过的最新数据**。consumer 最多只能消费到 HW 所在的位置，因为小于等于 HW 值的数据才是 Committed 备份过的。

![image.png](https://images.gitbook.cn/2020-06-11-070111.png) **LEO 和 HW
的更新的时机** ![image.png](https://images.gitbook.cn/2020-06-11-070113.png) Leader
除了 HW 和 LEO 还会有 **RemoteLEO** 这个属性，意思是 Follwer 的 LEO。Leader 在以下情况下会更新这三个属性。

  * LEO：Producer 端 **有数据写入成功** 时 Leader 会自动地更新 LEO 值
  * HW： **有请求** 的时候（Producer 或 Follower 的请求）会对比 LEO，Remote LEO 取小值更新 HW
  * Remote LEO：Leader 处理 Follower 的 Fetch 请求时，将 remote LEO 更新为 Follwer 请求中附带的 LEO 值。

Follower 只有 HW 和 LEO 两个属性，更新时机为：

  * LEO：同样是 **有数据写入成功** 时就会自动地更新 LEO 值。
  * HW：follower 更新 HW 发生在其 **更新 LEO 之后** ，一旦 follower 向 log 写完数据，它会尝试更新它自己的 HW 值。更新的条件是 **比较当前 LEO 值与 response 中 leader 的 HW 值** ，取两者的小者作为新的 HW 值，如图的 HW:1 会在此时更新为 2。

### Kafka 文件存储基本结构？

每个 Partition 为一个目录，Partition 命名规则为：【topic 名称】+【从 0 开始的有序序号】。
![kafka1.png](https://images.gitbook.cn/2020-06-11-070115.png) 每个 partition
目录包含多个大小相等，但内部数据量不一定相等的 Segment file，便于 **旧数据整个文件删除** 。 Segment file 由 index
file 和 data file 组成， **两个文件成对出现** ，后缀".index"和".log"分别表示为索引文件、数据文件。
![kafka2.png](https://images.gitbook.cn/2020-06-11-070116.png) **数据文件命名规则：**
partion 全局的第一个 segment 从 0 开始，后续每个 segment 文件名为 **上一个 segment 文件最后一条消息的 offset
值** 。数值最大为 64 位 long 大小，19 位数字字符长度，没有数字用 0 填充。 索引文件中元数据指向对应数据文件中 message
的物理偏移地址。

**列如：** 索引文件中元数据为“8,724”，表示 log 文件中的第 8 条数据，物理偏移地址为 724。(从 log 文件中又可以得到这条数据在全局
partiton 表示第 368772 个 message)

#### 查找文件的步骤？

读取 offset = 368776 的 message，需要通过以下步骤查找。

  * 找文件名，只要根据 offset 二分查找文件列表，就可以快速定位到 00000000000000368769.index 这个文件以及对应的 log 文件。
  * 然后在 00000000000000368769.index 文件中找到对应的物理偏移地址，再去 00000000000000368769.log 扫描，因为 kafka 采用的是稀疏索引，不会存每一条数据的偏移，所以找到小于等于目标值最近的偏移，然后从偏移的位置顺序查找直到 offset=368776 为止。

![image.png](https://images.gitbook.cn/2020-06-11-070117.png)

### kafka 能保证高效读写的原因？

**1.在磁盘只做 Sequence I/O 顺序读写** kafka 生产者写数据是有序的，即 Partition 内部有序，数据以 append
的方式顺序追加写入。Consumer 消费数据也是有序的，指定 offset 后顺序读出 offset
之后的数据。顺序读写可以避免磁盘读数据时的多次寻道和旋转延迟。

**2\. Sendfile 零拷贝机制** 一般 read/write 操作

  1. OS 从硬盘把数据读到内核区的 PageCache。
  2. 用户进程把数据从内核区 Copy 到用户区。
  3. 然后用户进程再把数据写入到内核区的 Socket Buffer 上。
  4. OS 再把数据从 Buffer 中 Copy 到网卡的 Buffer 上，这样完成一次发送。

零拷贝机制则省略了图中 2 和 3 的操作，即无需把数据拷贝到用户区。

![image.png](https://images.gitbook.cn/2020-06-11-070119.png)

**3\. pageCache 机制** PageCache 功能是底层操作系统提供的， **用于缓存文件的页数据** ，Kafka
重度依赖它实现大吞吐量和低延迟。 首先相对于 JVM 的缓存， **pageCache** 没有 GC，减少开销，即使 kafka 重启 pagecache
还在，回收 PageCache 的代价小，而且只要生产者与消费者的速度相差不大，消费者会直接读取之前生产者写入 Page Cache 的数据，没有磁盘
io。具体的过程如图所示。 ![image.png](https://images.gitbook.cn/2020-06-11-070120.png)

###  其他相关面试题

  1. **Producer 如何确定写入数据到哪个分区？**

ProducerRecord(topic, partition, key, value) 对象参数有四个，其中主题 Topic 和内容 Value 为必填，
Partition 为指定分区，Key 为指定 Hash 分区的键值，这两个项为选填。 Partitioner 根据上面 ProducerRecord
的参数，对发送的数据进行分区，分区规则如下：

  * 指定 Partition ID，不管有没有指定 Key， ProducerRecord 都会被发送至指定 Partition。
  * 指定 Key，未指定 Partition ID, 则 ProducerRecord 会按照对 key 做 Hash 后，发送至对应 Partition。
  * Partition ID 和 Key 都未指定， ProducerRecord 会按照 round-robin 模式发送到每个 Partition。

  1. **Consumer 和 Partition 个数如何配置才能达到最佳效果？**

Kafka 的设计原理决定，同一个分区只能被同一个消费者群组里面的一个消费者读取，不可能存在同一个分区被同一个消费者群里多个消费者共同读取的情况。所以同一个
ConsumerGroup 下的 Consumer 和 Partition 最好是一一对应，一个 Partition 对应一个 Consumer，如果
Consumer 的数量过多，必然有空闲的 Consumer 无法得到消息。

  3. **Kafka 是如何在可用性和一致性之间做平衡的？**

  * Kafka 可配置“在 Leader 宕机情况下，是否允许某个不完全同步的副本成为 Leader”。若配置为 true 可保证数据一致性，做到数据不丢失，配置为 false，则为了可用性会出现数据丢失的情况。
  * ISR 可配置“至少要有几个可用副本分区才能进行正常的数据读写”。数量多则数据的一致性强，最新的数据会存在多个副本，但是数据的副本间同步会有时间的消耗，且 ISR 副本多则增大了出现实际副本数量少于配置数量分区不可用的概率，可用性降低。

  3. **数据可靠性和持久性保证**

producer 端：Kafka 在生产者上有一个可选的参数 ack，该参数可配置数据写入成功的条件。

  * acks=0 ：无需服务端响应，数据发出即成功。（响应快，可靠性低）
  * acks=1 ： Leader 收到数据后立即响应，数据写入成功。（相对平衡）
  * acks=all ： Leader 收到数据且与所有参与复制的节点数据同步完成，才响应成功。（响应满，可靠性强）

Broker 端：配置多个分区副本以及参考问题 1 的配置。保证 Broker 数据的可靠性。 Consumer 端：自己管理
Offset，数据处理完成后再 commit 偏移，保证消费数据的完整性。

