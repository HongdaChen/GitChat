上一篇内容总结了 Flink 的运行架构和编程模型，这一篇我们要讨论的是 Flink
的设计，包括它是如何解决容错、内存管理、反压问题的，这些知识点是面试的常客，目的是考察大家对 Flink 理解的深度。

**本篇面试内容划重点：容错、内存管理、反压。**

### 容错机制

Flink 的容错机制主要是依靠 barrier + checkpoint 来产生分布式快照，分布式快照中保存了计算过程中 Operator/task
的中间状态信息（state）。这些非常轻量级的快照会频繁地异步生成，且对系统性能不会产生太大的影响。state
状态信息会持久化到磁盘。如果程序失败，Flink 会根据最新的 checkpoint 数据来重置
Operator，保证系统能够接着失败前的状态再正常运行。所以，我们要说清楚 Flink 的容错机制必须先搞懂
CheckPoint、Barriers、state 这几个概念。

#### **Checkpoint**

Checkpoint 是 Flink 实现容错机制的核心，它能够根据配置周期性地基于数据流中各个 Operator/task
的状态来生成快照，从而将这些状态数据定期持久化存储下来，当程序意外崩溃，重新运行程序时可以有选择地从这些快照进行恢复，从而修正因为故障带来的程序数据异常。

**Checkpoint 和 Savepoint 的区别？**

Checkpoint 是为 runtime 准备的，Savepoint 是为用户准备的。Checkpoint 机制的目标在于保证 Flink
作业意外崩溃重启不影响 exactly once 准确性，通常用于系统容错。而 Savepoint 的目的在于在 Flink
作业维护（比如更新作业代码）时将作业状态写到外部系统，以便维护结束后重新提交作业可以到恢复原本的状态。

  * Checkpoint 异常恢复，保证可用性，在任务发生故障时，为任务提供给自动恢复机制；Savepoint 需手动备份、恢复暂停作业的方法。
  * Checkpoint 支持 RocksDBStateBackend 的增量方式对状态信息进行快照，且任务停止状态信息删除；Savepoint 仅支持全量快照，任务停止状态信息不会删除。
  * Checkpoint 由 Flink 自动地周期性地创建和删除，无需用户的交互；Savepoint 由用户手动地管理，包括调度、创建、删除。

#### **State**

Flink 任务运行中的 Operator/Task 是有状态的，上文也提到了分布式快照中保存了计算过程中 Operator/task
的中间状态信息，这个状态指的就是 Flink 中的 State，这些状态数据在容错恢复起到了非常关键的作用。在 Flink 中，按照基本类型，对 State
做了以下两类的划分：

  * **Keyed State** ：和具体的 key 相关联，只能在 KeyedStream 的 function 和 operator 上使用。每个 keyed-state 逻辑上与一个 `<并行操作实例, 键>`（`<parallel-operator-instance, key>`）绑定在一起。
  * **Operator State** （或者 non-keyed state）：它是和 Key 无关的一种状态类型，相当于一个并行度实例，对应一份状态数据。因为这里没有涉及 Key 的概念，所以在并行度（扩/缩容）发生变化的时候，这里会有状态数据的重分布的处理。

#### **Barriers**

