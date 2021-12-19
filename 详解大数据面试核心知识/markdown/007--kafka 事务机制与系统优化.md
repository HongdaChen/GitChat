上一篇内容我们了解到 Kafka 通过 ISR 机制平衡了各个 Partition 之间的可用性和一致性，通过顺序读写/PageCache/零拷贝保证了
Kafka 的高性能。这里我们继续深入之前没有完成的话题，Kafka 还有一块非常重要的内容 ——
消息投递语义，这块内容和流式计算息息相关，解决的消费数据的唯一性的问题。

**本篇面试内容划重点：消息投递语义、幂等、rebalance、调优。**

### Kafka 的消息投递语义

kafka 支持 3 种消息投递语义，最多一次，最少一次，恰好一次。具体看下述说明。

**At most once：最多一次** 这种语义可能 **存在消息丢失的情况，但不会重复消费** 。Producer
端发送消息时配置异步发送即是这一种语义，可能存在数据丢失的情况，好处是高效，不需要等待 Broker 响应成功消息。 大家都知道，Consumer
在消费数据时需要管理 Offset，那么如果 consumer 在读取消息后，先修改
offset，然后处理消息，这就是最多一次的语义，但是生产环境中一般不会用这种方式。

**At least once：最少一次** 这种语义消息 **不会丢失，但是可能会重复** 。Producer 端同步发送消息时，需要等待 Broker
确认，如果由于网络问题，确认消息没有发送成功或者延迟了，Producer 会重试，所以可能会存在重复的情况但是不会丢失。 Consumer
消费数据后，先处理数据，然后修改 offset，这种方式保证了 At least once 语义。因为如果 consumer 已经处理完数据，但是在保存
Offset 之前宕机了，则下一次消费会从上一次的 offset 开始，则宕机前处理的数据会重复消费到。

**Exactly once：有且只有一次** **消息不丢失不重复，有且仅消费一次（0.11 中实现，仅限于下游也是 Kafka）** ，需要
Producer 和 Consumer 两端同时保证。exactly once 需要依赖 Kafka 的事务机制。下面就来看看 Kafka 的事务机制。

### Kafka 的事务保证

#### 幂等性发送

上文说到实现 Exactly Once 是需要 Producer 和 Consumer 两端同时保证的，在 0.11 版本中，下游也是 Kafka
是可以实现的，如果下游也是 kafka 那么此时 Consumer 就是 Producer ，这个时候的重点就在于** Producer 的幂等**实现了。

Kafka 引入了 **Producer ID 和 Sequence Number 来实现 Producer 的幂等** 。每个新的 Producer
在初始化的时候都会被分配一个对用户透明的唯一 PID，该 Producer 发送数据的每个 ****都对应一个从 0 开始单调递增的 Sequence
Number。

Broker 端也会为每个维护一个 ID，在接收到一条数据后，对比该数据的序号和当前维护的序号

  * 如果消息 ID 比 Broker ID 恰好大 1 则 Commit 这条数据并且 Broker ID 加 1。
  * 如果消息 ID 与 Broker ID 差值大于 1 则说明中间有数据丢失，或者出现乱序，Producer 抛 InvalidSequenceNumbe。
  * 如果消息 ID 与 Broker ID 查值小于 1 则说明有重复数据，直接抛弃数据，Producer 抛 DuplicateSequenceNumber。

**如果没有开启 Producer 的幂等发送，可能会发生什么问题？**

  * Broker Commit 数据后，发送 ACK 前宕机，或者 ACK 信息因网络延迟超过了超时阈值，Producer 认为消息未发送成功，然后重试造成数据重复。
  * Producer 发两条信息，第一条消息发送失败，或者网络延迟，第二条消息发送成功，然后第一条在第二条之后到打达 Broker 端，造成单个 Partition 内部的数据乱序。

#### 事务原子性

上述幂等设计只能保证单个 Producer 对于单个 Topic 分区的 Exactly Once 语义。 **无法保证多个读写操作的原子性**
，即要么都成功要么都失败。所以 Kafka 还需要有一个事务的保证，使得应用程序将生产数据和消费数据当作一个原子单元来处理，保证不同 topic
跨分区的数据要么成功要么失败，且失败后还能做事务恢复。

为了实现这种效果，用户必须在应用程序里提供一个稳定的（重启后不变） **唯一的 Transaction ID** 。Transaction ID 与 PID
一一对应。同时 Producer 还会维护一个 epoch，用来保证当前 Producer 是最新的进程。事务流程如下：

  * 先标记开启事务，写入数据
  * 全部成功则在 Transaction Log 中记录为 Prepare Commit 状态，否则记录 Prepare Abort 状态。
  * 然后给每个相关的 Partition 写入一条 commit 或 abort 消息，标记这个事务的 message 可以被读取或已经废弃。
  * 成功后在 Transaction log 记录下 commit/abort 状态，事务结束。

### Consumer rebalance 算法

