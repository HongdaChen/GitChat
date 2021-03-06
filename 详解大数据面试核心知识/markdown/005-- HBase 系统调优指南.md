上一篇用 4000 字的长文介绍了 HBase 的各种重要的面试知识点，原理方面的讲解已经相当地全面了，但这里仍然需要再单独拿出一个章节的篇幅来写
HBase 另外一大块非常重要的内容 ——
调优，可见这块内容在面试过程占据的重要地位。在任何面试中，系统调优能力都是衡量优秀工程师的很重要的一个指标，所以各位同学们振奋一下精神，把下面的内容都吃透了，在面试的过程中才能从容应对。

**本篇面试内容划重点：BLOOMFILTER、预分区、数据倾斜、rowkey 设计。**

### 关于表参数的调优 ？

HBase 虽然没有字段信息也没有类型的限制，但是建表的时候还是有很多需要注意的地方的，合理地配置表信息可以使你写的程序更高效地使用 HBase 。

#### BLOOMFILTER 布隆过滤器

默认值为 NONE，布隆过滤器的作用是可以过滤掉大部分不存在目标查询值的 HFile（即略去不必要的磁盘扫描），可以有助于降低读取延迟。 **配置方式：**
`create 'table',{BLOOMFILTER =>'ROW |``ROWCOL``'}`

  * ROW，表示对 Rowkey 进行布隆过滤，Rowkey 的哈希值在每次写入行时会被添加到 **布隆过滤器** 中，在读的时候就会通过 **布隆过滤器过滤掉大部分无效目标。**
  * ROWCOL 表示 **行键 + 列簇 + 列** 的哈希将在每次插入行时添加到布隆。

#### VERSIONS 版本

默认值为 1 ，我们知道 HBase 是一个多版本共存的数据库，多次写入相同的 **rowkey + cf + col**
则会产生多个版本的数据，在有些场景下多版本数据是有用的，需要把写入的数据做一个对比或者其他操作；但是如果我们不想保留那么多数据，只是想要覆盖原有值，那么将此参数设为
1 能节约 2/3 的空间。所以第一步搞清楚你的需求， **是否需要多版本共存** 。

**配置方式：** `create 'table',{VERSIONS=>'2'}`

#### COMPRESSION 压缩方式

默认值为 NONE ，常用的压缩方式是 Snappy 和 LZO，它们的特点是用于 **热数据压缩，占用 CPU
少，解压/压缩速度较其他压缩方式快，但是压缩率较低** 。Snappy 与 LZO 相比，Snappy 整体性能优于 LZO，
**解压/压缩速度更快，但是压缩率稍低** 。各种压缩各有不同的特点，需要根据需求作出选择。 另外，不知道选什么的情况下，我建议选 Snappy，毕竟在
HBase 这种随机读写的场景， **解压/压缩速度** 是比较重要的指标。

**配置方式：**

`create 'table',{NAME=>'info',COMPRESSION=>'snappy'}`

#### TTL 过期时间

默认值为 Integer.MAX_VALUE ，大概是 64 年，即约等于 **不过期**
，这个参数是说明该列族数据的存活时间。超过存活时间的数据将在表中不再显示，待下次 major compact
的时候再彻底删除数据。同样需要根据实际情况配置。

**配置方式：** `create 'table',{NAME=>'info', FAMILIES => [{NAME => 'cf',
MIN_VERSIONS => '0', TTL => '500'}]}`

#### HBase 预分区

在前面的内容中我们了解到，默认情况下，在创建 HBase 表的时候会自动创建一个 Region 分区，当导入数据的时候， 所有的 HBase
客户端都向这一个 Region 写数据，直到这个 Region
足够大了才进行切分。这样的局限是很容易造成数据倾斜，影响读写效率。预分区是一种比较好的解决这些问题的方案，即 **预先创建一些空的 regions**
，定义好每个 region 存储的数据的范围，这样可以有效避免数据倾斜问题，多个 region 同时工作，使得可以在集群内做数据的 **负载均衡** 。

  * **指定 rowkey 分割的点**