![image.png](https://images.gitbook.cn/de67f660-dc49-11ea-b119-779a577aa203)

Barriers（栅栏）是分布式快照的核心。如图，barrier 会被 operator 插入到数据流中和数据一起向下流动，需要说明的是 barrier
非常轻量，不会干扰数据流处理。barrier 会把数据流划分为两个快照（checkpoint），它的出现意味着上一个 checkpoint 的结束和下一个
checkpoint 的开始，所以当下游算子收到 barrier 事件后，将发起一次新的 checkpoint。当最终的 sink operator 接收到
barrier 时，会向 CheckpointCoordinator（JobManager 中负责管理 Checkpoint 的生命周期的组件）
确认快照是否已完成，当所有 sink 都确认了这个快照，快照就被标识为完成。

![image.png](https://images.gitbook.cn/ca728e90-dc49-11ea-931d-3be4c30e79b3)

在 exactly-once 语义下，消费端需要解决延迟数据问题，对齐不同 channel 的 barrier，具体的流程看上图：

  * operator 只要一接收到某个输入流的 barrier n，它就不能继续处理此数据流后续的数据，直到 operator 接收到其余流的 barrier n。否则会将属于 snapshot n 的数据和 snapshot n+1 的搞混。
  * barrier n 所属的数据流先不处理，从这些数据流中接收到的数据被放入接收缓存里（input buffer）。
  * 当从最后一个流中提取到 barrier n 时，operator 会发射出所有等待向后发送的数据，然后发射 snapshot n 所属的 barrier。

经过以上步骤，operator 恢复所有输入流数据的处理，优先处理输入缓存中的数据。

### 说说 Flink 的内存管理是如何做的？

上一篇文章提到的 TaskManager 是用来运行用户代码的 JVM 进程。所以我们来看看 TaskManager 的堆内存结构。

![](https://images.gitbook.cn/9df3def0-dc49-11ea-878e-43848333f7b7)

图片引用自：[tbcdn.cn](http://img3.tbcdn.cn/5476e8b07b923/TB17qs5JpXXXXXhXpXXXXXXXXXX)

  * **Network Buffers** ：默认数量是 2048 个 32KB 大小的 buffer，主要用于数据的网络传输。在 TaskManager 启动的时候就会分配。
  * **Memory Manager Pool** ：这是一个由 MemoryManager 管理的，由众多 MemorySegment 组成的超大集合，MemorySegment 是预分配的内存块，也是 flink 中最小的内存分配单元。Flink 中的 operator（如 sort/shuffle/join）会向这个内存池申请 MemorySegment，将序列化后的数据存于其中，使用完后释放回内存池。默认情况下，池子占了堆内存的 70% 的大小。
  * **Remaining (Free) Heap** ：这部分的内存是留给用户代码以及 TaskManager 的数据结构使用的。因为这些数据结构一般都很小，所以基本上这些内存都是给用户代码使用的。从 GC 的角度来看，可以把这里看成的新生代，也就是说这里主要都是由用户代码生成的短期对象。

**Flink 在内存使用上做的优化？**

  * 常驻型数据都以二进制的形式存放在老年代，可以不被 GC 回收，而由用户代码生成的短生命周期对象会在 Minor GC 发生时被快速回收。只要用户不去创建大量类似缓存的常驻型对象，那么老年代的大小是不会变的，Major GC 也就永远不会发生。减少了 GC 的压力。
  * 所有的运行时数据结构和算法只能通过内存池申请内存，保证了其使用的内存大小是固定的，不会因为运行时数据结构和算法而发生 OOM。内存紧张时，MemorySegment 会落盘，避免 OutOfMemoryErrors。
  * 只存储实际数据的二进制内容，节省了内存空间。

**Flink 也支持了对堆外内存的使用，为什么要用堆外内存？**

  * 启动超大内存（上百 GB）的 JVM 需要很长时间，GC 停留时间也会很长（分钟级）。使用堆外内存的话，可以极大地减小堆内存，使得 TaskManager 扩展到上百 GB 内存不是问题。
  * zero-copy，这个词在 Kafka 那个章节也提到过，就是使用堆外内存在写磁盘或网络传输时不用将数据传输到用户空间，拥有更高效的 IO 操作。
  * 堆外内存是进程间共享的，所以 JVM 进程崩溃也不会丢失数据，这可以用来做故障恢复。

但是，堆外内存也不是万能的，堆外内存的管理比堆内麻烦，在使用、监控、调试等方面都比较复杂，所有有些操作使用堆内存会更加高效。

### Flink 的反压问题

关于反压问题的描述大家都会提到一个版本节点 Flink 1.5，1.5 之前的版本并没有什么特别的反压机制，它利用 buffer
来暂存堆积的无法处理的数据，当 buffer 用满了，则上游的流阻塞，不再发送数据。可见此时的反压是从下游往上游传播的，一直往上传播到 Source
Task 后，Source Task 最终会降低或提升从外部 Source 端读取数据的速率。这种机制有一个比较大的问题，在这样的一个场景下：同一 Task
的不同 SubTask 被安排到同一个 TaskManager，则 SubTask 与其他 TaskManager 的网络连接将被多路复用并共享一个 TCP
信道以减少资源使用，所以某个 SubTask 产生了反压的话会把多路复用的 TCP 通道占住，从而会把其他复用同一 **TCP 信道** 的且没有流量压力的
SubTask 阻塞。

Flink 1.5 版本之后的基于 Credit 反压机制解决了上述问题。这种机制主要是每次上游 SubTask 给下游 SubTask 发送数据时，会把
Buffer 中的数据和上游 ResultSubPartition 堆积的数据量 Backlog size
发给下游，下游会接收上游发来的数据，并向上游反馈目前下游现在的 Credit 值，Credit 值表示目前下游可以接收上游的 Buffer 量，1 个
Buffer 等价于 1 个 Credit。可见，这种策略上游向下游发送数据是按需发送的，而不是和之前一样会在公用的 Netty 和 TCP
这一层数据堆积，避免了影响其他 SubTask 通信的问题。

![](https://images.gitbook.cn/7968b510-dc49-11ea-9e55-350579dffb16)

举个例子，如图所示：

  1. 上游 SubTask A 向下游的 SubTask B 发送数据并告诉 SubTask B 还积压了 5 个 buffer （Backlog size=5）的数据。
  2. SubTask B 接受到数据后，得知上游还有 5 个 buffer 需要发过来，于是向 Buffer Pool 申请 Buffer，但是容量不足，仅申请到 2 个 Buffer 空间。
  3. 此时 SubTask B 会向上游反馈 Credit=2，代表下游最多只能接收 2 个 buffer 了。
  4. SubTask A 收到反馈消息后得知空间不足，所以最多只给会下游发送 2 个 Buffer 的数据。

这样每次上游发送的数据都是下游 Buffer 可以承受的数据量，而不会在 TCP 信道阻塞，影响其他 SubTask 通信。