Consumer rebalance **目的是提升 Topic 的并发消费能力** ，它会发生在 ConsumerGroup 内部的** Consumer
数量发生变化的时候。**

  * 假设 TopicA ，具有如下 Partitions（根据 id 排好序）: P0,P1,P2,P3
  * ConsumerGroup 中，有如下 Consumer（根据 id 排好序）: C0,C1
  * 计算倍数: M = [P0,P1,P2,P3].size / [C0,C1].size，本例值 M=2(向上取整)
  * 然后依次分配 partitions: C0 = [P0,P1],C1=[P2,P3],即 Ci = [P(i * M),P((i + 1) * M -1)]

可见，每个 **Consumer 只负责调整自己所消费的 partition** ，为了保证整个 consumer group 的一致性，所以当一个
consumer 触发了 rebalance 时，该 consumer group 内的其它所有 consumer 也应该同时触发 rebalance。

另外，看看下面这两个是关于 rebalance 的参数，由此可见，rebalance 超时失败的时间为：rebalance.backoff.ms *
rebalance.max.retries

  * rebalance.backoff.ms:2000：消费均衡 **两次重试之间的时间间隔**
  * rebalance.max.retries:10：消费均衡的 **重试次数**

### 参数调优需要关注的配置

参数调优是一个需要耐心的活，就像老中医一样，需要慢慢地观察现象，一步一步调整，是不可能一蹴而就的。这里，我把调优这块内容分成了
Broker，Producer，Consumer，Zookeeper，JVM 这几部分，每个部分各司其职，解决不同的问题。

#### Broker

  * **num.network.threads**

broker 处理消息的最大线程数（默认为 3），建议 cpu _core_ num+1

  * **num.io.threads**

broker 处理磁盘 IO 的线程数，建议 cpu _core_ num*2

  * **log.flush.interval.messages:10000**

刷数据到磁盘的数据量阈值

  * **log.flush.interval.ms:1000**

刷数据到磁盘的时间间隔，（ms）

  * **log.retention.hours:72**

Kafka 数据保留的时间阈值

  * **log.segment.bytes:1073741824**

segment 文件大小，一般配 1G，太小不利于回收磁盘空间，重启 kafka 加载也会慢

  * **default.replication.factor**

创建的 Topic 的每个分区的 Replica 数量，Replica 过少会影响数据的可用性，太多会浪费资源，默认 3 即可。

#### Producer

  * **buffer.memory**

在 Producer 端用来存放尚未发送出去的 Message 的缓冲区大小。

  * **block.on.buffer.full**

缓冲区满了之后可以选择阻塞发送或抛出异常。

  * **compression.type**

发送的数据的压缩方式，默认无。

  * **batch.size**

Producer 会积累一批数据一次发送，这个参数指定一批数据总大小的上限。

  * **acks**

这个配置我们在上一篇文章中提到过了，0 即无需响应，1 需 Leader 响应，all 需所有 ISR 中的 Replica 响应。

#### Consumer

  * **num.consumer.fetchers:1**

启动 Consumer 的个数，适当增加可以提高并发度。

  * **fetch.min.bytes:1**

Fetch Request 返回的数据量阈值。

  * **fetch.wait.max.ms:100**

Fetch Request 的等待超时时间

#### zookeeper

  * **zookeeper.session.timeout.ms:5000**

zookeeper 会话超时时间

  * **zookeeper.sync.time.ms:2000**

ZooKeeper 集群中 leader 和 follower 之间的同步的时间

  * **auto.commit.enable:false**

自动向 ZooKeeper 提交 offset，不建议启动

  * **auto.commit.interval.ms:1000**

自动提交 offset 到 zookeeper 的时间间隔。

  * **zookeeper.connection.timeout.ms:10000**

zookeeper 连接的超时时间

#### JVM 垃圾回收

Java 的垃圾回收一直是个很大的问题，Kafka 的垃圾回收机制可以调整成 G1。在应用程序的整个生命周期，G1 会
**自动根据工作负载情况进行自我调节** ，而且它的 **停顿时间是恒定的** 。它可以轻松地 **处理大块的堆内存**
，把堆内存分为若干小块的区域，每次停顿时并不会对整个堆空间进行回收。下面是 G1 的两个关键参数和说明。 **

  * **MaxGCPauseMillis **

垃圾回收最长停顿时间的目标值。该值不是固定的，也就是说 JVM 会尽一切能力满足这个时间要求，但是不能保证一定在这个时间内。G1
可以根据需要来指定更长的时间。它的默认值是 200ms。如果这个值过小，会导致 youngGC 的频率大大增高。

  * **InitiatingHeapOccupancyPercent **

G1 启动新一轮垃圾回收之前可以使用的堆内存百分比，默认值是 45。即堆内存的使用率达到 45% 之前，G1
不会启动垃圾回收。这个百分比包括新生代和老年代的内存。Kafka 对堆内存的使用率非常高，容易产生垃圾对象，所以可以把这些值设得小一些。

