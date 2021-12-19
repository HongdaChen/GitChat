前面内容已经有很多学习案列带大家正式使用了 Flink，其中不仅有讲将 Flink 应用程序在 IDEA 中运行，也有讲将 Flink Job
编译打包上传到 Flink UI 上运行，在这 UI 背后可能是 YARN、Mesos、Kubernetes。那么这节就系统讲下如何部署和运行我们的
Flink Job，大家可以根据自己公司的场景进行选择！

在讲解完 Flink 中的配置后，接下来接着讲解 Flink 的部署情况。

### Standalone

第一种方式就是 Standalone 模式，这种模式笔者在前面 2.2 节里面演示的就是这种，我们通过执行命令：`./bin/start-
cluster.sh` 启动一个 Flink Standalone 集群。

    
    
    zhisheng@zhisheng  /usr/local/flink-1.9.0  ./bin/start-cluster.sh
    Starting cluster.
    Starting standalonesession daemon on host zhisheng.
    Starting taskexecutor daemon on host zhisheng.
    

默认的话是启动一个 Job Manager 和一个 Task Manager，我们可以通过 `jps` 查看进程有：

    
    
    65425 Jps
    51572 TaskManagerRunner
    51142 StandaloneSessionClusterEntrypoint
    

其中上面的 TaskManagerRunner 代表的是 Task Manager
进程，StandaloneSessionClusterEntrypoint 代表的是 Job Manager 进程。上面运行产生的只有一个 Job
Manager 和一个 Task Manager，如果是生产环境的话，这样的配置肯定是不够运行多个 Job 的，那么我们该如何在生产环境中配置
standalone 模式的集群呢？我们就需要修改 Flink 安装目录下面的 conf 文件夹里面配置：

    
    
    flink-conf.yaml
    masters
    slaves
    

将 slaves 中再添加一个 `localhost`，这样就可以启动两个 Task Manager 了。接着启动脚本 `start-
cluster.sh`，启动日志显示如下：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-23-161333.png)

可以看见有两个 Task Manager 启动了，再看下 UI 显示的：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-23-161431.png)

那么如果还想要添加一个 Job Manager 或者 Task Manager 怎么办？总不能再次重启修改配置文件后然后再重启吧！这里你可以这样操作。

**增加一个 Job Manager** ：

    
    
    bin/jobmanager.sh ((start|start-foreground) [host] [webui-port])|stop|stop-all
    

但是注意 Standalone 下最多只能运行一个 Job Manager。

**增加一个 Task Manager** ：

    
    
    bin/taskmanager.sh start|start-foreground|stop|stop-all
    

比如我执行了 `./bin/taskmanager.sh start` 命令后：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-23-161657.png)

Standalone 模式下可以先对 Flink Job 通过 `mvn clean package` 编译打包，得到 Jar 包后，可以在 UI
上直接上传 Jar 包，然后点击 Submit 就可以运行了。

### YARN

Flink 不仅仅支持以 standalone 模式运行，还支持在 YARN 上运行，YARN 是一种新的 Hadoop
资源管理器，它是一个通用资源管理系统，可为上层应用提供统一的资源管理和调度，它的引入为集群在利用率、资源统一管理和数据共享等方面带来了巨大好处。

相当于 standalone 模式，Flink on YARN 有如下好处：

  * 资源按需使用，提高集群的资源利用率
  * 基于 YARN 调度系统，能够自动处理各个角色的 failover（Job Manager 进程异常退出，Yarn ResourceManager 会重新调度 Job Manager 到其他机器；如果 Task Manager 进程异常退出，Job Manager 收到消息后并重新向 Yarn ResourceManager 申请资源，重新启动 Task Manager）

下图是 Flink on YARN 的架构图：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-04-27-004-flink-on-yarn.png)

[官网](https://ci.apache.org/projects/flink/flink-docs-
release-1.9/ops/deployment/yarn_setup.html) 对 Flink On YARN 讲解很多，包含 Flink 在
YARN 上的安装方式、 Flink YARN Session 和怎么允许单一的 Flink Job、怎么查看在 YARN
上查看日志，已经非常全了，大家可以多多参考官网。

### Mesos

Apache Mesos 诞生于 UC Berkeley 的一个研究项目，现已成为 Apache Incubator 中的项目。Apache Mesos
把自己定位成一个数据中心操作系统，它能管理上万台的从机。Framework 相当于这个操作系统的应用程序，每当应用程序需要执行，Framework 就会在
Mesos 中选择一台有合适资源（CPU、内存等）的从机来运行。

