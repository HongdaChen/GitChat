### 前言

上节课我们学习了 Spring Boot 整合 Spring Data JPA 的具体操作，Spring Data JPA
为关系型数据库提供了统一的操作模版，如果我们使用的是非关系数据库，则需要使用 Spring Data 家族中的其他子模块。

非关系型数据库（NoSQL）并不是“非 SQL”的意思，全称为 Not Only SQL，意为“不仅是
SQL”，非关系型数据库兴起的原因是什么？在互联网技术爆发式发展的今天，传统的关系型数据库已经无法应对结构复杂，规模超大的数据集合带来的挑战，比如社交网络服务，现实生活中群体的社交关系是错综复杂的，如果让你用关系型数据库来构建描述社交关系的数据表结构，将会是非常痛苦的，你可以自己试一下，为了解决这一问题，非关系数据库应运而生。

当下主流的非关系数据库为 Redis 和 MongoDB，Sprint Data 提供了 Spring Data Redis 和 Spring Data
MongoDB 来分别访问 Redis 和 MongoDB，毫无疑问，Spring Boot 也集成了 Spring Data Redis 和 Spring
Data MongoDB，我们先来学习 Spring Boot 与 Spring Data Redis 的整合。

在开始正式学习之前，我们先对 Reids 做一个简单的了解。

Redis 是一个主流的 NoSQL（非关系型）数据库，基于内存进行存储，支持 key-value 的存储形式，底层是用 C 编写的。Redis 相当于一个
key-value 的数据字典，结构非常简单，没有数据表的概念，直接用键值对形式完成数据的管理，Redis 支持的数据类型有 5
种，分别是字符串、列表、集合、有序集合、哈希，在定义 key 值的时候值既不要太长也不要太短，太长不利于检索，太短会降低可读性，尽量使用统一的命名规范。

接下来我们安装 Redis。

1\. 下载 Redis

到 Redis 的官网 https://redis.io/ 下载安装包，如下图所示，我下载的是 4.0.10 版本。

![](https://images.gitbook.cn/5856a180-c759-11e9-a81a-91f9bfe6443e)

2\. 下载完成之后将文件拷贝到 /usr/local 路径下。

3\. 在该路径下解压文件需要 root 权限，具体命令如下所示。

    
    
    sudo tar -zxf redis-4.0.10.tar.gz
    

4\. 通过终端进入 redis-4.0.10 目录，具体命令如下所示。

    
    
    cd redis-4.0.10
    

5\. 编译测试，具体命令如下所示。

    
    
    sudo make test
    

6\. 编译安装，具体命令如下所示。

    
    
    sudo make install
    

7\. 安装完成，接下来进行配置，在 /usr/local 路径下创建 redis 目录，同时创建三个子文件夹 bin、db、etc。

    
    
    mkdir redis
    cd redis
    mkdir bin
    mkdir db
    mkdir etc
    

8\. 把 redis-4.0.10/src 路径下的 mkreleasehdr.sh、redis-benchmark、redis-check-
dump、redis-cli、redis-server 拷贝到 redis/bin 目录下。

    
    
    cp ../redis-4.0.10/src/mkreleasehdr.sh bin/
    cp ../redis-4.0.10/src/redis-benchmark bin/
    cp ../redis-4.0.10/src/redis-check-dump bin/
    cp ../redis-4.0.10/src/redis-cli bin/
    cp ../redis-4.0.10/src/redis-server bin/
    

9\. 在 etc 路径下创建配置文件 redis.conf。

    
    
    cd etc
    vim redis.conf
    

10\. 编辑 redis.conf 文件，配置启动选项。

    
    
    #修改为守护模式
    
    daemonize yes
    
    #设置进程锁文件
    
    pidfile /usr/local/redis/redis.pid
    
    #端口
    
    port 6379
    
    #客户端超时时间
    
    timeout 300
    
    #日志级别
    
    loglevel debug
    
    #日志文件位置
    
    logfile /usr/local/redis/log-redis.log
    
    #设置数据库的数量，默认数据库为 0，可以使用 SELECT <dbid> 命令在连接上指定数据库 id
    
    databases 8
    
    ##指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合
    
    #save <seconds> <changes>
    
    #Redis 默认配置文件中提供了三个条件：
    
    save 900 1
    
    save 300 10
    
    save 60 10000
    
    #指定存储至本地数据库时是否压缩数据，默认为 yes，Redis 采用 LZF 压缩，如果为了节省 CPU 时间，可以关闭该选项，但会导致数据库文件变的巨大
    
    rdbcompression yes
    
    #指定本地数据库文件名
    
    dbfilename dump.rdb
    
    #指定本地数据库路径
    
    dir /usr/local/redis/db/
    
    #指定是否在每次更新操作后进行日志记录，Redis 在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。因为 Redis 本身同步数据文件是按上面 save 条件来同步的，所以有的数据会在一段时间内只存在于内存中
    
    appendonly no
    
    #指定更新日志条件，共有 3 个可选值：
    
    #no：表示等操作系统进行数据缓存同步到磁盘（快）
    #always：表示每次更新操作后手动调用 fsync() 将数据写到磁盘（慢，安全）
    #everysec：表示每秒同步一次（折衷，默认值）
    
    appendfsync everysec
    

11\. 保存退出，进入 etc 目录启动 redis 服务，具体命令如下所示。

    
    
    sudo ../bin/redis-server ./redis.conf
    

12\. 启动成功如下图所示。

![](https://images.gitbook.cn/2b516750-c75a-11e9-99c1-c37abd23c4b1)

13\. 进入 redis 目录启动 Redis 客户端，具体命令如下所示。

    
    
    ./bin/redis-cli
    

14\. 客户端启动成功之后如下图所示。

![](https://images.gitbook.cn/4107b900-c75a-11e9-a81a-91f9bfe6443e)

15\. 关闭 Redis 服务，在客户端执行如下命令。

    
    
    shutdown
    

16\. Redis 服务关闭之后如下图所示。

![](https://images.gitbook.cn/50b60140-c75a-11e9-9ae4-c3d609c8bfbd)

### 总结

本节课我们讲解了 NoSQL 数据库产品 Redis 的安装和使用，在前面的课程中我们已经讲解了另外一个主流 NoSQL 产品 MongoDB，相比于
MongoDB，Redis 是基于内存进行存储，支持 key-value 的存储形式，结构更加简单，没有数据表的概念，直接通过 key-value
的基本结构完成数据管理。

[点击这里获取 Spring Boot
视频专题](https://pan.baidu.com/s/1K2cNTk6JmZa50RYSKwvwGA)，提取码：e4wc

