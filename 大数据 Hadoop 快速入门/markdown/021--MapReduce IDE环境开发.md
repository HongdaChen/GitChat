### 基于 IDEA 编写 MapReduce

在开发过程中，使用 IDE 集成环境进行代码开发和测试，是最为便捷的。接下来讲解下如何使用 IDEA 进行 MapReduce 代码的开发。

首先在 IDEA 中创建 Java 项目。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027041424.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027041505.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027041528.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027041555.png)

选择覆盖当前窗口，开发时尽量保证桌面整洁，用不到的项目窗口就尽量关掉。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027041613.png)

接下来配置下需要在 Hadoop 安装目录中获取开发所需要的 jar 包，MapReduce 开发需要 common、yarn、mapreduce 目录下的
jar 包。

    
    
    cd $HADOOP_HOME/share/hadoop
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027053715.png)

包含的 jar 包如下图所示，Shell 工具使用的是 Xshll 的话，可以用 sz 命令进行下载。当然所有 jar
包已经传到百度云，也可以直接下载使用。

链接：

> <https://pan.baidu.com/s/16ZPbmf5YVh9KAvAfFgN_6w>
>
> 提取码：cacn

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027053858.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027053949.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027054108.png)

下载好所需 jar 包后，在项目中进行添加。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027041655.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027041734.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027041818.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027041844.png)

有 Maven 使用经验的可以直接导入以下依赖。

    
    
    <!-- https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-client -->
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-client</artifactId>
        <version>2.7.7</version>
    </dependency>
    

依赖添加完成后，在项目中创建 Java 文件，将前面的 wordcount 代码拷贝进去。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027041927.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027042010.png)

然后进行运行环境的配置。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027042306.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027042717.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027044855.png)

这里注意的是，程序的输入参数为 `{wc_input/*} wc_output`，为文件的输入和输出目录。

这是因为输入文件定位到目录级别会报错，所以使用通配符 `*` 来指定目录下的所有文件，当然也可以直接使用 wc_input/word.txt 指定单个文件。

接下来便在项目目录下分别创建文件的输入目录和结果输出目录。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027042913.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027043131.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027045132.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027043221.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027043250.png)

在本地执行 MapReduce 程序。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027043311.png)

运行完毕后，在 wc_output 目录下查看运行结果。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027045313.png)

### 将 MapReduce 程序提交到集群

首先使用 IDEA 将编写好的 MapReduce 程序打成 jar 包。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027052441.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027052526.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027052606.png)

需要移除所有的 Hadoop 依赖，因为 Hadoop 集群中已经存在了。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027052712.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027052731.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027052758.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027052815.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027052851.png)

然后将 jar 包通过 Shell 工具上传到集群，如果使用的是 Xshell，可以直接使用 rz 命令进行上传。

之后运行 jar 包，完成 MapReduce 词频统计。

    
    
    hadoop jar MapreduceApp.jar /tmp/mr/data/wc_input /tmp/mr/data/ide_wc_output
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027052218.png)

执行完成后，查看结果。

    
    
    hadoop fs -cat /tmp/mr/data/ide_wc_output/part-r-*
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201027052347.png)

