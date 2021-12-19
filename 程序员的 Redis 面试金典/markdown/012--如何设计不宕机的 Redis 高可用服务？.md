随着业务的不断发展和扩张我们需要更加稳定和高效的 Redis 服务，这是业务发展的必然趋势也是个人能力进阶的最高境界，我们需要一个高可用的 Redis
服务，来支撑和保证业务的正常运行。

我们本文的面试题是，如何设计一个不宕机的 Redis 高可用服务？

### 典型回答

想要设计一个高可用的 Redis 服务，那么一定要从 Redis 的多机功能来考虑，比如 Redis 的主从、哨兵以及 Redis 集群服务。

主从同步 (主从复制) 是 Redis 高可用服务的基石，也是多机运行中最基础的一个，它是将从前的一台 Redis 服务器，变为一主多从的多台 Redis
服务器，这样我们就可以将 Redis 的读写分离，而这个 Redis 服务器也能承载更多的并发操作。

Redis Sentinel（哨兵模式）使用监控 Redis 主从服务器的，当 Redis 的主从服务器出现问题时，可以利用哨兵模式自动的实现容灾恢复。

Redis Cluster（集群）是 Redis 3.0 版本中推出的 Redis
集群方案，它是将数据分布在不同的服务器上，以此来降低系统对单主节点的依赖，并且可以大大的提高 Redis 服务的读写性能。Redis Cluster
拥有所有主从同步和哨兵的所有优点，并且可以实现多主多从的集群服务，相当于将单台 Redis 服务器的性能平行扩展到了集群中，并且它还有自动容灾恢复的功能。

### 考点分析

Redis 多机知识是应聘中级和高级必问的知识点，它虽然看起来很高大上，其实它的概念却很好理解，并且 Redis 也提供了方便的多机构建的方案，例如
Redis 只需要一个命令就可以迅速的构建出一个集群服务等。

和此知识点相关的面试题还有以下这些：

  * Redis 主从同步如何开启？它的数据同步方式有几种？
  * Redis 哨兵模式如何开启？它是如何选择主服务器的？
  * Redis 集群是如何创建的？说一说它的故障处理？

### 知识扩展

#### 1.主从同步

我们可以将多台单独启动的 Redis
服务器设置为一个主从同步服务器，它的设置方式有两种，一种是在运行时将自己设置为目标服务器的从服务器；另一种是启动时将自己设置为目标服务器的从服务器。

##### 运行中设置从服务器

在 Redis 运行过程中，我们可以使用 `replicaof host port` 命令，把自己设置为目标 IP 的从服务器，执行命令如下：

    
    
    127.0.0.1:6379> replicaof 127.0.0.1 6380
    OK
    

如果主服务设置了密码，需要在从服务器输入主服务器的密码，使用 `config set masterauth 主服务密码` 命令的方式，例如：

    
    
    127.0.0.1:6377> config set masterauth pwd654321
    OK
    

###### ① 执行流程

在执行完 `replicaof` 命令之后，从服务器的数据会被清空，主服务会把它的数据副本同步给从服务器。

###### ② 测试同步功能

主从服务器设置完同步之后，我们来测试一下主从数据同步，首先我们先在主服务器上执行保存数据操作，再去从服务器查询。

主服务器执行命令：

    
    
    127.0.0.1:6379> set lang redis
    OK
    

从服务执行查询：

    
    
    127.0.0.1:6379> get lang
    "redis"
    

可以看出数据已经被正常同步过来了。

##### 启动时设置从服务器

我们可以使用命令 `redis-server --port 6380 --replicaof 127.0.0.1 6379`
将自己设置成目标服务器的从服务器。

主从同步的数据同步方式有以下三种。

###### ① 完整数据同步

当有新的从服务器连接时，为了保障多个数据库的一致性，主服务器会执行一次 `bgsave` 命令生成一个 RDB 文件，然后再以 Socket
的方式发送给从服务器，从服务器收到 RDB 文件之后再把所有的数据加载到自己的程序中，就完成了一次全量的数据同步。

###### ② 部分数据同步

