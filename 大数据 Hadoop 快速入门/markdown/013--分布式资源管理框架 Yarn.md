### 诞生背景

在 Hadoop 1.x 中，是没有 Yarn 这个分布式资源管理框架的，它在 Hadoop 2.x 中首次推出。它诞生的原因其实很简单，就是 Hadoop
1.x 中的架构存在一些问题。

Hadoop 1.x 中包括 HDFS 和 MapReduce。其中 MapReduce 身兼两职，它既是计算框架，又是资源管理框架。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201015042106.png)

它的架构是主从架构，其中 Job Tracker 为主节点，Task Tracker 为从节点。

但 Job Tracker **既做资源管理，又做任务调度，负载太大** 。Client（客户端）向 Job Tracker 提交 Job（作业），Job
Tracker 首先为提交的 Job 在从节点 Task Tracker 上分配资源，之后便将 Job 拆分为 Task 调度到 Task Tracker
中运行，而这些 Task 运行过程中的状态会实时向 Job Tracker 汇报，由 Job Tracker 来进行作业管理。

现在看来，作业的提交和运行流程也没有什么问题；但是在大型集群中，大量的 Job 被提交，会生成成百上千个 Task，这些 Task
的资源分配和作业管理全都交由 Job Tracker 来进行，负载极大，会造成性能瓶颈。

而且在 Hadoop 1.x 中，没有实现集群高可用，所以 Job Tracker 存在单点故障，在这种架构下更容易出现问题。

其次，它的 **资源描述模型过于简单，资源利用率较低** 。Job Tracker 在分配资源时，仅把 Task 数量看作资源，没有考虑 CPU
和内存，资源分配不合理；而且强制把资源分成 Map Task Slot 和 Reduce Task Slot，这样的话只能兼容 MapReduce
任务，而无法兼顾其它分布式计算任务。

正因为这些问题，导致它的扩展性较差，集群规模上限 4K；而且源码难以理解，升级维护困难。

### 设计目标

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201015044421.png)

由于 Hadoop 1.x 带来的问题，Yarn 在 Hadoop 2.x 中应运而生。

YARN（Yet Another Resource
Negotiator，另一种资源管理器），是一个分布式通用资源管理系统。注意这里的定位——资源管理框架，意味着它摈弃了 Job Tracker
的任务调度功能，只 **专注于资源管理** ；而 Job Tracker 的主要负载也正是来源于任务调度，这样一来 Yarn
就减少了运行负载，降低了单点故障率。

其次，Yarn 将资源管理独立出来，位置介于通用计算和数据存储 HDFS 之间，这样它便可以作为一个桥梁，除了原生支持的
MapReduce，也兼容其他分布式计算引擎任务。将分布式文件系统 HDFS 上的数据开放了出去，提升了 Hadoop 生态的 **开放性和通用性**
，使得企业在大数据选型时更加灵活。

而且，因为 Hadoop 2.x 中增加了 **高可用** 功能，使得 Hadoop 真正走向成熟。这一系列问题的解决，降低了 Yarn
的单点故障率，也进一步提升了其 **扩展性** 能。

