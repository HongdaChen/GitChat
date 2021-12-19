### 系统架构

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201015051619.png)

Yarn 在架构上是主从架构，其中 Resource Manager 是主节点，Node Manager 是从节点。在架构部署上，Node Manager
与 HDFS 的 DataNode 安装在同一节点，以便将计算任务移动到数据上。

其中主节点 Resource Manager 可以有热备节点，以实现集群高可用。当前主节点为 Active 状态，热备节点为 Standby 状态。

客户端 Client 向主节点（Resource Manager）提交作业后，Resource Manager 会在 Node Manager
上为当前作业（Job）分配资源。

Node Manager 上分配的资源会封装成 Container，相当于是一个容器（小型虚拟机），它包含了作业运行必备的 CPU、内存、环境变量等资源。

Resource Manager 分配完资源之后，之前在 Hadoop 1.x 中提到的作业管理怎么办？由作业自己进行管理。

作业（Job）提交后，Resource Manager 首先会找一个空闲的 Node Manager 节点，分配一个 Container。Job
首先会在这个 Container 中启动和运行自己的作业管理程序 Application Master，然后 Application Master 调用
Job 的核心代码，生成 Task 任务；之后 Application Master 根据生成的 Task 数量，再向 Resource Manager
申请资源，Resource Manager 将 Container 资源分配给 Application Master 之后，Application
Master 便将 Task 调度到 Container 中执行。执行过程中，Task 的运行状态会实时向 Application Master
进行汇报，由 Application Master 进行管理和调度。

所以可以看到，Resource Manager 现在只负责资源的分配，根据需求分配 Container，至于 Container 里究竟运行
Application Master 还是 Task，由程序自己来决定。

这种架构，减轻了 Yarn 主节点 Resource Manager 的负担，而且更加通用和灵活；只要分布式计算引擎任务，实现了 Application
Master 的接口，用于向 Yarn
申请资源，拿到资源后，便可以完全由程序自己控制运行流程和模式；基于这种方式，甚至可以自己实现自定义的分布式计算任务，运行在 Yarn 之上。

### 工作机制

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201020035122.png)

Yarn 的工作机制，在前面讲架构的时候已经提到了，这里再梳理一下。

  1. 客户端 Client 向 Resource Manager 提交作业（Job）。
  2. Resource Manager 会在所有 Node Manager 中的相对空闲节点中为 Job 分配 Container，Node Manager 启动 Container 之后，Job 便在 Container 中启动 Application Master。
  3. Application Master 首先向 Resource Manager 注册，然后调用程序，生成 Task，并根据 Task 数量向 Resource Manager 申请资源
  4. Resouce Manager 根据 Application 的申请，在 Node Manager 上启动相应 Container 资源，并返回给 Application Master 资源列表，Application Master 根据资源列表，将 Task 分发到 Container 中执行。
  5. Task 运行过程中，会向 Application Master 实时汇报状态和进度。
  6. 所有的 Task 执行完成后，Application Master 向主节点 Resource Manager 申请释放资源，Resouce Manager 会将所有的 Container 全部释放。

### 高可用

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201015064109.png)

Resource Manager 可以实现高可用，首先需要 2 个及以上的 Resource Manager 节点，其中 1 个 Active
Resource Manager、多个 Standby Resource Manager。Active Resource Manager
在运行过程中，会将必要信息存储到 Zookeeper 中进行保存。当 Active Resource Manager 宕机后，由 Zookeeper 从其他
Standby 节点中选举，选出新的 Active Resource Manager，并从 Zookeeper 中恢复数据。

新的 Active 节点将所有的 ApplicationMaster 重启，杀死所有运行中的 Container。

Resource Manager 的主从切换，会自动触发，也可以手动使用命令进行切换。