在 Redis 2.8
之前每次从服务器离线再重新上线之前，主服务器会进行一次完整的数据同步，然后这种情况如果发生在离线时间比较短的情况下，只有少量的数据不同步却要同步所有的数据是非常笨拙和不划算的，在
Redis 2.8 这个功能得到了优化。 Redis 2.8
的优化方法是当从服务离线之后，主服务器会把离线之后的写入命令存储在一个特定大小的队列中，队列是可以保证先进先出的执行顺序的，当从服务器重写恢复上线之后，主服务会判断离线这段时间内的命令是否还在队列中，如果在就直接把队列中的数据发送给从服务器，这样就避免了完整同步的资源浪费。

> 小贴士：存储离线命令的队列大小默认是 1MB，使用者可以自行修改队列大小的配置项 repl-backlog-size。

###### ③ 无盘数据同步

从前面的内容我们可以得知，在第一次主从连接的时候，会先产生一个 RDB 文件，再把 RDB 文件发送给从服务器，如果主服务器是非固态硬盘的时候，系统的
I/O 操作是非常高的，为了缓解这个问题，Redis 2.8.18 新增了无盘复制功能，无盘复制功能不会在本地创建 RDB
文件，而是会派生出一个子进程，然后由子进程通过 Socket 的方式，直接将 RDB 文件写入到从服务器，这样主服务器就可以在不创建 RDB
文件的情况下，完成与从服务器的数据同步。

要使用无须复制功能，只需把配置项 repl-diskless-sync 的值设置为 yes 即可，它默认配置值为 no。

#### 2.哨兵模式

