继续沿着“祖师爷” **Doug Cutting **的大数据之路，我们要讲的第二个分布式存储组件是 Apache
HBase，它的理论基础来自《Bigtable》同样是 Google 的三大经典大数据论文之一，底层依赖于上一个讲解的组件
HDFS，但是通过对内存的高效合理运用，它拥有了毫秒级的读写能力，能够适应更多不同的使用场景。把它放在第二位来讲可见它的重要性，在大数据面试中，HBase
是必考题。

**本篇面试内容划重点：Compact、Split、LSM-Tree、MVCC。**

### 进程的职责以及如何协调工作？

#### 进程的职责？

HBase Active 和 Backup Master 通过 Zookeeper 来做 failover ，Client 通过 Zookpeer
的元数据与 RegionServer 进行交互，这种架构算是比较常规，理解即可。然后考察的重点来了， RegionServer
内部的相关操作，存储的形式，结构，compact，split 等等，都是需要着重掌握的。 **HMaster**

  * 正常状态下，集群中同一个时间只存在一个 Active HMaster 。
  * 负责处理关于表 **schema 相关的客户端操作请求** 。
  * 为 RegionServer 分配 Region 并负责 RegionServer 上 **Region 数量的平衡** （负载均衡）。
  * 发现宕机的 RegionServer 并重新分配其上的 Region 至其他健康 RegionServer。

**Client**

  * Client 首先会连接 Zookeeper 获取 HBase 的集群信息。
  * 在读写的过程中会在本地 **缓存** Region 的位置信息，以加快后续对 HBase 的访问。

**Zookeeper**

  * HMaster 的 **主备容错** 。
  * 存储 RegionServer **状态信息** 。
  * 记录 .META. 表所在的位置。
  * 存储 HBase 的 **schema 和 table 元数据** 。
  * 和 HMaster 配合发现失效的 Region，并重新分配 Region。

**HRegionSever**

  * 维护 Region，接收并处理来自客户端的 region 读写请求，并维护与客户端的连接。
  * 负责 **切分（split）** 运行过程数据量达到阈值的 region。
  * 健康检测并与 zookeeper **维持心跳** 。

#### HRegionServer 的构成？

