### HDFS 单节点架构存在的问题

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012050717.png)

HDFS 单点架构存在一些问题，首先是 NameNode 内存受限，因为数据的元数据信息全部保存在 NameNode 内存中。如果数据量足够庞大，可能会将
NameNode 内存占满，这种情况下会导致 HDFS 的扩展性上限。

其次就是单点故障问题，主从架构的 HDFS 是依靠主节点 NameNode 来运转的，一旦主节点挂掉就会导致整个集群不可用。

### Federation（联邦）机制

联邦机制是 Hadoop 2.x 中提出的解决 NameNode 内存瓶颈问题的水平横向扩展方案。

它将多台 NameNode 组成联邦，每一台 NameNode 负责存储一部分元数据信息，共同负责 HDFS 的正常运行。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012051620.png)

联邦机制解决了 NameNode 单点的内存限制，提升了 HDFS 的扩展性能。

### NameNode High Availability（高可用）机制

NameNode High Availability 高可用机制是 Hadoop 2.x 中提出的，用于解决 NameNode
单节点故障问题的方案。实现高可用，至少提供两台 NameNode 做热备：Active、Standby，其中 Active 作为主节点，而 Standby
为备份主节点。当一台 NameNode 宕机，另外一台需要保持元数据一致（fsimage、edits），并且完成状态切换。

QJM 和 NFS 机制是 NameNode HA 的两种实现方式，主流的方案为 QJM。

#### **使用 QJM 实现元数据 edits 文件高可用**

当主节点挂掉后，备份节点为了保证与主节点的元数据一致，需要获取主节点上的 edits 文件，从而恢复最新的元数据。而 fsimage
因为最初集群初始化的时候，在 Active 和 Standby 中是一致的（均为空），而且 Standby 在合并完元数据之后，会在本地保存一份最新的
fsimage。

但主节点挂掉后，磁盘中的 edtis 文件便无法进行获取，稳妥的方案是将 edits 文件存储到第三方集群中，这样主节点的宕机对第三方集群并不产生影响。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012052339.png)

QJM 方案中，使用的三方集群为 JournalNode，它负责存储 edits 编辑日志；一般部署奇数（2N+1），因为它的算法模式基于 Paxos
算法实现，超过半数的 JournalNode 返回成功，就代表写入成功。JournalNode 集群最多可容忍 N 个 JournalNode 宕机。

#### **QJM 高可用方案**

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012052644.png)

在 QJM 高可用方案中，最重要的是角色是 JournalNode 集群，用于存储 edits
文件，实现主备间的元数据同步。除此之外，高可用集群还需要完成主备状态的切换操作，这个过程由 ZooKeeper 来完成。在搭建单点 NameNode 的
HDFS 集群时，ZooKeeper 是非必须的，但在 HA 集群搭建中，必须依赖 ZooKeeper 集群。

集群启动后，ZooKeeper 会启动 FailoverController 用于监控 NameNode 的存活情况，并定期通过心跳向 ZooKeeper
汇报。如果 Acitve NameNode 的状态超过一定时间没有汇报给 ZooKeeper，则 ZooKeeper 会移除它的 Active
状态，从所有的 Standby 节点中选举新的 Active。

在 ZooKeeper 中是实现这种主从切换的？简单来说就是锁。ZooKeeper 为 HDFS 创建一个选举用的目录（当然在 ZooKeeper
中，目录和文件是一回事），这个目录下只能创建一个文件，这个文件就是锁；所有的 NameNode 都来竞争创建这个文件，谁第一个创建完成，则成为 Active
节点，其余的节点为 Standby。当然在分布式环境中，实现起来会比较复杂，如果有兴趣，大家可以去深入了解一下这里的原理。

HDFS 高可用的推出，也意味着大数据技术成熟的标志，HDFS 能够真正投入生产环境进行使用。