使用主从同步的一个致命问题就是出现服务器宕机之后需要手动恢复，而使用哨兵模式之后就可以监控这些主从服务器并且提供自动容灾恢复的功能，哨兵模式构建流程如下图所示：
![哨兵模式.png](https://images.gitbook.cn/2020-06-30-035113.png)

> 小贴士：Redis Sentinel 的最小分配单位是一主一从。

Redis 哨兵功能保存在 src 目录下，如图所示：
![image.png](https://images.gitbook.cn/2020-06-30-35114.png) 我们需要使用命令
`./src/redis-sentinel sentinel.conf` 来启动 Sentinel，可以看出我们在启动它时必须设置一个
sentinel.conf 文件，这个配置文件中必须包含监听的主节点信息 `sentinel monitor master-name ip port
quorum` 例如 `sentinel monitor mymaster 127.0.0.1 6379 1` 其中：

  * master-name 表示给监视的主节点起一个名称；
  * ip 表示主节点的 IP；
  * port 表示主节点的端口；
  * quorum 表示确认主节点下线的 Sentinel 数量，如果 quorum 设置为 1 表示只要有一台 Sentinel 判断它下线了，就可以确认它真的下线了。

注意： **如果主节点 Redis 服务器有密码，还必须在 sentinel.conf 中添加主节点的密码，不然会导致 Sentinel
不能自动监听到主节点下面的从节点** 。 所以如果 Redis 有密码，sentinel.conf 必须包含以下内容：

> sentinel monitor mymaster 127.0.0.1 6379 1 sentinel auth-pass mymaster
> pwd654321

当我们配置好 sentinel.conf 并执行启动命令 `./src/redis-sentinel sentinel.conf` 之后，Redis
Sentinel 就会被启动，如下图所示：
![image.png](https://images.gitbook.cn/2020-06-30-035116.png) 从上图可以看出 Sentinel
只需配置监听主节点的信息，它会自动监听对应的从节点。

上面我们演示了单个 Sentinel 的启动，但生产环境我们不会只启动一台 Sentinel，因为如果启动一台 Sentinel
假如它不幸宕机的话，就不能提供自动容灾的服务了，不符合我们高可用的宗旨，所以我们会在不同的物理机上启动多个 Sentinel 来组成 Sentinel
集群，来保证 Redis 服务的高可用。

启动 Sentinel 集群的方法很简单，和上面启动单台的方式一样，我们只需要把多个 Sentinel 监听到一个主服务器节点，那么多个 Sentinel
就会自动发现彼此，并组成一个 Sentinel 集群。

我们启动第二个 Sentinel 来试一下，执行结果如下：

    
    
    [@iZ2ze0nc5n41zomzyqtksmZ:redis2]$ ./src/redis-sentinel sentinel.conf
    5547:X 19 Feb 2020 20:29:30.047 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
    5547:X 19 Feb 2020 20:29:30.047 # Redis version=5.0.5, bits=64, commit=00000000, modified=0, pid=5547, just started
    5547:X 19 Feb 2020 20:29:30.047 # Configuration loaded
                    _._                                                  
               _.-``__ ''-._                                             
          _.-``    `.  `_.  ''-._           Redis 5.0.5 (00000000/0) 64 bit
      .-`` .-```.  ```\/    _.,_ ''-._                                   
     (    '      ,       .-`  | `,    )     Running in sentinel mode
     |`-._`-...-` __...-.``-._|'` _.-'|     Port: 26377
     |    `-._   `._    /     _.-'    |     PID: 5547
      `-._    `-._  `-./  _.-'    _.-'                                   
     |`-._`-._    `-.__.-'    _.-'_.-'|                                  
     |    `-._`-._        _.-'_.-'    |           http://redis.io        
      `-._    `-._`-.__.-'_.-'    _.-'                                   
     |`-._`-._    `-.__.-'    _.-'_.-'|                                  
     |    `-._`-._        _.-'_.-'    |                                  
      `-._    `-._`-.__.-'_.-'    _.-'                                   
          `-._    `-.__.-'    _.-'                                       
              `-._        _.-'                                           
                  `-.__.-'                                               
    
    5547:X 19 Feb 2020 20:29:30.049 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
    5547:X 19 Feb 2020 20:29:30.049 # Sentinel ID is 6455f2f74614a71ce0a63398b2e48d6cd1cf0d06
    5547:X 19 Feb 2020 20:29:30.049 # +monitor master mymaster 127.0.0.1 6379 quorum 1
    5547:X 19 Feb 2020 20:29:30.049 * +slave slave 127.0.0.1:6377 127.0.0.1 6377 @ mymaster 127.0.0.1 6379
    5547:X 19 Feb 2020 20:29:30.052 * +slave slave 127.0.0.1:6378 127.0.0.1 6378 @ mymaster 127.0.0.1 6379
    5547:X 19 Feb 2020 20:29:30.345 * +sentinel sentinel 6455f2f74614a71ce0a63398b2e48d6cd1cf0d08 127.0.0.1 26379 @ mymaster 127.0.0.1 6379
    

从以上启动命令可以看出，比单机模式多了最后一行发现其他 Sentinel 服务器的命令，说明这两个 Sentinel 已经组成一个集群了。 Sentinel
集群示意图如下： ![哨兵模式-多哨兵.png](https://images.gitbook.cn/2020-06-30-035118.png)

一般情况下 Sentinel 集群的数量取大于 1 的奇数，例如 3、5、7、9，而 quorum 的配置要根据 Sentinel 的数量来发生变化，例如
Sentinel 是 3 台，那么对应的 quorum 最好是 2，如果 Sentinel 是 5 台，那么 quorum 最好是 3，它表示当有 3 台
Sentinel 都确认主节点下线了，就可以确定主节点真的下线了。

与 quorum 参数相关的有两个概念：主观下线和客观下线。

当 Sentinel 集群中，有一个 Sentinel 认为主服务器已经下线时，它会将这个主服务器标记为主观下线 (Subjectively Down,
SDOWN)，然后询问集群中的其他 Sentinel，是否也认为该服务器已下线，当同意主服务器已下线的 Sentinel 数量达到 quorum
参数所指定的数量时，Sentinel 就会将相应的主服务器标记为客观下线 (Objectively down, ODOWN)，然后开始对其进行故障转移。

上面我们模拟了 Redis Sentinel 自动容灾恢复，那接下来我们来看一下，主服务器竞选的规则和相关设置项。

##### 新主节点竞选优先级设置

我们可以 redis.conf 中的 `replica-priority` 选项来设置竞选新主节点的优先级，它的默认值是 100，它的最大值也是
100，这个值越小它的权重就越高，例如从节点 A 的 `replica-priority` 值为 100，从节点 B 的值为 50，从节点 C 的值为
5，那么在竞选时从节点 C 会作为新的主节点。

##### 新主节点竞选规则

新主节点的竞选会排除不符合条件的从节点，然后在剩余的从节点按照优先级来挑选。首先来说，存在以下条件的从节点会被排除：

  1. 排除所有已经下线以及长时间没有回复心跳检测的疑似已下线从服务器；
  2. 排除所有长时间没有与主服务器通信，数据状态过时的从服务器；
  3. 排除所有优先级 (replica-priority) 为 0 的服务器。

符合条件的从节点竞选顺序：

  1. 优先级最高的从节点将会作为新主节点；
  2. 优先级相等则判断复制偏移量，偏移量最大的从节点获胜；
  3. 如果以上两个条件都相同，选择 Redis 运行时随机生成 ID 最小那个为新的主服务器。

#### 3.Redis 集群

Redis Cluster
是无代理模式去中心化的运行模式，客户端发送的绝大数命令会直接交给相关节点执行，这样大部分情况请求命令无需转发，或仅转发一次的情况下就能完成请求与响应，所以集群单个节点的性能与单机
Redis 服务器的性能是非常接近的，因此在理论情况下，当水平扩展一倍的主节点就相当于请求处理的性能也提高了一倍，所以 Redis Cluster
的性能是非常高的。

Redis Cluster 架构图如下所示：
![image.png](https://images.gitbook.cn/2020-06-30-035119.png) Redis Cluster
的搭建方式有两种，一种是使用 Redis 源码中提供的 create-cluster 工具快速的搭建 Redis 集群环境，另一种是配置文件的方式手动创建
Redis 集群环境。

##### ① 快速搭建 Redis Cluster

create-cluster 工具在 utils/create-cluster 目录下，如下图所示：
![image.png](https://images.gitbook.cn/2020-06-30-035120.png) 使用命令 `./create-
cluster start` 就可以急速创建一个 Redis 集群，执行如下：

    
    
    $ ./create-cluster start # 创建集群
    Starting 30001
    Starting 30002
    Starting 30003
    Starting 30004
    Starting 30005
    Starting 30006
    

接下来我们需要把以上创建的 6 个节点通过 `create` 命令组成一个集群，执行如下：

    
    
    [@iZ2ze0nc5n41zomzyqtksmZ:create-cluster]$ ./create-cluster create # 组建集群
    >>> Performing hash slots allocation on 6 nodes...
    Master[0] -> Slots 0 - 5460
    Master[1] -> Slots 5461 - 10922
    Master[2] -> Slots 10923 - 16383
    Adding replica 127.0.0.1:30005 to 127.0.0.1:30001
    Adding replica 127.0.0.1:30006 to 127.0.0.1:30002
    Adding replica 127.0.0.1:30004 to 127.0.0.1:30003
    >>> Trying to optimize slaves allocation for anti-affinity
    [WARNING] Some slaves are in the same host as their master
    M: 445f2a86fe36d397613839d8cc1ae6702c976593 127.0.0.1:30001
       slots:[0-5460] (5461 slots) master
    M: 63bb14023c0bf58926738cbf857ea304bff8eb50 127.0.0.1:30002
       slots:[5461-10922] (5462 slots) master
    M: 864d4dfe32e3e0b81a64cec8b393bbd26a65cbcc 127.0.0.1:30003
       slots:[10923-16383] (5461 slots) master
    S: 64828ab44566fc5ad656e831fd33de87be1387a0 127.0.0.1:30004
       replicates 445f2a86fe36d397613839d8cc1ae6702c976593
    S: 0b17b00542706343583aa73149ec5ff63419f140 127.0.0.1:30005
       replicates 63bb14023c0bf58926738cbf857ea304bff8eb50
    S: e35f06ca9b700073472d72001a39ea4dfcb541cd 127.0.0.1:30006
       replicates 864d4dfe32e3e0b81a64cec8b393bbd26a65cbcc
    Can I set the above configuration? (type 'yes' to accept): yes
    >>> Nodes configuration updated
    >>> Assign a different config epoch to each node
    >>> Sending CLUSTER MEET messages to join the cluster
    Waiting for the cluster to join
    .
    >>> Performing Cluster Check (using node 127.0.0.1:30001)
    M: 445f2a86fe36d397613839d8cc1ae6702c976593 127.0.0.1:30001
       slots:[0-5460] (5461 slots) master
       1 additional replica(s)
    M: 864d4dfe32e3e0b81a64cec8b393bbd26a65cbcc 127.0.0.1:30003
       slots:[10923-16383] (5461 slots) master
       1 additional replica(s)
    S: e35f06ca9b700073472d72001a39ea4dfcb541cd 127.0.0.1:30006
       slots: (0 slots) slave
       replicates 864d4dfe32e3e0b81a64cec8b393bbd26a65cbcc
    S: 0b17b00542706343583aa73149ec5ff63419f140 127.0.0.1:30005
       slots: (0 slots) slave
       replicates 63bb14023c0bf58926738cbf857ea304bff8eb50
    M: 63bb14023c0bf58926738cbf857ea304bff8eb50 127.0.0.1:30002
       slots:[5461-10922] (5462 slots) master
       1 additional replica(s)
    S: 64828ab44566fc5ad656e831fd33de87be1387a0 127.0.0.1:30004
       slots: (0 slots) slave
       replicates 445f2a86fe36d397613839d8cc1ae6702c976593
    [OK] All nodes agree about slots configuration.
    >>> Check for open slots...
    >>> Check slots coverage...
    [OK] All 16384 slots covered.
    

在执行的过程中会询问你是否通过把 30001、30002、30003 作为主节点，把 30004、30005、30006 作为它们的从节点，输入 `yes`
后会执行完成。

我们可以先使用 redis-cli 连接到集群，命令如下：

    
    
    $ redis-cli -c -p 30001
    

在使用 nodes 命令来查看集群的节点信息，命令如下：

    
    
    127.0.0.1:30001> cluster nodes
    864d4dfe32e3e0b81a64cec8b393bbd26a65cbcc 127.0.0.1:30003@40003 master - 0 1585125835078 3 connected 10923-16383
    e35f06ca9b700073472d72001a39ea4dfcb541cd 127.0.0.1:30006@40006 slave 864d4dfe32e3e0b81a64cec8b393bbd26a65cbcc 0 1585125835078 6 connected
    0b17b00542706343583aa73149ec5ff63419f140 127.0.0.1:30005@40005 slave 63bb14023c0bf58926738cbf857ea304bff8eb50 0 1585125835078 5 connected
    63bb14023c0bf58926738cbf857ea304bff8eb50 127.0.0.1:30002@40002 master - 0 1585125834175 2 connected 5461-10922
    445f2a86fe36d397613839d8cc1ae6702c976593 127.0.0.1:30001@40001 myself,master - 0 1585125835000 1 connected 0-5460
    64828ab44566fc5ad656e831fd33de87be1387a0 127.0.0.1:30004@40004 slave 445f2a86fe36d397613839d8cc1ae6702c976593 0 1585125835000 4 connected
    

可以看出 30001、30002、30003 都为主节点，30001 对应的槽位是 0-5460，30002 对应的槽位是 5461-10922，30003
对应的槽位是 10923-16383，总共有槽位 16384 个 (0 ~ 16383)。

**create-cluster
搭建的方式虽然速度很快，但是该方式搭建的集群主从节点数量固定以及槽位分配模式固定，并且安装在同一台服务器上，所以只能用于测试环境。**

我们测试完成之后，可以 **使用以下命令，关闭并清理集群** ：

    
    
    $ ./create-cluster stop # 关闭集群
    Stopping 30001
    Stopping 30002
    Stopping 30003
    Stopping 30004
    Stopping 30005
    Stopping 30006
    $ ./create-cluster clean # 清理集群
    

## ### ② 手动搭建 Redis Cluster

由于 create-cluster 本身的限制，在实际生产环境中我们需要使用手动添加配置的方式搭建 Redis 集群，为此我们先要把 Redis
安装包复制到 node1 到 node6 文件中，因为我们要安装 6 个节点，3 主 3 从，如下图所示：
![image.png](https://images.gitbook.cn/2020-06-30-035122.png)
![image.png](https://images.gitbook.cn/2020-06-30-035123.png) 接下来我们进行配置并启动
Redis 集群。

###### 设置配置文件

我们需要修改每个节点内的 redis.conf 文件，设置 `cluster-enabled yes` 表示开启集群模式，并且修改各自的端口，我们继续使用
30001 到 30006，通过 `port 3000X` 设置。

###### 启动各个节点

redis.conf 配置好之后，我们就可以启动所有的节点了，命令如下：

    
    
    cd /usr/local/soft/mycluster/node1 
    ./src/redis-server redis.conf
    

###### 创建集群并分配槽位

之前我们已经启动了 6 个节点，但这些节点都在各自的集群之内并未互联互通，因此接下来我们需要把这些节点串连成一个集群，并为它们指定对应的槽位，执行命令如下：

    
    
    redis-cli --cluster create 127.0.0.1:30001 127.0.0.1:30002 127.0.0.1:30003 127.0.0.1:30004 127.0.0.1:30005 127.0.0.1:30006 --cluster-replicas 1
    

其中 create 后面跟多个节点，表示把这些节点作为整个集群的节点，而 cluster-replicas 表示给集群中的主节点指定从节点的数量，1
表示为每个主节点设置一个从节点。

在执行了 create 命令之后，系统会为我们指定节点的角色和槽位分配计划，如下所示：

    
    
    >>> Performing hash slots allocation on 6 nodes...
    Master[0] -> Slots 0 - 5460
    Master[1] -> Slots 5461 - 10922
    Master[2] -> Slots 10923 - 16383
    Adding replica 127.0.0.1:30005 to 127.0.0.1:30001
    Adding replica 127.0.0.1:30006 to 127.0.0.1:30002
    Adding replica 127.0.0.1:30004 to 127.0.0.1:30003
    >>> Trying to optimize slaves allocation for anti-affinity
    [WARNING] Some slaves are in the same host as their master
    M: bdd1c913f87eacbdfeabc71befd0d06c913c891c 127.0.0.1:30001
       slots:[0-5460] (5461 slots) master
    M: bdd1c913f87eacbdfeabc71befd0d06c913c891c 127.0.0.1:30002
       slots:[5461-10922] (5462 slots) master
    M: bdd1c913f87eacbdfeabc71befd0d06c913c891c 127.0.0.1:30003
       slots:[10923-16383] (5461 slots) master
    S: bdd1c913f87eacbdfeabc71befd0d06c913c891c 127.0.0.1:30004
       replicates bdd1c913f87eacbdfeabc71befd0d06c913c891c
    S: bdd1c913f87eacbdfeabc71befd0d06c913c891c 127.0.0.1:30005
       replicates bdd1c913f87eacbdfeabc71befd0d06c913c891c
    S: bdd1c913f87eacbdfeabc71befd0d06c913c891c 127.0.0.1:30006
       replicates bdd1c913f87eacbdfeabc71befd0d06c913c891c
    Can I set the above configuration? (type 'yes' to accept): 
    

从以上信息可以看出，Redis 打算把 30001、30002、30003 设置为主节点，并为它们分配槽位，30001 对应的槽位是
0-5460，30002 对应的槽位是 5461-10922，30003 对应的槽位是 10923-16383，并且把 30005 设置为 30001
的从节点、30006 设置为 30002 的从节点、30004 设置为 30003 的从节点，我们只需要输入 `yes` 即可确认并执行分配，如下所示：

    
    
    Can I set the above configuration? (type 'yes' to accept): yes
    >>> Nodes configuration updated
    >>> Assign a different config epoch to each node
    >>> Sending CLUSTER MEET messages to join the cluster
    Waiting for the cluster to join
    ....
    >>> Performing Cluster Check (using node 127.0.0.1:30001)
    M: 887397e6fefe8ad19ea7569e99f5eb8a803e3785 127.0.0.1:30001
       slots:[0-5460] (5461 slots) master
       1 additional replica(s)
    S: abec9f98f9c01208ba77346959bc35e8e274b6a3 127.0.0.1:30005
       slots: (0 slots) slave
       replicates 887397e6fefe8ad19ea7569e99f5eb8a803e3785
    S: 1a324d828430f61be6eaca7eb2a90728dd5049de 127.0.0.1:30004
       slots: (0 slots) slave
       replicates f5958382af41d4e1f5b0217c1413fe19f390b55f
    S: dc0702625743c48c75ea935c87813c4060547cef 127.0.0.1:30006
       slots: (0 slots) slave
       replicates 3da35c40c43b457a113b539259f17e7ed616d13d
    M: 3da35c40c43b457a113b539259f17e7ed616d13d 127.0.0.1:30002
       slots:[5461-10922] (5462 slots) master
       1 additional replica(s)
    M: f5958382af41d4e1f5b0217c1413fe19f390b55f 127.0.0.1:30003
       slots:[10923-16383] (5461 slots) master
       1 additional replica(s)
    [OK] All nodes agree about slots configuration.
    >>> Check for open slots...
    >>> Check slots coverage...
    [OK] All 16384 slots covered.
    

显示 OK 表示整个集群就已经成功启动了。

接下来，我们使用 redis-cli 连接并测试一下集群的运行状态，代码如下：

    
    
    $ redis-cli -c -p 30001 # 连接到集群
    127.0.0.1:30001> cluster info # 查看集群信息
    cluster_state:ok # 状态正常
    cluster_slots_assigned:16384 # 槽位数
    cluster_slots_ok:16384 # 正常的槽位数
    cluster_slots_pfail:0 
    cluster_slots_fail:0
    cluster_known_nodes:6 # 集群的节点数
    cluster_size:3 # 集群主节点数
    cluster_current_epoch:6
    cluster_my_epoch:1
    cluster_stats_messages_ping_sent:130
    cluster_stats_messages_pong_sent:127
    cluster_stats_messages_sent:257
    cluster_stats_messages_ping_received:122
    cluster_stats_messages_pong_received:130
    cluster_stats_messages_meet_received:5
    cluster_stats_messages_received:257
    

相关字段的说明已经标识在上述的代码中了，这里就不再赘述。

##### 故障发现

故障发现里面有两个重要的概念：疑似下线 (PFAIL-Possibly Fail) 和确定下线 (Fail)。

集群中的健康监测是通过定期向集群中的其他节点发送 PING 信息来确认的，如果发送 PING 消息的节点在规定时间内，没有收到返回的 PONG
消息，那么对方节点就会被标记为疑似下线。

一个节点发现某个节点疑似下线，它会将这条信息向整个集群广播，其它节点就会收到这个消息，并且通过 PING
的方式监测某节点是否真的下线了。如果一个节点收到某个节点疑似下线的数量超过集群数量的一半以上，就可以标记该节点为确定下线状态，然后向整个集群广播，强迫其它节点也接收该节点已经下线的事实，并立即对该失联节点进行主从切换。

这就是疑似下线和确认下线的概念，这个概念和哨兵模式里面的主观下线和客观下线的概念比较类似。

##### 故障转移

当一个节点被集群标识为确认下线之后就可以执行故障转移了，故障转移的执行流程如下：

  1. 从下线的主节点的所有从节点中，选择一个从节点 (选择的方法详见下面“新主节点选举原则”部分)；
  2. 从节点会执行 SLAVEOF NO ONE 命令，关闭这个从节点的复制功能，并从从节点转变回主节点，原来同步所得的数据集不会被丢弃；
  3. 新的主节点会撤销所有对已下线主节点的槽指派，并将这些槽全部指派给自己；
  4. 新的主节点向集群广播一条 PONG 消息，这条 PONG 消息是让集群中的其他节点知道此节点已经由从节点变成了主节点，并且这个主节点已经接管了原本由已下线节点负责处理的槽位信息；
  5. 新的主节点开始处理相关的命令请求，此故障转移过程完成。

##### 新主节点选举原则

新主节点选举的方法是这样的：

  1. 集群的纪元 (epoch) 是一个自增计数器，初始值为 0；
  2. 而每个主节点都有一次投票的机会，主节点会把这一票投给第一个要求投票的从节点；
  3. 当从节点发现自己正在复制的主节点确认下线之后，就会向集群广播一条消息，要求所有有投票权的主节点给此从节点投票；
  4. 如果有投票权的主节点还没有给其他人投票的情况下，它会向第一个要求投票的从节点发送一条消息，表示把这一票投给这个从节点；
  5. 当从节点收到投票数量大于集群数量的半数以上时，这个从节点就会当选为新的主节点。

### 总结

本文我们讲了 Redis 高可用的三种实现手段：主从同步、哨兵模式和 Redis 集群服务，其中主从同步是 Redis
高可用服务的基石，也是多机运行中最基础的一个，但它有一个重要的问题当发生故障时，我们需要手动恢复故障，因此就有了哨兵模式用于监控和实现主从服务器的自动容灾，最后我们讲了高可用最常用也最终极的方法
Redis 集群。

