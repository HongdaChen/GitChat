### 可视化界面

通过 50070 端口，可以访问 HDFS Web UI：http://activeNameNodeHost:50070，需将
activeNameNodeHost 自行替换为主节点 IP，如 http://192.168.31.41:50070。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023062109.png)

其中 Overview 页面可以查看集群的基本运行情况：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023063523.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023063528.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023063537.png)

DataNode 页面可以查看 DataNode 的使用和退役情况：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023063553.png)

Datanode Volume Failures 页面可以查看 DataNode 卷损坏情况：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023063607.png)

Snapshot 页面用于查看快照情况：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023063611.png)

Startup Progress 页面可以查看 HDFS 启动详情：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023063646.png)

Utilities 工具中有 Browse the file system 可以直观查看 HDFS 文件，logs 则用于查看集群日志：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023063657.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023063700.png)

其中 logs 日志也可以在 \$HADOOP_HOME/logs 目录中查看。其中日志有两种，分别以 log 和 out 结尾。以 log
结尾的日志，是通过 log4j 日志记录格式进行记录的日志，采用的日常滚动文件后缀策略来命名日志文件，内容比较全。以 out
结尾的日志，是记录标准输出和标准错误的日志，内容比较少，默认的情况，系统保留最新的 5 个日志文件。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201025070604.png)

### HDFS 管理命令

#### **NameNode 格式化或恢复**

在第一次启动 HDFS 的时候，需要执行 NameNode 格式化命令，目的是使集群所有 DataNode“认主”。

执行命令后，首先在 NameNode 节点会生成 NameNode ID，然后将 NameNode ID 分发到所有 DataNode
节点，DataNode 会记录到本地磁盘。之后 DataNode 在寻找 NameNode 时，除了根据配置文件中的 IP 与端口，还会根据
NameNode ID 确认 NameNode 的身份。

    
    
    # hdfs  namenode [-format [-clustered cid] [-force] [-nonInteractive] ] | [-recover [-force] ]
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201014053234.png)

首次进行格式化时，只需要使用如下命令：

    
    
    hdfs  namenode -format
    

如果非首次格式化，必须执行 `-force` 选项；否则 NameNode 重新生成了 NameNode ID，而 DataNode 却因为保存了旧的
NameNode ID 从而导致更新失败，这样会导致 DataNode 找不到 NameNode 而发生节点丢失的情况。

    
    
    hdfs  namenode -format -force
    

#### **报告文件系统信息**

在对 HDFS 运维时，可以使用 report 命令，查看 HDFS 文件系统的基本信息和统计信息（如存储的使用情况）。

    
    
    # hdfs dfsadmin [generic_options] [-report [-live] [-dead] [-decommissioning] ]
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201014053912.png)

使用命令报告文件系统信息：

    
    
    hdfs dfsadmin -report
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023063904.png)

#### **检查文件系统健康状况**

如果需要查看 HDFS 更细粒度的状态，如 Block 坏块，副本的丢失率等，可以通过 fsck 命令来进行。

    
    
    hdfs fsck <path>
    

例如查看 /tmp 目录的健康状况。

    
    
    hdfs fsck /tmp
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023064051.png)

#### **安全模式**

当 NameNode 检测到任何异常后，会自动进入安全模式（也支持手动进入），该模式下只支持读操作；进入安全模式后，也可以手动退出，但不推荐。

    
    
    # hdfs  dfsadmin  [generic_options] [-safemode enter | leave | get | wait]
    

但手动进入安全模式后，只能手动退出。

    
    
    # 进入安全模式
    hdfs dfsadmin -safemode enter
    # 退出安全模式
    hdfs dfsadmin -safemode leave 
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023064140.png)

#### **主备切换**

在 HDFS 搭建了高可用集群后，可以使用命令手动进行 NameNode 主备切换。

    
    
    # hdfs haadmin -failover [--forcefence] [--forceactive] <serviceId> <serviceId>
    # hdfs haadmin -getServiceState <serviceId>
    # hdfs haadmin -transitionToActive <serviceId> [--forceactive]
    # hdfs haadmin -transitionToStandby <serviceId>
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201014065633.png)

因为脚本自动化安装的集群是非高可用集群，所以主备切换的命令无法进行测试。

#### **退役和服役**

在生产集群中，如果删除一个 DataNode 节点，肯定不会直接将主机节点物理移除掉，这样的话实际上和 DataNode
直接宕机的情况是没有区别的，会直接触发 HDFS 集群的容灾处理。而增加一个 DataNode
节点，也不会直接添加到集群当中，因为新增的节点没有存储数据，会导致集群出现数据倾斜。

而 HDFS 节点的退役和服役，是 HDFS 提供的，DataNode 在生产集群中扩容和缩容的温和方法。

DataNode 节点需要移除时，首先将它退役，HDFS 便会将数据从 DataNode 节点中进行转移，数据转移完成后，DataNode 状态会由
Inservice 变为 Decommission，之后就可以删除这个节点了。

DataNode 节点需要扩容时，首先将新增节点服役，HDFS 便会将数据拷贝到服役节点，以避免数据倾斜，之后将节点添加到集群中即可。

    
    
    # 读取主机包含和排除文件，以便完成退役和服役
    # hdfs  dfsadmin  [generic_options]  -refreshNodes
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201014072141.png)

#### **数据重分布**

HDFS
中的数据如果出现倾斜（每个节点存储的数据不大致均匀，而有较大的偏差），会导致存储数据较少的节点资源出现浪费，而存储数据较多的节点负载较大，影响集群的稳定性。

