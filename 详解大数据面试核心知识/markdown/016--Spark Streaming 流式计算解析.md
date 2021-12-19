流式计算在大数据生态中也是非常地有份量，Spark Streaming 在过去很长一段时间都是最常用的流式计算引擎，直到出现了
Flink。所以常常有人问我，有了 Flink 之后 Spark Streaming 还需要学吗，答案是肯定的，Spark Streaming
目前在大部分公司的应用还是非常广泛的，虽然有被 Flink
替代的趋势，但是技术的迭代是需要消耗成本的，并不是每个公司都愿意去承担这个成本，而且大部分公司的业务需求也并没有那么复杂，所以很多公司还是抱着能用就行的心态。

另外，Spark Streaming 依附着强大的 Spark 生态，Spark 能做的事情太多了，Flink 除了流式计算，在其他方面目前都还是不如
Spark 的，所以直接使用 Spark 全家桶来做计算目前也还是一个比较常规的选择。

**本篇面试内容划重点：架构、容错、反压、窗口函数。**

### SparkStreaming 的底层执行流程

![image.png](https://images.gitbook.cn/b59facf0-d4d3-11ea-a0a4-91ded31f57b1)

#### **Driver 端**

  * **StreamingContext** 是程序的入口，在 Driver 端执行，负责初始化和控制图中的 SparkStreaming 程序的各个功能模块。
  * **JobScheduler** 负责总体的动态作业调度，并维护了一个线程池来负责提交 JobSet。它会调用下面的 JobGenerator 和 ReceiverTracker 来完成构建 JobSet 的目的。
  * **ReceiverTracker** 负责启动，管理各个 Executor 端的 Receiver（用于接收数据） 及维护 Receiver 汇报的 BlockId。
  * **JobGenerator** 负责构建 JobSet(time, jobs, receivedBlockInfo)，过程涉及到与 DStreamGraph 和 ReceiverTracker 的交互，最终通过 JobScheduler 提交 JobSet。
    * **ReceiverTracker** 会接收并维护 Receiver 汇报的 BlockId 信息。（JobSet 的 receivedBlockInfo）
    * **DStreamGraph** 负责生成 RDD DAG 的实例，由 DStream 和 转换计算的算子组成。（JobSet 的 jobs）

#### **Executor 端**

Executor 端 主要由 ReceiverSupervisor 来负责与 ReceiverTracker 通信。

  * ReceiverSupervisor 负责启动 Receiver 来接受数据，Receiver 将不断接收到的数据交给 ReceiverSupervisor 进行数据转储。
  * ReceiverSupervisor 通过 BlockGenerator 将接收到的数据在内存生成块（Block）。
  * ReceiverSupervisor 向 ReceiverTracker 上报 Executor 内存中 BlockID 和日志文件中数据块的偏移信息。
  * 在每个批处理时间间隔中都有从 Driver 提交来的 Job 的业务逻辑在 Executor 中运行。

### 容错机制

![image.png](https://images.gitbook.cn/16d8be80-d4d4-11ea-9203-1bcff1a80007)

基于上面运行流程的图，这里增加了容错机制的内容，如图红色方块所示，SparkStreaming 的容错分为两块，Executor 端和 Driver
端，具体见下述分析。

#### **Executor 端**

  * **replica 热备** ：如果配置了 StorageLevel 为 MEMORY _ONLY_ 2 ，那么在 receiver 接收到数据之后，除了在本 Executor-1 内存存储 Block 数据之外，同时会把数据 replicate 到 executor-2 上去。这样在一个 replica 失效后（Executor 宕机），可以立刻无感知切换到另一份 replica 进行计算。
  * **WriteAheadLog（WAL）冷备** ：在 receiver 接收到数据之后，除了在本 Executor 内存存储 Block 数据之外，还会把块数据作为 log 写出到 WriteAheadLog 里作为冷备，WAL 保存在 HDFS。当 executor 宕机时，就可以由其他的 executor 获取 WAL 来恢复数据。WAL 存在 HDFS，可以保证数据的可靠性，但是可能会带来性能的问题，无论是写入数据时还是恢复数据时都需要与 HDFS 交互，会降低实效性。

#### **Driver 端**

  * **WriteAheadLog（恢复任务数据）** ：Executor 接收到的数据块的元信息会发送给 Driver, 这些元数据包括：executor 内存中数据块的 blockID 和日志文件中数据块的偏移信息。Driver 端可以配置将这些元数据保存到 WAL 中，避免 Driver 端的异常情况导致数据丢失。
  * **checkpoint（恢复执行进度）** ：将定义流式计算的相关元数据信息持久化到 HDFS，主要对 DStreamGraph 和 JobScheduler 做 Checkpoint，记录整个 DStreamGraph 的变化、和每个 batch 的 job 的完成情况。

### 反压机制

![image.png](https://images.gitbook.cn/4c6fc4d0-d4d4-11ea-8873-4fd2c96cab0c)

反压即 BackPress，通俗地说就是流量控制。仍然是基于上面的流程图，我们来看看反压机制的流程（上图黑色方块），同样是分为 Driver 端和
Executor 端两部分共同完成。

#### **Driver 端速率估算**

**RateController** 组件是 JobScheduler 的监听器，主要监听集群所有作业的提交、运行、完成情况，并从 BatchInfo
实例中获取以下信息，交给 **速率估算器（RateEstimator）** 做速率的估算。

  * 当前批次任务处理完成的时间戳 （processingEndTime）
  * 该批次从第一个 job 到最后一个 job 的实际处理时长 （processingDelay）
  * 该批次的调度时延，即从被提交到 JobScheduler 到第一个 job 开始处理的时长（schedulingDelay）
  * 该批次输入数据的总条数（numRecords）

#### **速率估算器（RateEstimator）**

Spark 2.x 只支持基于 PID 的速率估算器，这里只讨论这种实现。基于 PID
的速率估算器简单地说就是它把收集到的数据（当前批次速率）和一个设定值（上一批次速率）进行比较，然后用它们之间的差计算新的输入值，估算出一个合适的用于下一批次的流量阈值。这里估算出来的值就是流量的阈值，用于更新每秒能够处理的最大记录数。

估算好的速率值会通过 JobScheduler 的 receiverTracker 来发送给各个 Executor 端的
ReceiverTracker，最终作用在 BlockGenerator 上。

#### **Executor 端流量控制**

最后，该阈值通过 RPC 的方式发布到 Executor 端的 BlockGenerator，BlockGenerator 是用于 Receiver
接收数据后将数据存入缓存的对象，流量控制的具体实现就是在这里完成的，代码中直接用了 Google Guava 工具包中的限流器
RateLimiter（令牌桶算法）来实现，参数就是 RateEstimator 估算出来的值。

### 窗口函数

如图，窗口函数会把原始 DStream 的若干批次的数据合并成为一个新的 Windowed DStream。合并的过程有有两个很重要的概念，窗口长度
windowLength 和 窗口移动速率 slideInterval。

**WindowLength**

**窗口长度** ，即每次生成新 DStream 需合并的原始 DStream 个数，图中 windowLength 的值为
3，time3、time4、time5 三个时间点生成的 DStream 会合并为一个新的 Windowed DStream。

**SlideInterval**

**窗口移动速率，** 即合并的原始 DStream 的时间间隔，图中 slideInterval 的值为 2，即每隔 2 个时间点就生成一次新的
Windowed DStream。

![](https://images.gitbook.cn/7dfb89d0-d4d9-11ea-baf7-95cdfa1a7574)

**相关函数**

**window(windowLength, slideInterval)**

根据窗口长度和窗口移动速率合并原始 DStream 生成新 DStream。

**举例** ：每 2 秒生成一个窗口长度为 5 秒的 Dstream

    
    
    val windowedDstream = dstream.Window(Seconds( 5 ), Seconds( 2))
    

**countByWindow(windowLength, slideInterval)**

返回指定长度窗口中的元素个数。

**举例** ：每 2 秒统计一次近 5 秒长度时间窗口的 DStream 中元素的个数：

    
    
    val windowedDstream = dstream.countByWindow(Seconds( 5 ), Seconds( 2))
    

**reduceByWindow(func, windowLength, slideInterval)**

对设定窗口的 DStream 做 reduce 操作，类似 RDD 的 reduce 操作，只是增加了时间窗口维度。

**举例** ：每 2 秒合并一次近 5 秒长度时间窗口的 DStream 中元素用“-”分隔：

    
    
    val windowedDstream = dstream.reduceByWindow(_ + "-" + _, Seconds( 5 ), Seconds( 2))
    

**reduceByKeyAndWindow(func, windowLength, slideInterval, [numTasks])**

根据 Key 和 Window 来做 Reduce 聚合操作，在上述 reduceByWindow 的基础上增加了 Key 维度，func 是相同 Key
的 value 值的聚合操作函数。数据源的 DStream 中的元素格式必须为 (k, v) 形式，windowLength 和 slideInterval
同样是用于确定一个窗口 Dstream 作为数据源。numTasks 是一个可选的并发数参数。

**举例** ：每 2 秒根据 Key 聚合一次窗口长度为 5 的 DStream 中元素，下例中聚合的方式为 value 相加。

    
    
    val windowedDstream = pairsDstream.reduceByKeyAndWindow((a:Int , b:Int) => (a + b) , Seconds(5) , Seconds( 2 ))
    

**reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval,
[numTasks])**

这个方法比上一个多传入一个函数 invFunc。func 是 value 值的聚合操作函数，在数据流入的时候执行这个操作。invFunc
是在数据流出窗口的范围后执行的操作。

**举例** ：每 2 秒根据 Key 聚合一次窗口长度为 5 的 DStream 中元素，聚合的方式为 value 相加。 invFunc：假设
invFunc 的参数如下例为 a 和 b，那么 a 是上个 window 经过 func 操作后的结果，b 为此次 window 与上次 window
在时间上交叉的元素经过 func 操作后结果。

    
    
    val windowedDstream = pairsDstream.reduceByKeyAndWindow((a: Int, b:Int ) => (a + b) , (a:Int, b: Int) => (a - b) , Seconds(5) , Seconds( 2 ))
    

**countByValueAndWindow(windowLength, slideInterval, [numTasks])**

统计时间窗口中元素值相同的元素个数，类似于 RDD 的 countByValue 操作，在这个基础上增加了时间窗口维度。同样，数据源的 DStream
中的元素格式必须为 (k, v) 形式，返回的 DStream 格式为 (K, Long)。

**举例** ：每 2 秒根据 Key 聚合一次窗口长度为 5 的 DStream 中元素，下例中聚合的方式为 value 相加。

    
    
    val windowedDstream = pairsDstream.countByValueAndWindow(Seconds( 5 ), Seconds( 2))
    

### Structured Streaming

Structured Streaming 就像是 Spark SQL 和 Spark Streaming
结合的产物，它最关键的思想是将实时数据流视为一个不断增加的表，从而就可以像操作批静态数据一样来操作流数据了。Structured Streaming
的设计理念和 Flink 很类似，他们的实现都参考了 Google 的 _The Dataflow Model_ 这篇论文，但是 Flink
做得更加出色一些，这也是 Structured Streaming 一直都不温不火的原因。既然很多概念上的东西和 Flink 是很相似的，这里就不详细说
Structured Streaming 的内容了，总结一些基本概念，有兴趣的可以去官网学习一下，后面两篇再花多一些章节去总结 Flink 相关的面试题。

![](https://images.gitbook.cn/19534220-d4d9-11ea-a0a4-91ded31f57b1)

支持了基于事件时间（event time）的窗口操作：

![](https://images.gitbook.cn/081495e0-d4d9-11ea-a813-c73f72be9c7a)

通过结合水位线（watermark）来处理延迟数据：

![](https://images.gitbook.cn/f91dddd0-d4d8-11ea-a813-c73f72be9c7a)

关于 event time、watermark、window、time、state 等相关概念，在后面 Flink 的章节会详细描述，这里不再赘述。

