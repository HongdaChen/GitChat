### HDFS 系统架构

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201010065252.png)

HDFS 是主从架构（Master/Slave），当然这也是大数据产品最常见的架构。主节点为 NameNode，从节点为 DataNode。其中
DataNode 用于存储数据，存储的数据会被拆分成 Block 块（默认按照 128M 进行切分），然后均匀的存放到各个 DataNode
节点中，为了保证数据安全性，这些 Block 块会进行多副本的存储，备份到不同的节点。而 NameNode
则负责管理整个集群，并且存储数据的元数据信息（记录数据被拆分为哪几块，分别存储到了哪个 DataNode 中）。DataNode 会通过心跳机制，与
NameNode 进行通信（默认 3 秒），汇报健康状况和存储的 Block 数据信息，如果 NameNode 超过一定时间没有收到 DataNode
发送的心跳信息，则认为 DataNode 宕机，会启动容灾机制。

HDFS Client 是客户端，客户端通过与 NameNode 进行交互，从而实现文件的读写等操作。

NameNode 在没有实现高可用的时候，会存在两个角色，NameNode 和 Secondary NameNode，其中 Secondary
NameNode 并不是 NameNode 的热备节点，它只用于辅助 NameNode 的工作（元数据合并），之后会详细讲到。

NameNode 如果实现了高可用，则 NameNode（Active）和 NameNode（Standby）互为热备节点，其中备用节点 Standby
NameNode 至少存在一个，除了保证 Active NameNode 宕机后能接替集群管理工作，还用于辅助 NameNode
的工作。但大多数时间，Standby NameNode 处于空闲状态。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201010070850.png)

接下来按照文件的大致存储过程对 HDFS 中的各个组件进行详细的讲解。当一个 10G 的文件要存储到 HDFS 中的时候，首先在 HDFS Client
客户端会将文件拆分为 Block 块（默认按照 128M 进行拆分），注意文件拆分过程是在客户端进行，这样在大数据场景下可以减轻主节点 NameNode
的压力，拆分后的 Block 块会被存储到 DataNode 中，并且以多副本的形式保存到不同节点。数据存储完成后，NameNode
节点会记录一份元数据，包含存储的文件名，文件被拆分成了哪些 Block 块，每个 Block 块又被存储到了哪些 DataNode 节点。

之后客户端在访问文件的时候，便可以通过 NameNode 的元数据文件，迅速找到 Block 块的存储位置，并读取到客户端，在客户端本地将 Block
块按照顺序进行合并，合并成原文件后返回。这也是 HDFS 不适合低延时数据访问的原因。

### HDFS 存储机制

#### **Block 文件存储**

**1\. Block 存储**

在 HDFS 中 Block 是 HDFS 的最小存储单元，它和元数据分开存储：Block 存储于 DataNode，元数据存储于 NameNode。

Block 默认大小为 128M，但在早期 HDFS 1.x 中默认是 64M，它作为文件拆分的依据。那假设一个文件大小正好为 129M，它在 HDFS
Client 拆分的时候，会被拆分为 2 个 Block 块，一个 128M，剩下一个 1M，但是在 HDFS 中，因为 Block
是最小存储单元，它并不关心每个 Block 块中存储的数据究竟有多大，它只认 Block 块数；所以它在数据计算时，会粗狂的认为这个文件的大小为
128M*2=256M，这也是为什么 HDFS 不适合小文件存储，因为 Block 块是 1M 还是 128M，对于 HDFS
来说是一样的处理过程；所以一个 128M 的文件被拆分为 128 个 1M 的 Block，和被拆分为 1 个 128M 的 Block
进行存储，那它带来的压力显然是不一样的，后者在处理时花费的时间是前者的 128 倍。

Block 的默认大小是可以设置的，但不推荐去更改。Block 大小的设置是有规范的，它的目标是为了最小化寻址开销，降到 1% 以下，以前服务器配置大多为
5400 转磁盘，60~90MB/s，所以设置为 64M 可以一次将 Block 块读取出来；而现在常用 7200 转磁盘，读写大致在
130~190MB/s，所以 128M 也非常适用。

Block 块在存储时，会进行多副本备份，它默认的副本数为 3（1 原数据 + 2 副本），HDFS
在存储副本时，会进行自动机架感知，将副本存放到不同机架的 DataNode 上，实现数据的高容错。并且会在所有 DataNode
中实现副本的均匀分布，减少数据倾斜的出现。

**2\. Block 文件**

Block 文件最终会存储到每台 DataNode 节点的磁盘中，是 DataNode 本地磁盘中以“blk_”为前缀的 Linux 文件。DataNode
在启动时自动创建存储 Block 文件的存储目录，无需格式化。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201010080433.png)

