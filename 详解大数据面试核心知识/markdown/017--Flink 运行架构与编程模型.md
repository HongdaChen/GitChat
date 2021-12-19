七年前如果提起实时流式计算，老程序猿们想到的应该是 Storm，四年前再提到，大家脱口而出的会是 Spark
Streaming，现在再说到实时计算那无疑都会指向 Flink 了。可见开源世界技术的迭代是飞速的，稍不留神就落伍了。

言归正传，上一篇我们讲了 SparkStreaming 和简单地介绍了衍生的 StructedStreaming，也提到了
StructedStreaming 在很多概念和设计理念上都是和 Flink 相似的， _The Dataflow Model_ 是一篇非常经典的
Paper，建议学习流式计算的小伙伴都可以好好地读这篇论文。

**本篇面试内容划重点：各种角色和概念，Time、Window、WaterMark。**

### Flink 运行架构详解

![image.png](https://images.gitbook.cn/776afc30-d4e9-11ea-aefa-4fa1d18dcf14)

**Flink 集群运行的各个组件：**

  * 在 **Client** 端，Flink 会根据开发者写的程序代码构建作业图（JobGraph），逻辑数据流图（logical dataflow graph），然后将任务提交给 Dispatcher 进行下一步执行。
  * **Dispatcher** ：Dispatcher 服务提供了 REST 接口来接收 client 的任务，然后传递给 JobManager。另外，Dispatcher 还提供了一个 WEB UI 界面，用于监控作业的执行情况。
  * **JobManagers** ：主要负责协调和管理程序的执行，具体包括安排任务、管理 checkpoint、故障恢复等。它接收的来自 Dispatcher 的执行程序主要包含了作业图、逻辑数据流图、所有的 classes 文件、第三方类库等等，在 JobManagers， JobGraph 会转换为执行图（ExecutionGraph），然后向 ResourceManager 申请资源来执行该任务，一旦申请到资源，就将执行图分发给对应的 TaskManagers。
  * **ResourceManager/NodeManager** ：ResourceManager 和 NodeManager 都是 Yarn 里面的概念，ResourceManager 负责管理 slots 并协调集群资源。ResourceManager 接收来自 JobManager 的资源请求，并将存在空闲 slots 的 TaskManagers 分配给 JobManager 执行任务。NodeManager 负责管理具体各个节点的资源调度和状态监控。
  * **TaskManagers** ：TaskManagers 负责实际的子任务 (subtasks) 的执行，每个 TaskManagers 都拥有一定数量的 slots。TaskManagers 启动后，会将其所拥有的 slots 注册到 ResourceManager 上，由 ResourceManager 进行统一管理。

**Flink 执行层面上重要概念：**

  * **Operators** ：对数据集执行的 Map、Reduce 等操作，类似 Spark 里算子的概念。
  * **subtask** ：operators 会被划分为 operator subtasks，每个 SubTask 线程运行在一个独立的 Slot，允许多个 subtasks 共享 slots。subtask 是 slot 具体执行时的最小单元。
  * **Slot** 是一组固定大小的资源的合集（计算能力、存储空间）。Slot 在 Flink 里面可以认为是资源组，Flink 将每个任务分成子任务（subtask）并且将这些子任务分配到 Slot 中，从而支持并行执行程序。Slot 的数量通常与每个 TaskManager 的可用 CPU 内核数成比例。
  * **Task** 是一个抽象概念，为了更高效地分布式执行，Flink 会尽可能地将 operator 的 subtask 链接（chain）在一起形成 task。每个 task 在一个线程中执行。将 operators 链接成 task 是非常有效的优化：它能减少线程之间的切换，减少消息的序列化/反序列化，减少数据在缓冲区的交换，减少了延迟的同时提高整体的吞吐量。

**Flink 的 Slot 和 parallelism 有什么区别？**

下面两张是官网上非常经典的图：

slot 是指 TaskManager 的并发执行能力，如果代码运行前我们将 slot 的个数配置为
3（taskmanager.numberOfTaskSlots） ，那么每个 TaskManager 会分配 3 个 Slot 来执行 task，如果配置了
3 个 taskmanager 那么就如图一共有 9 个 Slot。

![](https://images.gitbook.cn/646d3ad0-d4e9-11ea-8a86-ed86f9ad27de)

parallelism 是指 TaskManager 在实际运行过程中的并发。默认并行度的配置为 1（parallelism.default），那么如图 9
个 Slot 只有一个是在工作的，其他 8 个都空闲。在用户开发的过程中可以通过 setParallelism 方法给每个 Operators
算子配置并行度。

![](https://images.gitbook.cn/4be9bf60-d4e9-11ea-9a28-578527398d60)

### 时间语义

SparkStreaming 只支持了 Processing Time，而 Flink 支持如下三种时间语义。

![image.png](https://images.gitbook.cn/3e8eb5f0-d4e9-11ea-a48e-2d1c419ba8b6)

**Processing Time**

数据被处理时服务器的当前系统时间，这种时间语义比较常用，一般用于对时序性和准确性要求不太高的场景。

  * 最简单的 Time 概念，对于程序来说拥有最好的性能和最低的延迟。
  * 分布式和异步环境下，不能保证结果数据的准确性，存在时序问题。
  * 数据延迟对 Flink 的输出结果影响比较大。

**Event Time**

事件发生的时间，是一条数据本身携带的时间字段。有时序要求，比如必须现有下单，再有支付等有先后关系的业务场景。

  * 这种时间来自于数据本身，在事件到达 Flink 之前就已经确定。
  * 必须指定如何生成 WaterMarks，用来表示 Event Time 进度的机制。
  * 无论事件什么时候到达或者其怎么排序，最后处理 Event Time 将产生完全一致和确定的结果，可以解决时序问题。

**Ingestion Time**

事件进入数据源（Flink Source）的时间。介于 Event Time 和 Processing Time 之间，与 Processing Time
相比会自动生成并使用稳定的时间戳，虽然有一定成本，结果更可预测，与 Event Time 相比无法处理无序事件或延迟数据，但是 Ingestion Time
不必指定如何生成水印，具有自动分配时间戳和自动生成水印功能。

### 窗口 window

时间窗口的概念在 SparkStreaming 中也有提到过，相比之下 Flink 提供了更加丰富的 Window 支持。

  * **Time Window** ：根据时间来聚合流数据，一分钟的时间窗口就只会收集一分钟的元素，滑动窗口和前面提到的 SparkStreaming 类似，可以定义一个每 5s 滑动一次长度为 10s 的时间窗口，用于每隔 5s 去统计过去 10s 窗口内的数据。
  * **Count Window** ：统计窗口可以接收一个数值参数，用于表示达到多少事件数后会触发计算。比如计数窗口的值设置的为 4，那么将会在窗口中收集 4 个事件，并在添加第 4 个元素时计算窗口中所有事件的值。
  * **Session Window** ：会话窗口也是接收一个时间参数，表示维持的会话持续时长，如果超过这个时间没有事件到达，就代表着超出会话时长，则会触发计算。

### 水位 watermark 处理延迟数据

Watermark 通常结合 EventTime 来处理乱序和延迟问题，EventTime 前面说多了，是事件本身的时间，所以数据延迟会造成
EventTime 的延迟，这种情况下，我们不可能耗费资源一直等待延迟的数据，于是 Flink 通过 watermark 来保证一个特定的时间后，必须触发
window 计算。我们可以把 Watermark 看作是数据允许延迟的最长时间。

**标点水位线（Punctuated Watermark）**

标点水位线通过数据流中 **某些特殊标记事件** 来触发新水位线的生成。这种方式下窗口的触发与时间无关，而是决定于何时收到 **标记事件** 。
在实际的生产中 Punctuated 方式在 TPS 很高的场景下会产生大量的 Watermark
在一定程度上对下游算子造成压力，所以只有在实时性要求非常高的场景才会选择 Punctuated 的方式进行 Watermark 的生成。

**定期水位线（Periodic Watermark）**

周期性的（允许一定时间间隔或者达到一定的记录条数）产生一个
Watermark。水位线提升的时间间隔是由用户设置的，在两次水位线提升时隔内会有一部分消息流入，用户可以根据这部分数据来计算出新的水位线。

在实际的生产中 Periodic 的方式必须结合时间和积累条数两个维度继续周期性产生 Watermark，否则在极端情况下会有很大的延时。

### 延迟数据的处理

虽说水位线表明着早于它的事件不应该再出现，但是接收到水位线以前的的消息是不可避免的。实际上迟到事件是乱序事件的特例，和一般乱序事件不同的是它们的乱序程度超出了水位线的预计，导致窗口在它们到达之前已经关闭。

迟到事件出现时窗口已经关闭并产出了计算结果，因此处理的方法有 3 种：

  * Allowed Lateness：重新激活已经关闭的窗口并重新触发计算来修正结果，代价比较大，可以设置一个允许的最大迟到时长。
  * Side Output：将迟到事件收集起来单独放入一个数据流分支，以便用户获取并对其进行其他处理。
  * 将迟到事件视为错误消息并丢弃（默认）。

