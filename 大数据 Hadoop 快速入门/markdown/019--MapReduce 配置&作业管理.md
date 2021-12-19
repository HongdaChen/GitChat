### 基本配置

MapReduce 的配置文件为 mapred-site.xml，配置内容分为配置 MapReduce 运行程序、配置 History-Server。

#### **配置 MapReduce 运行程序**

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027033544.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027033555.png)

#### **配置 History-Server**

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027033601.png)

### 提交与管理

MapReduce 任务编写完成后，打包为 jar 包形式，便可以使用客户端对任务进行提交。

提交作业的基本命令为：

    
    
    # hadoop  jar  {jarFile}  [mainClass]  args
      -jarFIle: MapReduce 运行程序的 jar 包
      -mainClass: jar 包中 main 函数所在类的类名
      -args: 程序调用需要的参数，如：输入输出路径
    

这里使用官方自带的 MapReduce 案例包来完成作业提交：

    
    
    cd $HADOOP_HOME/share/hadoop/mapreduce
    # 计算圆周率，第一个参数为 Map 运行次数，第二个参数为投掷次数（用于计算圆的一种方式，此参数越大，计算出的圆周率越准确）
    hadoop jar hadoop-mapreduce-examples-2.7.7.jar pi 10 10000
    

作业提交之后，新打开 Shell 窗口，使用 Yarn 来查看和终止作业：

    
    
    yarn application -list
    yarn application -kill {application_id}
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024061139.png)

当然计算 pi 的案例不需要数据，一般的 MapReduce 作业都需要数据输入，接下来使用官方案例，来完成词频统计任务。

首先准备数据，并上传到 HDFS 中：

    
    
    // 在 HDFS 中创建作业输入目录
    hadoop fs -mkdir -p /tmp/mr/data/wordcount_input
    // 为目录赋权
    hadoop fs -chmod 777 /tmp/mr/data/wordcount_input
    // 在本地创建词频统计文件
    echo -e "hello hadoop\nhello hdfs\nhello yarn\nhello mapreduce" > wordcount.txt
    // 将 wordcount.txt 上传到作业输入目录
    hadoop fs -put wordcount.txt /tmp/mr/data/wordcount_input
    

词频统计作业提交：

    
    
    cd $HADOOP_HOME/share/hadoop/mapreduce
    # 这里有两个参数，第一个是词频文件的输入目录，第二个是计算结果的保存目录（不能提前创建，如果已存在会报错）
    hadoop jar hadoop-mapreduce-examples-2.7.7.jar wordcount /tmp/mr/data/wordcount_input /tmp/mr/data/wordcount_output
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024062156.png)

查看作业输出结果：

    
    
    // 查看输出目录是否创建
    hadoop fs -ls /tmp/mr/data/wordcount_output
    // 查看输出文件内容
    hadoop fs -cat /tmp/mr/data/wordcount_output/part-*
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024062324.png)

### 作业监控与诊断

MapEeduce 作业提交之后，因为提交到 Yarn 集群中，所以可以通过 Yarn 的 Web 界面来查看作业运行情况。

Web 监控地址为：http://{AM _IP}:8088/proxy/{application_
id}/，如：http://192.168.31.41:8088。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024062520.png)

点击页面中的某个任务，可以查看任务的概览情况。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024062738.png)

如果作业运行出错，可以点击任务概览页面的 Logs，查看任务运行日志。其中 stderr、stdout
是标准输出（System.out）的信息，syslog 是 Log4j 日志数据。一般在 MapReduce 中不会使用 System.out，所以主要关注
syslog 中的信息。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042227.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201024042236.png)

除了 Web 界面，还可以到运行日志目录，排查作业出错原因。日志目录可以自定义指定，配置参数为 yarn.nodemanager.log-dirs。默认值为
$HADOOP_HOME/logs，作业的运行日志便存放在 userlogs 目录下。

从监控页面中已经看到，作业提交到了 node01 中，所以到 node01 中查看运行日志。

    
    
    cd $HADOOP_HOME/logs/userlogs
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201025071640.png)

根据作业 id，进入对应目录查看运行日志。

    
    
    cd {application_id}
    cd {container_id}
    cat syslog
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201025072436.png)

