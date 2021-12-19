### 环境规划

#### **操作系统及组件版本**

各组件版本如下，学习环境尽量保持一致，避免版本不一致带来的操作问题。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012060639.png)

#### **集群规划**

使用 3 台虚拟机来进行搭建集群，分别为 Node01、Node02、Node03。集群的规划如下：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012060759.png)

其中 Hadoop 一共 3 个节点，主节点搭建在 Node01 上，从节点在 Node01~Node03 上分别有一个。

### 虚拟机准备

#### **安装说明 &文件下载**

下载并安装 [Virtual Box](https://www.virtualbox.org/wiki/Downloads)。

准备并安装 3 台 CentOS7.2 的虚拟机，主机名命名为 node01、node02、node03。

虚拟机的安装可以使用纯系统镜像，安装后配置主机名。但过程会比较繁琐，学习环境讲求开箱即用，尽量少的在环境上花费时间，否则会打击学习的热情。所以，也可以直接导入已经配置好的虚拟机镜像文件，方便使用。

使用纯镜像安装，下附 CentOS 镜像下载地址：

> <https://pan.baidu.com/s/1CV0C7bZ0-7tKziNf6RmdVg>
>
> 提取码：h931

推荐直接导入虚拟机镜像文件，下附虚拟机镜像下载地址：

> <https://pan.baidu.com/s/1T1VhTv6EwGAr2odlAUPBtA>
>
> 提取码：pljf

#### **虚拟机镜像文件导入流程**

1\. 下载虚拟机镜像文件：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200819093601.png)

2\. 打开 Virtual Box，选择导入虚拟电脑：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200819093850.png)

3\. 选择文件位置，进行导入：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200819093919.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200819094017.png)

4\. 配置虚拟机，自定义将虚拟机文件存放到指定目录，然后点击确定，完成导入。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/image-20200819094401086.png)

5\. 依次导入 Node01、Node02、Node03：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200819094828.png)

6\. 开启虚拟机，使用 root/123456 进行登录：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/image-20200819095050483.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200819095220.png)

7\. 修改虚拟机 IP 地址：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200819095431.png)

    
    
    vim /etc/sysconfig/network-scripts/ifcfg-enp0s3
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200819095656.png)

8\. 使用 XShell，或者其它远程 SSH Linux 登录工具进行远程连接虚拟机：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200819102338.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200819102355.png)

### 自动化安装脚本准备

1\. 下载并上传自动化安装脚本
[automaticDeploy.zip](https://github.com/MTlpc/automaticDeploy/archive/master.zip)
到虚拟机 node01 中。

    
    
    wget https://github.com/MTlpc/automaticDeploy/archive/master.zip
    

2\. 解压 automaticDeploy.zip 到 /home/hadoop/ 目录下：

    
    
    mkdir /home/hadoop/
    unzip master.zip -d /home/hadoop/
    mv /home/hadoop/automaticDeploy-master /home/hadoop/automaticDeploy
    

3\. 更改自动化安装脚本的 frames.txt 文件，配置组件的安装节点信息（如无特殊要求，默认即可）：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200807071331.png)

4\. 编辑自动化安装脚本的 configs.txt 文件，配置 MySQL、Keystore 密码信息（如无特殊要求，默认即可，末尾加 END
表示结束）：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200823094848.png)

5\. 编辑 host_ip.txt 文件，将 3 台虚拟机节点信息添加进去（需自定义进行修改）：

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200807071550.png)