`create 'table1','f1',SPLITS => ['\x10\x00', '\x20\x00', '\x30\x00',
'\x40\x00']`

  * **rowkey 前缀完全随机**

`create 'table2','f1', { NUMREGIONS => 8 , SPLITALGO => 'UniformSplit' }`

  * **rowkey 是十六进制的字符串作为前缀的**

`create 'table3','f1', { NUMREGIONS => 10, SPLITALGO => 'HexStringSplit' }`

其中，NUMREGIONS 为 region 的个数，一般按每个 region 10GB 左右来计算 region 数量，集群规模大，region
数量可以适当取大一些。

#### 关于缓存配置

**之前提到过，BLOCKCACHE** 是读缓存，如果该列族数据顺序访问偏多，或者为不常访问的冷数据那么可以关闭这个
blockcache，这个配置需要谨慎对待，因为对读性能会有很大影响。**

**配置方式：**

`create 'mytable',{NAME=>'cf',BLOCKCACHE=>'false'}`

### 如何合理设计 HBase 的表结构？

#### 1\. Rowkey 设计

##### **rowkey 长度**

Rowkey 是一个 **二进制码流，建议越短越好** ，一般不超过 16 个字节，主要是出于以下的考虑：

  1. 数据的持久化文件 HFile 中是按照 KeyValue 存储的，即你写入的数据可能是一个 rowkey 对应多个列族，多个列，但是实际的存储是每个列都会对应 rowkey 写一遍，即这一条数据有多少个列，就会存储多少遍 rowkey，这会极大影响 HFile 的存储效率；
  2. MemStore 和 BlockCache 都会将缓存部分数据到内存，如果 Rowkey 字段过长内存的有效利用率会降低，系统将无法缓存更多的数据，这会降低检索效率。
  3. 目前操作系统一般都是 64 位系统，内存 8 字节对齐。控制在 16 个字节，8 字节的整数倍， **利用操作系统的最佳特性。**

##### **rowkey 散列设计**

我们已知 HBase 的 Rowkey 是按照字典序排列的，而数据分布在 RegionServer 上的方式是做高位哈希，所以如果我们的 rowkey
首位存在大量重复的值那么很可能会出现数据倾斜问题，关于数据倾斜的问题下面会详细说明，总之，原则上就是 **rowkey 的首位尽量为散列** 。

##### **常访问的数据放到一起**

对于需要批量获取的数据，比如某一天的数据，可以把一整天的数据存储在一起，即把 rowkey 的高位设计为时间戳，这样在读数据的时候就可以指定 start
_rowkey 和 end_ rowkey 做一个 scan 操作，因为高位相同的 rowkey
会存储在一起，所以这样读是一个顺序读的过程，会比较高效。但是这样有一个很明显的问题，违背了上一条“ **rowkey 散列设计**
”原则，很可能会出现数据倾斜问题。所以说没有最好的设计，具体如何权衡就得看实际业务场景了。

#### 2\. 列族设计