Flink 也是支持在 Mesos 上部署运行的，在[官网](https://ci.apache.org/projects/flink/flink-
docs-release-1.9/ops/deployment/mesos.html)也有介绍，主要是讲 Flink 运行在
DC/OS（它是具有复杂应用程序管理层的 Mesos 分支，预装了Marathon），如果安装好了 DC/OS 的话，那么你可以使用它的 CLI 工具来安装
Flink，比如：

    
    
    dcos package install flink
    

在 Mesos 上运行 Flink 有两种方式：Flink 会话集群（session cluster）和作业集群（job cluster）。

#### 会话集群

Flink 会话集群需要在运行的 Mesos 上部署执行，然后你可以在一个会话集群上运行多个 Flink
作业，在部署会话集群之后，需要将每个作业提交给集群。在 Flink 的安装目录 bin 下，你可以找到 2 个在 Mesos 上启动 Flink 的脚本。

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-23-161843.png)

  * mesos-appmaster.sh：它将启动 Mesos 应用程序主程序，会注册 Mesos 调度程序，负责启动工作节点

  * mesos-taskmanager.sh：Mesos 进程的入口点，你不需要手动执行该脚本，它由 Mesos 工作节点自动启动来启动新的 Task Manager。

在运行 mesos-appmaster.sh 脚本之前，你需要在 flink-conf.yaml 中定义 mesos.master 配置或者通过启动参数
`-Dmesos.master=...` 传给进程。当执行 mesos-appmaster.sh 后，它会在执行该脚本的机器上创建一个 Job
Manager，另外就是 Task Manager 将作为 Mesos 集群中的任务来运行。

#### 作业集群

Flink 作业集群是一个运行单个作业的专用集群，不需要额外的作业提交。在 Flink 的安装目录 bin 下面有 `mesos-appmaster-
job.sh` 脚本，该脚本会启动 Mesos 应用程序主程序，会注册 Mesos 调度程序，检索到 JobGraph 后相应的启动 Task
Manager。

在执行 mesos-appmaster-job.sh 脚本之前，你需要在 flink-conf.yaml 中定义 mesos.master 和
internal.jobgraph-path 或者你可以通过启动参数 `-Dmesos.master=... -Dinterval.jobgraph-
path=...` 传入进程。

你可以这样获取到 JobGraph：

    
    
    final JobGraph jobGraph = env.getStreamGraph().getJobGraph();
    jobGraph.setAllowQueuedScheduling(true);
    final String jobGraphFilename = "job.graph";
    File jobGraphFile = new File(jobGraphFilename);
    try (FileOutputStream output = new FileOutputStream(jobGraphFile);
        ObjectOutputStream obOutput = new ObjectOutputStream(output)){
        obOutput.writeObject(jobGraph);
    }
    

通常你可以像下面这样全部通过启动参数来传入配置：

    
    
    bin/mesos-appmaster.sh \
        -Dmesos.master=master.foobar.org:5050 \
        -Djobmanager.heap.mb=1024 \
        -Djobmanager.rpc.port=6123 \
        -Drest.port=8081 \
        -Dmesos.resourcemanager.tasks.mem=4096 \
        -Dtaskmanager.heap.mb=3500 \
        -Dtaskmanager.numberOfTaskSlots=2 \
        -Dparallelism.default=10
    

更多关于 Flink On Mesos 的安装以及配置可以访问
[官网](https://ci.apache.org/projects/flink/flink-docs-
release-1.9/ops/deployment/mesos.html) 查看。

### Kubernetes

Kubernetes（k8s）是 Google 开源的容器集群管理系统，在 Docker
技术的基础上，为容器化的应用提供部署运行、资源调度、服务发现和动态伸缩等一系列完整功能，提高了大规模容器集群管理的便捷性。它是一个完备的分布式系统支撑平台，具有完备的集群管理能力，多层次的安全防护和准入机制、多租户应用支撑能力、透明的服务注册和发现机制、內建智能负载均衡器、强大的故障发现和自我修复能力、服务滚动升级和在线扩容能力、可扩展的资源自动调度机制以及多粒度的资源配额管理能力。同时
Kubernetes 提供完善的管理工具，涵盖了包括开发、部署测试、运维监控在内的各个环节。