6\. 对 /home/hadoop/automaticDeploy/ 下的 Hadoop、systems 所有脚本添加执行权限：

    
    
    chmod +x /home/hadoop/automaticDeploy/hadoop/* /home/hadoop/automaticDeploy/systems/*
    

### 大数据环境一键安装

1\. 下载 frames.zip 包，里面包含大数据组件的安装包，并上传到 node01 中。

> <https://pan.baidu.com/s/17T3zIbedTaQgk1knxvchPA>
>
> 提取码：cvtq

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200823074409.png)

2\. 将 frames.zip 压缩包，解压到 /home/hadoop/automaticDeploy 目录下：

    
    
    unzip frames.zip -d /home/hadoop/automaticDeploy/
    

3\. 将自动化脚本分发到其它两个节点：

    
    
    # 需提前在另外两个节点创建 /home/hadoop 目录（此时还未配置 hosts，需将 node02\node03 替换为对应 IP）
    ssh root@node02 "mkdir /home/hadoop"
    ssh root@node03 "mkdir /home/hadoop"
    scp -r /home/hadoop/automaticDeploy root@node02:/home/hadoop/
    scp -r /home/hadoop/automaticDeploy root@node03:/home/hadoop/
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200823075132.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200823075323.png)

4\. 依次在各个节点执行 systems/batchOperate.sh 脚本，完成环境初始化：

    
    
    /home/hadoop/automaticDeploy/systems/batchOperate.sh
    

为了避免脚本中与各个节点的 SSH 因为环境问题，执行失败，需要手动测试下与其它节点的 SSH 情况，如果失败，则手动添加。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200823080120.png)

失败后重新添加 SSH：

    
    
    ssh-copy-id node02
    

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200823080401.png)

5\. 在各个节点执行脚本，安装 Hadoop 集群：

    
    
    /home/hadoop/automaticDeploy/hadoop/installHadoop.sh
    source /etc/profile
    # 在 Node01 节点执行，初始化 NameNode
    hadoop namenode -format
    # 在 Node01 节点执行，启动 Hadoop 集群
    start-all.sh
    

6\. 使用本地浏览器访问 node01:50070，成功则搭建成功。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20200823101155.png)

### 基本配置

虽然已经使用脚本，快速完成了集群的搭建，但关于 HDFS 的一些基本配置，仍然需要学习和掌握。

#### **配置文件**

HDFS 的配置文件有四个：core-site.xml、hdfs-site.xml、hadoop-env.sh、slaves，在 hadoop 安装目录下的
etc/hadoop 目录中可以找到。使用脚本搭建的 hadoop 集群，安装目录为：/opt/app/hadoop-2.7.7。

其中 core-site.xml 是 Hadoop 的全局配置文件，因为 HDFS 是 Hadoop 的一个子项目，而完整的 Hadoop 包含
HDFS、Yarn、MapReduce。

所以 core-site.xml 主要是对整个 Hadoop 集群进行全局配置。

而 hdfs-site.xml 是 HDFS 的局部配置文件，对 HDFS 做一些针对性配置、优化。

hadoop-env.sh 设置了 Hadoop 集群运行所需的环境变量，因为 Hadoop 是 Java 语言开发，运行在 JVM 之上，所以这里主要进行
Java 环境配置和 JVM 调优。

slaves 文件用于配置 HDFS 集群的从节点 DataNode，使用 start-all.sh 命令时，会依次在各个节点上启动
DataNode。它的配置内容如下：

    
    
    #localhost
    node01
    node02
    node03
    

#### **core-site.xml 全局配置**

core-site.xml 和 hdfs-site.xml 文件一样，都使用 xml 标签进行配置。其中 `<configuration>`
标签中包含多个配置项 `<property>`，每个 `<property>` 标签中，使用 `<name>` 和 `<value>`
标签来设置配置的名称与所对应的值。

为保证基础的 HDFS 运行，只需要在这里配置 HDFS 的服务地址即可。这里将 HDFS 服务端口配置为 9000，nameservice 更改为
HDFS NameNode 的 IP 地址。

    
    
    <configuration>
       <property>
          <name>fs.defaultFS</name>
          <value>hdfs://nameservice:9000</value>
       </property>
    </configuration>
    

#### **hdfs-site.xml 局部配置**

hdfs-site.xml 配置格式与 core-site.xml 一致，常见的 hdfs-site.xml 的配置项如下图所示。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201014044709.png)

其中，dfs.namenode.name.dir 和 dfs.datanode.data.dir 是必须配置的，如果不进行手动配置的话，namenode 和
datanode 中存储的文件默认会被存放到磁盘的 /tmp 目录下，服务器重启则会造成数据丢失。

其它的配置项，如 dfs.blocksize 可以设置 block 大小，dfs.replication 可以设置默认副本数，可以按需进行配置。