Block 数据在存磁盘的时候，除了数据文件，还有一个元信息文件（*.meta），它包含文件的基本属性（文件大小、类型、权限等）和一系列校验值组成。

**3\. Block 副本放置策略**

为了保证 Block 数据的安全性，在第一个副本放置时，会随机选择 DataNode 节点进行存放，优先选择空闲的 DataNode 节点。对于
Client 在 DataNode 节点的（登录到 DataNode 节点，调用客户端进行文件上传），直接存放在当前节点。

如果存在多个机架，副本 2 会放在不同的机架节点上，副本 3 放在与第二个副本同一机架的不同节点上。为什么第 3
个副本不存放到其他机架上呢？因为其实前两个副本放置在不同机架上已经足够保证数据的一致性了，副本跨机架存储是需要一定时间的，第 3 个副本放置在第 2
个副本的同机架上，就是为了保证存储速度。

如果只有单机架，会随机选择空闲节点进行存储。

超过 3 个副本，也会随机选择空闲节点进行存储。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201010080824.png)

#### **元数据存储**

**1\. 元数据**

元数据（Metadata）存放在 NameNode 内存中，包含 HDFS 中文件及目录的基本属性信息（如拥有者、权限信息创建时间等）、文件有哪些
Block 构成、以及 Block 的位置存放信息。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201010083303.png)

**2\. 元数据的持久化**

为了保证元数据的安全性，需要将元数据持久化到磁盘中。

元数据在持久化存储时，只存储文件基本属性信息、文件有哪些 Block 块构成，至于 Block
的位置信息并不会做持久化。因为在大规模集群中，DataNode 宕机是很常见的事情，所以 Block 的位置信息也会随时变动，而如果在磁盘中保存了
Block 的位置信息，势必会在磁盘中产生大量的随机修改，对 NameNode 是很大的负担。而 Block 位置信息实际上并没有存储的必要，它会随着
DataNode 的心跳（每 3 秒一次）进行上报，DataNode 会汇报当前节点所存储的所有 Block 块，根据这些信息，NameNode
可以动态在内存中将 Block 的位置信息补全。

元数据信息持久化到磁盘，会生成文件
fsimage（元数据镜像检查点文件），但如果只有这一个文件的话，在海量数据规模场景中会出现一些问题。在内存中的元数据文件随着用户的操作，势必会发生变动（对文件进行删除、追加），那对应到磁盘的
fsimage 也会进行频繁的随机修改，这样对于 NameNode 会造成不小的负载。

所以 HDFS 在设计时，还存在一个 edits 文件（编辑日志文件），用于应对这种情况，fsimage
保存的是某个时间点的元数据，之后对元数据的所有修改，都以日志的方式记录到 edits
文件中。因为记录日志的方式，就是将操作记录顺序追加，而对于磁盘，顺序读写效率要远高于随机读写，这样就避免了随机读写造成的主节点负载。

每次 HDFS 集群启动时，NameNode 会将 fsimage 直接加载到内存中，映射成元数据，此时元数据还是较老的版本；之后再加载 edits
文件，按照日志内容依次对元数据进行操作，从而还原最新的元数据。这样的话，edits 文件不能特别大，因为 edits 越大，意味着还原元数据的时间越长，产生
NameNode 假死的情况（实际上是在还原元数据）。所以要定期将 edits 的内容合并到 fsimage 文件中，从而保证 edits
文件的大小不超过某个量级。

**3\. 元数据合并**

元数据合并的操作，与 HDFS 集群是否做了高可用有关。如果集群没有做高可用，那么就由 Secondary NameNode 来完成元数据的合并（edits
+ fsimage）；如果集群做了高可用，就由 Standby NameNode 来完成。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012034428.png)

在集群没有做高可用的情况下，Secondary NameNode 会定期将 NameNode 节点的 edits 和 fsimage
文件，通过网络拉取到内存中，将其合并成最新的 fsimage 文件。而主节点 NameNode 的 edits 文件被拉取的时候，会生成一个
edits.new 文件用于保存在合并期间对元数据的所有操作记录。

Secondary NameNode 将合并之后的 fsimage 推送到 NameNode 节点后，会替换掉原有的 fsimage；此时
edits.new 文件也会成为新的 edits 文件。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012035043.png)

集群做了高可用（QJM 方式）之后，元数据文件 edits 除了本地会保存之外，也会被远程存储到第三方集群 JournalNode
中，以便于进行主备切换时，元数据信息不丢失。所以 Standby NameNode 在进行元数据合并时，只需要从 JournalNode 中取到 edits
文件，进行合并即可。而 Active NameNode 和 Standby NameNode 中在初始化启动时，fsimage
是保持一致的（都为空），Standby NameNode 在合并完元数据后，将 fsimage 保存一份在本地，并直接推送到 Active NameNode
中即可。

