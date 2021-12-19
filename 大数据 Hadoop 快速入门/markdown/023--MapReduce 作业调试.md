### History Server 开启

因为 Yarn 集群重启之后，作业的历史运行日志和信息就被清理掉了，对于定位历史任务的错误信息很不友好，所以首先开启 History Server
用于保存所有作业的历史信息。

首先编辑 yarn-site.xml 文件，开启 Yarn 的日志聚合功能。

    
    
    cd $HADOOP_HOME/etc/hadoop
    vim yarn-site.xml
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027062047.png)

添加的配置如下：

    
    
      <property>
          <name>yarn.log-aggregation-enable</name>
          <value>true</value>
      </property>
    

然后编辑 mapred-site.xml 文件，添加 History-Server 基本配置：

    
    
    cd $HADOOP_HOME/etc/hadoop
    vim mapred-site.xml
    

添加的配置如下：

    
    
      <property>
          <name>mapreduce.jobhistory.address</name>
          <value>node01:10020</value>
      </property>
        <property>
          <name>mapreduce.jobhistory.webapp.address</name>
          <value>node01:19888</value>
      </property>
        <property>
          <name>mapreduce.jobhistory.intermediate-done-dir</name>
          <value>/mr-history/log</value>
      </property>
        <property>
          <name>mapreduce.jobhistory.done-dir</name>
          <value>/mr-history/done</value>
      </property>
    

因为集群搭建脚本中已经添加了前 2 个配置项，所以只需要添加后 2 个即可。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027062625.png)

配置的具体含义如下：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027062659.png)

然后重启集群，使配置生效：

    
    
    stop-all.sh
    start-all.sh
    

启动 history-server：

    
    
    mr-jobhistory-daemon.sh start historyserver
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027063242.png)

此时，可以在 hdfs 中看到 HistoryServer 的存储目录就被创建出来了。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027063347.png)

于是，现在历史作业的运行信息就可以被保留下来了，但前提是在 history-server 在启动的情况下。

### 辅助脚本

#### **作业清理 &提交**

MapReduce 任务在集群中提交时，如果报错，则需要清理环境，删除 jar 包和中间编译的文件，并且在 HDFS 中删除结果输出目录。

如果频繁进行调试，那重复删除便会花费很多的时间，所以可以把这部分内容放置到脚本中去，节省时间。

    
    
    #!/bin/bash
    
    rm -rf SumDeptSalary.*
    
    hadoop fs -rm -r /tmp/mr/data/sum_output
    
    touch SumDeptSalary.java
    

除此之外，Java 程序的编译和提交也是重复工作，在测试过程中也可以加到脚本中。

    
    
    #!/bin/bash
    
    export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
    hadoop com.sun.tools.javac.Main SumDeptSalary.java
    jar cf SumDeptSalary.jar SumDeptSalary*.class
    hadoop jar SumDeptSalary.jar SumDeptSalary /tmp/mr/data/dept_input/dept /tmp/mr/data/emp_input/emp /tmp/mr/data/sum_output
    

这些脚本是减少重复执行的时间，为了方便，所以不必写的那么通用，每次测试时重写一个就可以了。

#### **日志查看**

再有就是，MapReduce 程序在集群中进行调试时，可以在程序中添加 System.out 来输出信息，当然更推荐使用 Log4j 日志打印。

其中 System.out 输出的信息，会存放到 $HADOOP_HOME/logs/userlogs 目录下对应 application 的中不同
container 输出的 stderr 和 stdout 文件中。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027064253.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027064541.png)

而 Log4j 日志会存放到 syslog 中。

因为 Hadoop 依赖中已经添加了 Log4j 的日志包，所以在程序中直接使用即可。

    
    
    // 定义成员变量 logger，这里定义到了 Mapper 类 MapClass 中
    public Logger logger = Logger.getLogger(MapClass.class.getName());
    // 在函数中直接调用 logger 进行日志打印
    logger.info("Log4j：进入 SetUp 方法");
    

其中 Log4j 的配置文件存放在 $HADOOP_HOME/etc/hadoop 目录下，可以自定义修改。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027071053.png)

程序输出日志后，然而寻找这些日志信息，首先需要在 web 监控界面查看任务的 appication id 和任务被提交到了哪些 NodeManager
中执行，然后分别进入到对应 NodeManager 节点中查看这些日志，而且因为每次执行的 application id
不同，导致找到并进入准确的目录花费的时间较长。所以可以编写一个日志查看脚本，从所有节点查询 application id 对应的日志并返回。

观察 application id，末尾的序号是顺序递增的；在虚拟机测试环境中，不同于生产环境的严谨，只需要关注末尾的序号即可，比如
0001、0002。所以在脚本中传入的 $1 参数为序号，直接正则匹配 `application\_*_${1}` 即可。

提供以下脚本，遍历所有从节点，并输出 syslog 日志。

    
    
    #!/bin/bash
    
    if [ $# -le 0 ]
    then
        echo 缺少参数
        exit 1
    fi
    
    for n in `cat $HADOOP_HOME/etc/hadoop/slaves`
    
    do
        echo ===========查看节点 $n============
        ssh $n "cat $HADOOP_HOME/logs/userlogs/application_*_${1}/container_*/syslog"
    done
    

使用时，加上 grep 命令过滤，效果更佳。