![image.png](https://images.gitbook.cn/2020-06-11-070021.png)
列族的设计原则是越少越好，为什么呢？两个方面的考虑。

  * 如图，我们已知 **Region 由一个或者多个 Store 组成，每个 Store 保存一个列族** 。当同一个 Region 内，如果存在大小列族的场景，即一个列族一百万行数据，另一个列族一百行数据，此时总数据量达到了 Region 分裂的阈值，那么不光那一百万行数据会被分布到不同的 Region 上，小列族的一百行数据也会分布到不同 region，问题就来了，扫描小列族都需要去不同的 Region 上读取数据，明显会影响性能。
  * MemStore 刷盘的过程大家也不陌生了，如图，每个 Store 都会有自己对应的那一块 MemStore，但是 **MemStore 的触发是 Region 级别的** ，意思就是大列族对应的 MemStore 刷盘的时候会导致小列族对应的 MemStore 也刷盘，尽管小列族的 MemStore 还没有达到刷盘的阈值。这样又会导致小文件问题。

所以最稳妥的做法就是只设置一个列族，如果需要区分业务可以在列名前加前缀。

### 如何处理数据倾斜问题？

数据倾斜是分布式领域一个比较常见的问题，在大量客户端请求访问数据或者写入数据的时候，只有少数几个或者一个 RegionServer
做出响应，导致该服务器的负载过高，造成读写效率低下，而此时其他的服务器还是处于空闲的状态，就是所谓“ **旱的旱死，涝的涝死**
”。那么为什么会造成这种情况，主要的原因就是数据分布不均匀，可能是数据量分布不均匀，也可能是冷热数据分布不均匀。而 **糟糕的 rowkey
设计就是发生热点即数据倾斜的源头** ，所以这里会详细说说避免数据倾斜的 rowkey 设计方法。

  * **加盐**

加盐即在原本的 rowkey 前面加上随机的一些值。

  * 随机数：加随机数这种方式在各种资料中经常会被提到，就是在 rowkey 的前面增加随机数来打散 rowkey 在 region 的分布，但是我觉得这不是一种好的选择，甚至都不能作为一种选择， **因为 HBase 的设计是只有 rowkey 是索引，rowkey 都变成随机的了，读数据只能做性能极低的全表扫描了** 。总之不推荐。

  * 哈希：哈希的方式明显比随机数更好， **哈希会使同一行永远用一个前缀加盐** 。同样可以起到打散 rowkey 在 region 的分布的目的，使负载分散到整个集群，最重要 **读是可以预测的** 。使用确定的哈希可以让客户端重构完整的 rowkey，可以使用 get 操作准确获取某一个行数据。

  * **反转**

反转即把低位的随机数反转到高位。比如手机号码或者时间戳的反转，高位基本固定是 1 开头的，而末位是随机的。这种同样是一种比较常规的构成散列的方式。

  * ** hbase 预分区**

**预分区上面已经提到过，这种方式对于处理** 数据量分布不均匀，和冷热数据分布不均匀都是有一定效果的，但是需要对业务的应用场景做好准确的预判。

### 参数调优需要重点关注哪些？

HBase 的参数很多，一般都是在使用和优化的过程中不断地调整的，这里只列举出比较重要和常用的几个配置让大家参考一下。

#### 合并与分裂

  * hbase.hregion.majorcompaction **storeFile 的合并时间间隔**
  * hbase.regionserver.regionSplitLimit **最大的 region 数量**
  * hbase.hregion.max.filesize ** 每个 region 的最大限制**

这三个参数，是用于控制 StoreFile 的合并和 Region 的分裂的。在生产环境下，这两个操作一般都会设置为禁止自动合并和禁止自动
split，因为这两步的操作都是比较耗费资源的，自动会让操作时间不可控，如果是在业务繁忙的时间做了这些操作造成的影响是非常大的，所以一般
**配置禁止自动** ，转为自己管理，在系统没那么繁忙的晚上手动出发相关操作。

#### 内存

  * file.block.cache.size ** BlockCache 的内存大小限制**
  * hbase.regionserver.global.memstore.upperLimit **Memstore 占用内存在总内存中的比例**

我们之前提到过，RegionServer 的内存主要分为两块，读缓存 block cache 和写缓存 memstore，第一个配置是 block
cache 的内存大小限制，第二个配置是 memstore 占用内存在总内存中的比例， **所以这两个配置要配合使用才能控制读和写缓存的大小。**
block cache 默认是 0.25，在读多于写的业务中，可以适当调大该值。upperLimit 默认值 0.4，写偏多的场景可以适量调大建议
0.45，RegionSerever 中 block cache 和 memstore cache 的总大小不会超过 0.8，太大可能会造成 oom。

