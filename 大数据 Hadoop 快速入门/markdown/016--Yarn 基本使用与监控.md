### 基本使用

Yarn 是一个资源管理框架，所以它可以对提交到集群中的任务进行查看，并可以强制结束这些任务。

它常用的 Shell 命令有：

    
    
    yarn  application  [command_options]
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201018044127.png)

一般使用流程，是先用 list 查看集群中未完成的所有任务以及它的 ID，如果想查看任务详细信息则使用 status，如果想强制终止任务则使用 kill。

    
    
    # 查看 Yarn 中未完成的所有任务
    yarn application -list
    # 查看某个任务的运行状态
    yarn application -status <Application ID>
    # 强制终止某个任务
    yarn appication -kill <Application ID>
    

首先使用 MapReduce 官方自带的案例，提交到 Yarn 集群中运行，然后再将其终止掉。

    
    
    cd $HADOOP_HOME/share/hadoop/mapreduce
    # 计算圆周率，第一个参数为 Map 运行次数，第二个参数为投掷次数（用于计算圆的一种方式，此参数越大，计算出的圆周率越准确）
    hadoop jar hadoop-mapreduce-examples-2.7.7.jar pi 10 10000
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023073859.png)

新打开一个 Shell 窗口，执行 yarn 命令，终止作业运行：

    
    
    yarn application -list
    yarn application -kill <Application ID>
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/image-20201023073921611.png)

虽然知道有个问题大家基本都不会犯，但还是提一下，因为之前授课时，有的学员出现了这个问题。当任务提交到 Yarn
集群中运行的时候，默认情况下，控制台会输出作业运行的 log 信息，此时使用 CTRL^C
不能终止任务，只是停止其在控制台的信息输出，而任务已经提交到分布式集群中去运行了。终止任务，必须先使用 `yarn application -list`
获取进程号，再使用 `-kill` 进行终止。

### 系统配置

Yarn 的基本配置文件为 yarn-site.xml。内容上分别为配置 ResourceManager 和 NodeManager、配置
ResourceManager、配置 NodeManager、配置 History Server 功能。

#### **配置 ResourceManager 和 NodeManager**

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024060038.png)

#### **配置 ResourceManager**

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024060106.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024060113.png)

#### **配置 NodeManager**

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024060129.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024060134.png)

#### **配置 History Server**

因为集群重启之后，运行完成的作业日志会被删除，无法溯源，所以需要开启第三方集群 History Server
专门用于保存和展示日志信息。在此之前，要开启日志聚合功能，将日志存放到 HDFS 中。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024060158.png)

### 运维监控

#### **作业监控**

一般提交到集群中的任务，我们会使用浏览器访问 Resource Manager 的 8088 端口，进入监控页面，如
http://192.168.31.41:8088，来查看任务运行的具体情况。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023074510.png)

访问 Scheduler 界面，可以查看集群调度策略和队列使用情况。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042140.png)

点击页面中的某个任务，可以查看任务的概览情况。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042213.png)

点击任务概览页面的 logs，可以查看任务运行日志。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042227.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042236.png)

当然当集群重启后，这些任务的历史运行数据就被清除了，如果想要永久保存，需要启动 history
server，相当于是一个第三方服务，用于保存作业的历史运行数据。

#### **集群监控**

访问 Resource Manager 的 8088 端口，点击 Abort 标签，进入集群概览页。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042341.png)

访问 8088 监控页面，点击 Nodes 标签，进入节点监控页面。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042357.png)

在 Node Labels 标签中，可以查看集群各个节点标签配置。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042406.png)

在 Tools 工具集中，可以查看集群配置信息（configuration）、本地日志信息（Local logs）、集群运行信息（Server
Stacks）、集群元数据（Server metrics）。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042420.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042425.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042432.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042439.png)

#### **节点信息**

访问 Node Manager 的 8042 端口，进入节点概览页。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042518.png)

访问 Node Manager 的 8042 端口，点击 List of Applications 查看节点上的作业运行情况。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042531.png)

点击 List of Containers 查看节点上 Containers 分配情况。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042541.png)

