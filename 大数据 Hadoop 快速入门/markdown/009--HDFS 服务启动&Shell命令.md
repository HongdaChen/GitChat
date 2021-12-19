### 服务启动

安装好 HDFS 服务之后，可以使用以下命令启动 HDFS 集群。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201014051527.png)

因为脚本执行过程中，已经自动将 Hadoop 环境变量和节点间 SSH 免密登录配置好了，所以直接执行 start-dfs.sh 便可以直接启动 HDFS
集群（同时会启动 Yarn）。

    
    
    start-dfs.sh
    

### 文件系统操作命令

#### **操作语法**

使用 shell 命令操作 HDFS，命令格式为 `hadoop fs \<args>` 或者 `hdfs dfs
\<args>`，这两个命令现阶段使用基本没有差异，但留意的是 `hadoop fs \<args>`
使用面最广，可以操作任何文件系统，比如本地文件系统，而 `hdfs dfs \<args>` 只能操作 HDFS 文件系统。

Shell 命令大部分用法和 Linux Shell 类似，可通过 help 查看帮助，常用的命令如表中所示。

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012062758.png)

![](https://gitee.com/QiaoLuManMan/ImageUpload/raw/master/img/20201012062809.png)

接下来操作一下 HDFS 的一些基本 Shell 命令。

1\. 创建目录：在 HDFS 中创建 /file/ 目录和 /tmp/ 目录。

    
    
    # 为了保证足够权限，执行以下命令，切换 HDFS 用户为超级管理员
    export HADOOP_USER_NAME=HDFS
    hadoop fs -mkdir /file
    hadoop fs -mkdir /tmp
    

2\. 修改目录权限：将 HDFS 目录“/file”和“/tmp”的权限改为“rwxrwxrwx”。

    
    
    hadoop fs -chmod -R 777 /file
    hadoop fs -chmod -R 777 /tmp
    

3\. 文件上传：创建本地文件 file.txt，并将文件上传到 HDFS 的 /tmp 目录下。

    
    
    # 创建 file.txt 文件
    touch file.txt
    echo "hello world"  >   file.txt
    # 将文件上传到 HDFS
    hadoop fs -put file.txt /tmp/
    

4\. 查看目录内容：查看 HDFS 目录“/tmp/”的内容。

    
    
    hadoop fs -ls /tmp
    

5\. 查看文件内容：查看 HDFS 文件“/tmp/file.txt”的内容。

    
    
    hadoop fs -cat /tmp/file.txt
    

6\. 下载文件：将 HDFS 中的 /tmp/file.txt 文件下载到本地的 /tmp/ 目录下。

    
    
    hadoop fs -get /tmp/file.txt /tmp/
    // 查看是否下载成功
    cat /tmp/file.txt
    

7\. 文件追加：对“/tmp/file.txt”文件追加内容。

    
    
    // 在 Linux 本地创建文件 file_append.txt
    touch file_append.txt
    echo "hello hdfs"  >  file_append.txt
    // 将 word_append.txt 文件中的内容追加到 HDFS 文件 file.txt 中
    hadoop fs -appendToFile ./file_append.txt /tmp/file.txt
    // 检查是否追加成功
    hadoop fs -cat /tmp/file.txt
    

8\. 文件移动：将 /tmp/file.txt 文件移动到 /file 目录中。

    
    
    hadoop fs -mv /tmp/file.txt /file/
    

9\. 文件拷贝：将 /file/file.txt 文件拷贝到 /tmp 目录中。

    
    
    hadoop fs -cp /file/file.txt /tmp/
    

10\. 查看文件末尾内容：查看 HDFS 文件“/tmp/file.txt”末尾 1000 字节的内容。

    
    
    hadoop fs -tail /tmp/file.txt
    

11\. 查看文件长度：查看 HDFS 文件“/tmp/file.txt”的长度。

    
    
    hadoop fs -du -s /tmp/file.txt
    

12\. 删除文件：将 HDFS 上的“/tmp/file.tx”文件进行删除。

    
    
    // 删除/tmp/file.txt 文件
    hadoop fs -rm -f /tmp/file.txt
    // 查看是否删除成功
    hadoop fs -ls /tmp/
    

#### **HDFS URI**

在前面的操作中，使用的 HDFS 路径为简写，直接从根目录开始，如 /parent/child，因为客户端中已经配置了 HDFS 的服务信息。

但一般情况下，会使用 HDFS 的路径全写，如 hdfs://nameservice/parent/child，其中 nameservice 为 HDFS
服务地址 IP 和端口，可以为 192.168.31.41:9000，默认端口在配置文件中可以进行修改。

使用全写，命令则变成了如下形式：

    
    
    hadoop fs -get hdfs://192.168.31.41:9000/file/file.txt /tmp/
    

