### 前言

本节课我们进入 NoSQL 的学习，当前主流的 NoSQL 产品是 MongoDB 和 Redis，对于习惯使用关系型数据库的开发者来说，MongoDB 比
Redis 更好上手，因为 MongoDB 是所有 NoSQL 中最像关系型数据库的。

MongoDB 基于分布式文件存储，功能非常强大，与关系型数据库的最大区别在于它的数据表结构非常灵活，是 BSON 格式的。BSON 是什么？它是一种类似于
JSON 的数据格式，可以存储各种不同的数据类型。

MongoDB 可以将数据结构完全不同的数据存储在同一张表中，也不需要预先定义数据表的结构，需要什么格式的数据直接存储就好，因此在灵活性上，MongoDB
是远远超过关系型数据库的，同时检索数据速度也更快。

使用 MongoDB，第一步搭建服务器，下面就教大家如何搭建一个简单的 MongoDB 服务器。

1\. 下载 MongoDB。

> <https://www.mongodb.com/download-center#community>

选择对应的操作系统版本进行下载。

![](https://images.gitbook.cn/8b569920-9ad5-11e8-831e-0180aea56660)

2\. 将下载完毕的文件进行解压。

![](https://images.gitbook.cn/9504a020-9ad5-11e8-8cbe-ad3f3badcc18)

3\. 创建 MongoDB 文件夹，在该目录中启动 MongoDB 服务，同时创建四个子文件夹：

  * data 保存数据文件
  * log 保存日志文件
  * bin 保存可执行文件
  * conf 保存配置文件

    
    
    mkdir mongodb
    cd mongodb
    mkdir data
    mkdir log
    mkdir conf
    mkdir bin
    

创建完成之后，查看 mongodb 目录下的所有文件，可以看到四个子文件夹创建成功。

![](https://images.gitbook.cn/a3b96d30-9ad5-11e8-a178-519d5b470954)

4\. 将解压文件中编译好的 mongod 文件复制到 bin 目录中。

    
    
    cp ../mongodb_home/mongodb-osx-x86_64-4.0.0/bin/mongod bin/
    

5\. 在 conf 目录中，编辑 mongod.conf 文件，配置启动选项。

    
    
    cd conf
    vim mongod.conf
    

port 指定自定义端口，dbpath 指定数据文件的存储路径，logpath 指定日志文件的存储路径，fork=true 表示启动后台进程。

    
    
    port = 12303
    dbpath = data
    logpath = log/mongod.log
    fork = true
    

![](https://images.gitbook.cn/b8e2e830-9ad5-11e8-831e-0180aea56660)

保存退出。

6\. 进入 mongodb 目录，运行 mongod 文件启动服务，指定配置文件 mongod.conf。

![](https://images.gitbook.cn/c5a3fb90-9ad5-11e8-8cbe-ad3f3badcc18)

终端显示 started successfully 表示服务启动成功。

7\. 使用 MongoDB 自带的客户端程序 mongo 来启动客户端访问数据库，与 mongod 一样，先将压缩文件中编译好的 mongo 文件复制到
mongodb/bin 目录中。

    
    
    cp ../mongodb_home/mongodb-osx-x86_64-4.0.0/bin/mongo bin/
    

8\. 进入 mongodb 目录，运行 mongo 文件启动客户端，我们没有设置用户名密码，只需要输入本机 IP 和 mongod.conf
中配置的端口即可。

    
    
    ./bin/mongo 127.0.0.1:12303/southwind
    

9\. 客户端启动成功，通过命令查看数据库信息。

查看所有数据库。

    
    
    show dbs
    

选择 local 数据库。

    
    
    use local
    

查看 local 数据库中所有表。

    
    
    show tables
    

![](https://images.gitbook.cn/dfdf8740-9ad5-11e8-831e-0180aea56660)

接下来就可以通过命令行对数据库进行操作。

10\. 关闭 MongoDB 服务。

    
    
    use admin
    db.shutdownServer()
    

![](https://images.gitbook.cn/ed01f480-9ad5-11e8-831e-0180aea56660)

如图显示，表示 MongoDB 服务器已经关闭。

11\. 退出客户端，control+C。

![](https://images.gitbook.cn/f5423ab0-9ad5-11e8-a178-519d5b470954)

### 总结

本节课我们讲解了 MongoDB 数据库的相关操作，包括基本概念、如何安装、如何启动数据库服务，以及常用的 MongoDB 命令完成对数据库的基本操作。

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