因为 Kubernetes 的好处这么多，所以现在好多公司也开始使用 Kubernetes，那么 Flink 也有必要支持部署在 Kubernetes
上，在[官方文档](https://ci.apache.org/projects/flink/flink-docs-
release-1.9/ops/deployment/kubernetes.html)里面也介绍了两种部署的方法：

  * Flink session cluster

  * Flink job cluster

下面我演示一下如何部署 Flink session cluster 在 K8s 上。首先你需要创建 jobmanager-
service、jobmanager-deployment、taskmanager-deployment，在利用 kubectl
命令创建之前你需要分别创建这几个 yaml 文件。

  * jobmanager-service.yaml

    
    
    apiVersion: v1
    kind: Service
    metadata:
      name: flink-jobmanager
    spec:
      ports:
      - name: rpc
        port: 6123
      - name: blob
        port: 6124
      - name: query
        port: 6125
      - name: ui
        port: 8081
      selector:
        app: flink
        component: jobmanager
    

  * jobmanager-deployment.yaml

    
    
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: flink-jobmanager
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app: flink
            component: jobmanager
        spec:
          containers:
          - name: jobmanager
            image: flink:latest
            args:
            - jobmanager
            ports:
            - containerPort: 6123
              name: rpc
            - containerPort: 6124
              name: blob
            - containerPort: 6125
              name: query
            - containerPort: 8081
              name: ui
            env:
            - name: JOB_MANAGER_RPC_ADDRESS
              value: flink-jobmanager
    

  * taskmanager-deployment.yaml

    
    
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: flink-taskmanager
    spec:
      replicas: 2
      template:
        metadata:
          labels:
            app: flink
            component: taskmanager
        spec:
          containers:
          - name: taskmanager
            image: flink:latest
            args:
            - taskmanager
            ports:
            - containerPort: 6121
              name: data
            - containerPort: 6122
              name: rpc
            - containerPort: 6125
              name: query
            env:
            - name: JOB_MANAGER_RPC_ADDRESS
              value: flink-jobmanager
    

创建好这三个文件后，分别执行下面三个命令：

    
    
    kubectl create -f jobmanager-service.yaml
    
    kubectl create -f jobmanager-deployment.yaml
    
    kubectl create -f taskmanager-deployment.yaml
    

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-05-31-070748.jpg)

然后去 K8s 的 Dashboard 上面查看 Flink 的情况：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-05-31-071343.jpg)

我们如果要看 Flink 自带的 UI 的话需要将端口映射一下，使用如下命令：

    
    
    kubectl port-forward service/flink-jobmanager 8081:8081
    

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-05-31-071954.jpg)

然后访问 <http://localhost:8081> 就可以看到 Flink 自带的 UI 了：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-05-31-072103.jpg)

如果我们要提交 Job 的话，我们先用命令行来操作一下：

    
    
    ./bin/flink run -d -m localhost:8081 ~/word-count.jar
    

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-23-162109.png)

执行完命令后的话，就可以去页面看到刚才提交的 Job 了：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-23-162157.png)

另外你也可以通过 Flink UI 上传 Jar 包把 Job run 起来。这里再对上面这几个配置文件进行讲解：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-05-31-073347.jpg)

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-05-31-073411.jpg)

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-05-31-073434.jpg)

启动完了之后的话，如果你想删除就需要使用下面命令：

    
    
    kubectl delete -f jobmanager-deployment.yaml
    
    kubectl delete -f taskmanager-deployment.yaml
    
    kubectl delete -f jobmanager-service.yaml
    

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-05-31-075628.jpg)

### 小结与反思

这部分介绍了下 Flink 的所有配置文件及其配置文件中的参数的作用，然后讲解了 Flink 的多种部署方式，比如
Standalone、YARN、Mesos、Kubernetes。每种方式的差异性还不小，如果你们公司没有使用类似 YARN、Mesoss、K8S
等分布式调度系统，那么只好使用 Standalone 的 Flink 集群了，这种模式比较简单，可以直接使用 Flink 自带的 UI 上传作业 Jar
包，不像和 YARN、K8S 这种一样会有多种方式供你选择。根据平时在社群里面的答疑，目前将 Flink 部署在 YARN 上的是非常多的，Flink on
K8S 好多公司也处于在调研阶段。其实具体该选择哪种方式运行 Flink，最主要还是得看自己公司的架构选型和允许分配的资源（人力 & 机器资源 &
作业的数量）。

附：Confluence 上有个 FLIP 讲 Flink 的部署，感兴趣可以看看。

  * [FLIP-6 - Flink Deployment and Process Model - Standalone, Yarn, Mesos, Kubernetes, etc](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65147077)