一般而言，HDFS 在正常状态下不会出现数据倾斜情况，NameNode
在分配数据时，会优先考虑最空闲的节点。但人为因素，如指定节点进行文件上传；或者系统异常，如 DataNode
宕机较长时间后，重新启动；这些异常场景可能会导致数据倾斜的情况。

数据出现倾斜，可以使用数据重分布命令，对集群数据进行平衡。

    
    
    # hdfs balancer [-threshold <threshold>]
                             [-exclude [-f <hosts-file> | <comma-separated list of hosts>] ]
                             [-include [-f <hosts-file> | <comma-separated list of hosts>] ]
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201014073027.png)

集群平衡的标准是：每个 DataNode 的存储使用率和集群总存储使用率的差值均小于阀值（默认为 10，可以自定义配置）。

接下来对集群进行数据重分布操作。

    
    
    hdfs balancer
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023064558.png)

需要注意的是，数据重分布默认带宽为 1M/s，主要为了 Balance 的同时不影响 HDFS 操作。但为了保证效率，建议 Balance 的时候，带宽设为
10M/s，并且停止操作 HDFS，或者在集群空闲时进行。

    
    
    # hdfs  dfsadmin  [generic_options]  [-setBalancerBandwidth <bandwidth in bytes per second>]
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201014073536.png)

设置重分布时宽带为 10M/s：

    
    
    hdfs dfsadmin -setBalancerBandwidth 1048576
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023064926.png)

#### **分布式拷贝**

Distcp（分布式拷贝）是用于大规模集群内部和集群之间拷贝的命令。它使用 MapReduce 实现文件分发、错误处理恢复，以及报告生成。

普通的 cp 命令并发数少，且只能用于当前集群文件的内部移动。而 Distcp 使用 MapReduce 进行并发处理，主要应用场景是跨集群拷贝，如从当前
HDFS 集群将数据拷贝到其他 HDFS 集群；或者大规模数据移动的情况。

    
    
    # hadoop distcp options [source_path...] <target_path>
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201014074115.png)

#### **配额限制**

HDFS 允许管理员对用户的目录设置 Quota（配额限制），主要从两个维度：文件数量和文件大小。

可以限制指定目录及子目录中的文件总数，或者限制指定目录中的所有文件的容量大小。

    
    
    # hdfs dfsadmin -setSpaceQuota <N> <directory>...<directory>
    Notes: 为每个文件夹设置空间配额
    # hdfs dfsadmin -clrSpaceQuota <directory>...<directory>
    Notes: 移除每个文件夹的空间配额
    # hdfs dfsadmin -setQuota <quota> <dirname>…<dirname>
    Notes: 为每个文件夹设置数量配额
    # hdfs dfsadmin -clrQuota <dirname>…<dirname>
    Notes: 移除每个文件夹的数量配额
    # hadoop fs -count -q [-h] [-v] <directory>...<directory>
    Notes:-q 可查看配额详情. –h 格式化显示. –v 显示首行.
    

现在 HDFS 上创建两个目录 /name_quota 和 /space_quota，其中设置 /name_quota 目录只能存放 100 个文件，而
/space_quota 目录最大容量为 10g。并分别查看这两个目录的配额限制情况。

    
    
    # 分别创建目录
    hadoop fs -mkdir /name_quota
    hadoop fs -mkdir /space_quota
    # 限制/name_quota 目录下文件数为 100
    hdfs dfsadmin -setQuota 100 /name_quota
    # 限制/space_quota 空间为 10G
    hdfs dfsadmin -setSpaceQuota 10g /space_quota
    # 查看目录的配额限制
    hadoop fs -count -q /name_quota
    hadoop fs -count -q /space_quota
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023065551.png)

#### **快照**

HDFS 快照是只读的，记录文件系统在某个时间点的副本，以便于数据恢复。快照可应用于任意目录。

    
    
    # hdfs lsSnapshottableDir
    Notes: 获取用户有权限创建快照的快照表目录.
    # hdfs snapshotDiff <path> <fromSnapshot> <toSnapshot>
    Notes: 比较两个快照的不同，需要对所有的快照有读权限.
    # hdfs dfsadmin -allowSnapshot <path>
    Notes: 允许目录创建快照. 
    # hdfs dfsadmin -disallowSnapshot <path>
    Notes: 禁止目录创建快照. 
    # hdfs dfs -createSnapshot <path> [<snapshotName>]
    Notes: 为目录创建快照，需要对此目录有所有者权限.
    # hdfs dfs -deleteSnapshot <path> <snapshotName>
    Notes: 删除目录快照，需要对此目录有所有者权限
    

创建好的 Snapshot 文件夹在源文件夹下，命名为 .snapshot/[<snapshotName>]。

恢复的时候，直接使用 cp 命令即可。

现将 /tmp 目录执行快照，然后删除 /tmp 目录下的 /tmp/java_data 后，进行恢复，之后将快照删除。

    
    
    # 允许/tmp 目录创建快照
    hdfs dfsadmin -allowSnapshot /tmp
    # 为/tmp 目录创建快照
    hdfs dfs -createSnapshot /tmp back1
    # 删除/tmp 目录
    hdfs dfs -rm -r /tmp/java_data
    # 查找/tmp 的快照
    hdfs lsSnapshottableDir
    hadoop fs -ls /tmp/.snapshot
    # 从快照恢复/tmp 目录
    hdfs dfs -cp /tmp/.snapshot/back1/* /tmp
    # 查看是否恢复成功
    hdfs dfs -ls /tmp/
    # 删除快照
    hdfs dfs -deleteSnapshot /tmp back1
    # 禁止目录快照
    hdfs dfsadmin -disallowSnapshot /tmp
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201023073126.png)

