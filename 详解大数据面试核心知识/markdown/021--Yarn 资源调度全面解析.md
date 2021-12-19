远在 Hadoop v1 版本时期，Yarn 还没有出现，资源管理是和 Hadoop 耦合在一起的，我们没法在 Hadoop 集群运行除了
MapReduce 以外其他计算框架的任务，在策略和管理上也没那么成熟。Yarn 的出现可是说是为 Hadoop
生态圈大数据技术的百花齐放奠定了一个基础，一直到目前为止 Yarn 仍然是在大数据领域最常用的一个资源调度框架，Spark/Flink/Hive
等常用的计算框架都可以依赖 Yarn 来做资源调度，它的重要性不言而喻。

**本篇面试内容划重点：调度流程、调度器的类别、调度策略。**

### 关键概念和角色和资源调度流程

![image.png](https://images.gitbook.cn/c5b545c0-e2b9-11ea-921d-1f9dabf08aa3)

**ResourceManager**

ResourceManager 内部主要有两个组件：Scheduler 和 ApplicationManager。

  * Scheduler 是一个资源调度器，它主要负责 **协调集群中各个应用的资源分配** ，且只负责调度容器 Containers，不会关心应用程序监控及其运行状态等信息，也不会插手任务失败的问题。调度的方式后面会有详细说明。
  * ApplicationManager 负责系统中所有应用程序的管理工作。接收 Client 端的应用提交请求，为应用分配第一个 Container 来运行 ApplicationMaster，并监控 ApplicationMaster，在遇到失败时重启 ApplicationMaster 运行的 Container。

**NodeManager**

NodeManager 是 ResourceManager 的小弟，执行 ResourceManager 发布的任务并上报信息。NodeManager
进程运行在集群中的各个服务器节点上，负责管理所在节点上的资源分配和监控运行节点的健康状态，主要包括两个方面的内容。

  * 接收 ResourceManager 的资源分配请求（比如需要使用 xx CPU，xx 内存），NodeManager 负责分配 Container 。
  * 负责监控并报告 Container 状态信息、可用资源信息、健康状态等给 ResourceManager。

NodeManager 只负责管理自身的 Container，它并不知道运行在它上面应用的信息。 负责管理应用信息的组件是
ApplicationMaster，在后面会讲到。

**Container**

即容器，是 Yarn 的计算单元，具体执行应用的基本单位，一个 Container 就是一组分配的系统资源，目前只包含两种系统资源，CPU
和内存资源（之后可能会增加磁盘、网络等资源）。

Container 和集群节点的关系如上图：一个节点会运行多个 Container，但一个 Container 不会跨节点。

值得说明的是，ResourceManager 只负责告诉 ApplicationMaster 哪些 Containers
可以用，ApplicationMaster 还需要去找 NodeManager 请求分配具体的 Container。

**ApplicationMaster**

运行在 Container 中，向 ResourceManager 申请资源并和 NodeManager
协同工作来运行应用的各个任务然后跟踪它们状态及监控各个任务的执行，和重启失败的任务。

**资源调度流程**

总结上面的内容，可以得出 ResourceManager 是管理集群资源的，ApplicationMaster 是负责管理实际应用程序的，而
ApplicationMaster 是可以开发者自己实现的，这也让 Yarn 成为了一个通用化的资源调度平台，Spark/MapReduce/Flink
等常见的框架都实现了借助 Yarn 来做资源管理。下面串联一下整个流程。

  1. 首先客户端 Client 端是向 ResourceManager 提交的应用程序。
  2. 然后 ResourceManager 会向 NodeManager 请求启动第一个 Container 来运行应用程序的 ApplicationMaster。
  3. ApplicationMaster 被 NodeManager 启动后向 ResourceManager 进行注册，并上报自己的状态。
  4. ApplicationMaster 向 ResourceManager 请求资源来启动 Container 运行应用程序。
  5. ResourceManager 拥有 NodeManager 实时上报的集群资源信息，它分配好资源后将可用的 Container 信息给到 ApplicationMaster。
  6. ApplicationMaster 请求 NodeManager 启动 Container 运行应用程序，
  7. Containers 被 NodeManager 启动后会将运行的进度、状态等信息上报给 ApplicationMaster。
  8. 在应用程序运行期间，客户端可以通过与 ApplicationMaster 通信获取应用的运行状态、进度更新等信息。
  9. 应用程序执行完成后，ApplicationMaster 向 ResourceManager 注销并关闭，同时归还 Containers。

### 资源调度器的种类和特点

ResourceManager 中的资源调度器（Scheduler）是 Yarn
的核心组件，它是一个可拔插的插件，负责为所有的应用按照特定的限制分配资源，Yarn 中实现了 Scheduler 的三种调度方式。

#### **FIFO Scheduler**

FIFO Scheduler 是最基础的调度方式，由于调度逻辑过于简单粗暴，在实际生产环境应用已经比较少了。它会将所有的 Applications
放到同一个队列中，先按照作业的优先级，其次按照应用提交的时间先后，为每个先提交的应用程序分配资源。且分配资源的时候不会考虑其他应用的情况，会尽最大可能满足当前应用的需求，如果资源不够则会阻塞等待。所以，FIFO
这种单队列运行的方式 **不适用于多任务的场景** ，可能会频繁造成任务阻塞无法运行的问题。如下图，Job1 执行完成之后 Job2 才能执行，Job1
运行期间 Job2 只能阻塞等待。

![image.png](https://images.gitbook.cn/3ecbf5d0-e2ba-11ea-b5e8-83076a351173)

#### **Capacity Scheduler**

为了解决 FIFOScheduler 在多任务场景下效率低的问题，CapacityScheduler
将资源按照比例分为多个队列，每个队列拥有独立的资源（CPU、Mem），每个队列还可以进一步划分成层次结构（Hierarchical
Queues），如下图所示。CapacityScheduler
可以支持用户通过指定执行队列来提交应用程序，以此避免资源被其他的应用完全占用而造成阻塞的情况，队列有如下特性：

  * 在每个队列内部，按照 FIFO 方式调度应用程序。
  * 资源共享，当某个队列的资源空闲时，可以将它的剩余资源共享给其他队列。

![image.png](https://images.gitbook.cn/55b681c0-e2ba-11ea-921d-1f9dabf08aa3)

#### **Fair Scheduler**

FairScheduler 是基于最大最小公平算法来分配资源的，允许应用在一个集群中公平地共享资源。FairScheduler 和
CapacityScheduler 有点相似，同样支持以队列划分资源；支持设定最低保证和最大使用上限；支持资源共享，即某队列空闲时可以将资源共享给其他队列。

FairScheduler 中的资源分配有两个比较重要的配置需要说明下：

  * **Fair Sharing 公平共享量** ：集群中某些队列的资源可能会有剩余，这时调度器会自动将这些队列中剩余的资源共享给其他需要资源的队列，其他队列获取的共享资源的量主要由可配置的队列权重（Weigth）决定，权重越大，可获取的资源越多。
  * **Minimum Sharing 最小共享量** ：FairScheduller 允许指定最小共享量（minimum shares to queues），这个配置可以保证所有的应用都能至少获取一定量的资源。如果单个 App 有资源剩余，也是可以将超出的资源分给其他 App 使用。

对于单个队列而言，Fair Scheduler 队列内部支持的资源调度策略比较丰富：

  * FIFO 基本的先进先出策略。
  * FAIR（如果队列中的 N 个作业，每个获得该队列 1/N 的资源）
  * DRF（Dominant Resource Fairness，基于多种资源类型的公平资源分配策略）

### 调度器为空闲节点选择 Container 的策略？

当服务器节点出现空闲资源时，调度器会在该节点上启动新的 Container 来跑应用，但是集群中有那么多队列，队列下有那么多应用，应用又对应了那么多
Container，调度器是如何为这个空闲节点来选择应该启动哪个 Container 的呢？

![image.png](https://images.gitbook.cn/98645c90-e2ba-11ea-b02d-87747a818337)

Capacity Scheduler 和 Fair Scheduler 类似都采用了三级资源分配策略，依次选择队列、应用程序和
Container，如上图所示：

  * **选择资源队列，** 因为队列是个树形结构，所以需要从根队列开始遍历子队列，按照它的子队列资源使用率由小到大依次遍历，找出资源使用率最小的子队列（已使用的资源量/最小队列资源容量），优先为这个队列下的应用分配资源。
  * **选择应用程序，** 找到子队列后，获取该队列下的所有应用程序，按照 Application ID 排序，找到最早提交的应用程序，优先分配资源。
  * **选择具体 Container，** 选好应用程序之后，选择该应用程序中优先级最高的 Container。对于优先级相同的 Containers，优选选择满足本地性的 Container，会依次选择 node local、rack local、no local。