![image.png](https://images.gitbook.cn/2020-06-11-070027.png) HRegionServer
从图中也可以看出来，里面的内容非常丰富，像套娃一样，但是层层拨开都是不同的，给人“惊喜”。类似“一个 XXX 包含了几个 XXX
？”这种句式的问题在这里会说得明明白白，所以碰见这类问题要有条件反射“这不就是套娃那个知识点嘛，简单”。

**HRegionServer 包含 1+ 个 Region**

  * 一个 HRegionServer 包含一到多个 Region，而 Region 就是一张 HBase 表按一定阈值横向切割的一部分。
  * Region 是按大小分割的，每个表开始只有一个 region，随着数据增多，region 不断增大，当增大到一个阈值的时候，region 就会等分会两个新的 region，以此类推，之后会有越来越多的 region。
  * Region 是 Hbase 中分布式存储和 **负载均衡** 的最小单元，不同 Region 分布到不同 RegionServer 上;

**Region 包含 1+ 个 Store （即 columns family 列族个数）**

  * Region 由一个或者多个 Store 组成，每个 store 保存一个 columns family;
  * Store 是一个抽象的概念，简单地说就是一个存储，它的个数和 **HDFS 上的存储目录个数** 是一致的，而一个存储目录对应的就是一个 columns family 列族。
  * **Region 达到阈值会分裂** ，分裂后由 HMaster 分配到不同 server 起到负载均衡作用（热点问题后一篇会详细解析）。

**Store 包含 1 个 MemStore 和 0+ 个 StoreFile**

  * Store 上面说了就是一个存储，他包含了一个内存的存储和 0+个文件存储，一个 Store 的所有文件都存在一个目录下，这个目录下的所有文件对应的是一个列族。
  * StoreFile 是实际存储数据的对象，是** HFile 的轻量级包装**，存储在 HDFS 上的。
  * StoreFile 达到阈值个数会进行 **合并（compact）** 。

#### HRegionServer 宕机后的可用性保证？

可用性是个老生常谈的问题，也是个非常重要的问题，特别是系统比较复杂的情况下，一定要把每一个步骤都讲清楚。

  * HRegionServer 会向 Zookeeper 实时上报 **心跳信息** ，确保 ZooKeeper 能及时发现 HRegionServer 宕机。
  * HRegionServer 宕机则其负责的 Region **全部失效停止对外提供服务。**
  * HMaster 收到 HRegionServer 宕机的通知，它的职责是 **重新分配 region** ，分配的方式是把 region 信息放在 Zookeeper 等健康的 Regionserver 来获取。这样成功把 region 分配至其他 HRegionServer。
  * 同时 HMaster 还会对 HRegionServer 上存在 memstore 中还未持久化到磁盘中的数据 **通过 WAL 来进行恢复** ，WAL 就是一个存储在 HDFS 上的文件，用于记录客户端对 HRegionServer 数据的操作指令。

### HBase 的读写详细解析

![image.png](https://images.gitbook.cn/2020-06-11-070028.png)

#### 写流程（包括 StoreFile 的 Compact、Region 的 Split）

  1. 向 zookeeper 发起请求，获得** META 所在的 region**，再根据 table、namespace、rowkey 信息去 META 表中找到目标数据对应的 Region 信息以及 Regionserver；（ROOT 表从 0.96 版本开始已经被淘汰）
  2. 把数据分别写到 HLog 和 MemStore 上各一份；

  * MemStore 达到一个阈值后则会把数据刷成一个 **StoreFile** 文件落到磁盘；
  * 同时将 **内存中的数据删除** ，并 **删除 Hlog** 中的历史数据。
  * 在 Hlog 中做 **标记点** ，若 MemStore 中的数据有丢失，则可以从 HLog 上恢复；

  1. 当多个 StoreFile 文件达到一定的大小后，会触发 Compact 合并操作，合并为一个 StoreFile，这里同时进行已标记删除数据的 **版本合并** 和 **实际数据的删除** 。
  2. 当 Compact 后，逐步形成越来越大的 StoreFile 后，Region 也会达到 Split 的阈值，会触发 Split 操作，把一个大的 region 分割成两个 region（细粒度来看其实也是 StoreFile 的分割）。

##### **storefile 的 compact 合并流程？**

在 hbase 中每当有 memstore 数据 flush 到磁盘之后，就形成一个 storefile，当 storeFile
的数量达到一定程度后，就需要将 storefile 文件来进行 Compact 操作。Compact 的作用如下：

  * **清除过期，多余版本的数据。**
  * 合并文件， **减少需要检索的文件数量** ，提高读数据的效率。

HBase 中实现了两种 Compact 的方式：minor and major. 这两种 Compact 方式的区别是：

  * Minor 操作会获取相邻的 **部分小 StoreFile** **来执行合并操作，** 不做 **清理多版本数据和删除数据** 的操作，尽量 **不影响集群的正常工作** 。
  * Major 操作是对 Region 下的 Store 的 **所有 StoreFile 执行合并操作，输出成一个 StoreFile** ，这是一个比较耗费资源的操作，所以 **不宜频繁 Major Compact** 。

##### **region 的 split 分裂流程？**

Region 切分是一个 **事务过程** ，分成三个阶段

  * **prepare 阶段** ：在内存中初始化两个子 region，具体是生成两个 HRegionInfo 对象，包含 **tableName、regionName、startkey、endkey** 等信息。同时会生成一个用来 **记录 Split 进展** 的对象。
  * **execute 阶段** ：
  * 首先更改当前 Region 在 Zookeeper 中的状态为 SPLITING。master 也会同步这个状态。
  * 生成两个子文件，只存储 **切分点 splitkey 和一个 Boolen 类型变量** （用来标记这个文件是上半部分还是下半部分）。
  * 为避免数据的频繁读写，只有在子 Region 执行 **Major Compact** 后才会将父 Region 中属于该子 Region 的所有数据读出来并写入数据文件中。
  * **rollback 阶段** ：如果 execute 阶段出现异常，则执行 rollback 操作。为了实现回滚，整个切分过程被 **分为很多子阶段** ，回滚程序会根据当前进展到哪个子阶段来清理对应的垃圾数据，根据切分进展来做不同的回滚操作。

**Region 的 Split 过程中的切分点如何定位？** 切分点 SplitPoint 是整个 **Region 中最大 Store * _中的_
*最大 StoreFile * _中_ *最中心的一个 Block **的 **首个** **Rowkey** 。如果定位到的 rowkey
是整个文件的首个 rowkey 或者最后一个 rowkey 的话，就认为没有切分点。

#### 读流程（rowkey 定位流程）

  1. 同样从 zookeeper 获得 meta 表所在 region 位置，再根据 table、namespace、rowkey 去 meta 表中获取读对象所在的 RegionServer。
  2. 找到 RegionServer 之后，首先用 MemStoreScanner **搜索 MemStore 里是否有所查的 rowKey** （这一步在内存中，很快）；
  3. 如果没有则需要定位 HFile 了，具体流程是：

  * 用 Bloom Block 通过算法过滤掉大部分一定不包含所查 RowKey 的 HFile；（ **Bloom Block 在内存中** ）
  * 另外，内存中还会记录每个 HFile 的偏移量，可以快速排除掉剩下的部分 HFile；（ **偏移量在内存中** ）
  * 经过上面两步，剩下的就是很少一部分的 HFile 了，就需要根据 Index Block 索引数据快速查找 Rowkey 所在的 Block 的位置；（ **索引数据在内存中** ）
  * 找到 Block 的位置后，检查这个 Block 是否在 BlockCache 中，在则直接去取，如果不在的话把这个 Block **加载到 BlockCache 进行缓存** ，当下一次再定位到这个 Block 的时候就不需要再进行一次 IO 将整个 Block 读取到内存中。
  * 最后扫描这些读到内存中的 Block（ **可能有多个，因为有多版本** ），返回需要的版本。

  1. 整个过程完成返回数据到客户端之后，客户端会对 Block 数据块缓存。

##### **HBase 如何保证读效率？**

不同于 HDFS，HBase 是个对读有比较高要求的系统，所以在保证高效读的方面，HBase 还是做了很多优化的。

  * **缓存**

** **HBase 有两块主要的内存缓存，** MemStore 和 BlockCache。**

  * 一个查询过来 regionserver 后，首先用 MemStoreScanner 搜索 MemStore 里是否有所查的 rowKey ，这一步在内存中，所以是很快的。
  * 如果不在 memstore 中，会经过一系列的索引寻址 **定位到 block 的位置** 。
  * 如果 block 在 BlockCache 缓存中则可以直接在内存中操作，速度很快，不需要再进行一次 IO 将整个 block 读取到内存中。
  * **过滤**
  * RegionServer 启动的时候就会把每个 HFile 的 **起止 rowkey 加载到内存中** ，在定位 HFile 的时候可以过滤掉大部分 HFile；
  * 同时同样是加载到内存的 Bloom Block 也会通过说的 bloomFilter 也会过滤掉大部分一定不包含所查 rowKey 的 HFile。
  * **索引**
  * 经过了上面的过滤，其实只剩下很少一部分的 HFile 需要去检索了，HBase 有三级索引， **第一级索引会常驻内存** ，二三级的索引会以 block 的形式存在 HFile 中。
  * 另外因为 HBase 是多版本共存的，所以结果可能是会有多个的，因此检索的过程不是找到一个就返回了，而是要找到所有的，然后将 **结果合并** 。

### HBase 的 LSM Tree

LSM Tree 即是日志结构合并树。

  * **日志结构** ：日志的特点是它是 **顺序追加写** 的，可以保证 **非常好的写操作性能** ，但是从日志文件中读一些数据将会比写操作需要更多的时间，需要倒序扫描来找到所需的内容。LSM Tree 是通过 **把随机写的数据写到内存，然后定期 flush 到磁盘，对于磁盘来说，让所有的操作顺序化，而不是随机读写** 。
  * **合并树** ：LSM Tree 的原理是把一棵大树拆分成 N 棵小树，它首先写入内存中即是小树，随着小树越来越大，会 flush 到磁盘中，磁盘中的树定期可以做 **merge **操作，合并成一棵大树，以优化读性能，如下图所示。

**![image.png](https://images.gitbook.cn/2020-06-11-070029.png)****LSM-Tree
结构写比读快的原理** LSM-Tree 结构写入快的原因是它将对数据的修改 **增量保持在内存中** ，达到指定的大小限制后才将这些修改操作
**批量写入磁盘** 。 读取的时候会比较麻烦，需要 **合并磁盘中历史数据和内存中最近修改操作** ，所以写入性能大大提升，读取时可能需要先看
**是否命中内存** ，否则需要访问较多的磁盘文件。极端的说，基于 LSM 树实现的 HBase 的写性能比 MySQL
高了一个数量级，读性能低了一个数量级。

### HBase 保证数据的强一致性的机制

有三个方面要注意，另外 HBase 是牺牲了数据的部分可用性来保证它的数据强一致性的，即 CAP 原理中舍弃了一部分的可用性，HBase 是个 “cp”
系统。

  * HBase 中每一条数据只会出现在一个 Region，它的数据冗余备份不是在 region 这个层面做的，还是依赖 HDFS 来做的冗余。而且 **同一时间一个 Region 只会被分配给一个 RegionServer，这就保证了系统中只会有一条可以使用的数据** 。
  * HBase 支持 **行级事物** ，即一个 put 操作要么成功，要么失败。
  * 另外当有 RegionServer 宕机的时候，Region 会被分配到其他的 RegionServer 上，同时重写 WAL Log，这个过程中整个 Region 中的数据是不可用的，因为它是缺失的。 **如果可用性强的话那么必定会有数据不一致的问题（即写入过的数据查询不到），所以这里用可用性来换取了强一致性** ，等到 WAL 写完，保证了数据完整性之后，才可重新访问。

### MVCC 多版本并发机制

MVCC(Multi Version Consistency Control)，简单地说，是一种通过 **数据的多版本来解决读写一致性问题**
的解决方案。我们知道 HBase 是会保留多版本的数据的，每次写入都会产生一个新版本的数据，每次读取都会默认读最新版本的数据，那么 HBase
是在并发请求的场景下是怎么控制这些多版本的呢？
![image.png](https://images.gitbook.cn/2020-06-11-070030.png)
首先如图所示，LinkedList 每个元素里面有两个属性：

  * **writeNumber** ：即 Region 级别的 **事务 ID** ，每个客户端请求都会分配一个事务 ID。
  * **completed** ： **数据写入是否完成** ，初始状态为 Flase，数据写入成功后会更新为 True。

  1. 客户端写入事务请求到达 Region，先写入  到 LinkedList 中，10 是当前事务的 ID，False 表示当前事务还在进行中， **数据还不可读** 。
  2. Client 将数据写入 **memstore 和 WAL** ，写入完成即可结束事务。
  3. 将 completed 更新为 true，表示事务结束。
  4. 同时，Client 会按顺序遍历 LinkedList 里的元素，若 completed:true 则将 readPoint 更新到这个位置， **说明此处的数据是可读的** ，遍历到 completed:false 则停止。
  5. 此时 **数据写入还不会返回成功** ，即事务 10 还是不可读的状态，因为需要 **保证时序** ，client2 和 3 还在写事务 7 和 9 没有完成，当前可读的数据只到事务 6 的位置。等到 client2 和 3 完成事务并将 readPoint 更新到 10，则事务 10 返回写入成功，数据可读。

关于 MVCC 更多细节的内容在分布式理论的模块会详细讲解。

### 分享交流

为了方便与作者交流与学习，GitChat 编辑团队组织了一个专栏读者交流群，添加小助手，回复关键字【1043】给小助手伽利略获取入群资格。
![R2Y8ju](https://images.gitbook.cn/2020-07-23-%E9%A6%99%E9%A6%99.png)

