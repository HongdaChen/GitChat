在某些极端情况下，软件为了能正常运行会做一些保护性的措施，比如运行内存超过最大值之后的处理，以及键值过期之后的处理等，都属于此类问题，而专业而全面的回答这些问题恰好是一个工程师所具备的优秀品质。

我们本文的面试题是 Redis 内存用完之后会怎么？

### 典型回答

Redis 的内存用完指的是 Redis 的运行内存超过了 Redis 设置的最大内存，此值可以通过 Redis 的配置文件 redis.conf
进行设置，设置项为 maxmemory，我们可以使用 `config get maxmemory` 来查看设置的最大运行内存，如下所示：

    
    
    127.0.0.1:6379> config get maxmemory
    1) "maxmemory"
    2) "0"
    

当此值为 0 时，表示没有内存大小限制，直到耗尽机器中所有的内存为止，这是 Redis 服务器端在 64 位操作系统下的默认值。

> 小贴士：32 位操作系统，默认最大内存值为 3GB。

当 Redis 的内存用完之后就会触发 Redis 的内存淘汰策略，执行流程如下图所示：
![image.png](https://images.gitbook.cn/2020-06-11-072027.png) 最大内存的检测源码位于
server.c 中，核心代码如下：

    
    
    int processCommand(client *c) {
        // 最大内存检测
        if (server.maxmemory && !server.lua_timedout) {
            int out_of_memory = freeMemoryIfNeededAndSafe() == C_ERR;
            if (server.current_client == NULL) return C_ERR;
            if (out_of_memory &&
                (c->cmd->flags & CMD_DENYOOM ||
                 (c->flags & CLIENT_MULTI && c->cmd->proc != execCommand))) {
                flagTransaction(c);
                addReply(c, shared.oomerr);
                return C_OK;
            }
        }
        // 忽略其他代码
    }
    

Redis 内存淘汰策略可以使用 `config get maxmemory-policy` 命令来查看，如下所示：

    
    
    127.0.0.1:6379> config get maxmemory-policy
    1) "maxmemory-policy"
    2) "noeviction"
    

从上述结果可以看出此 Redis 服务器采用的是 `noeviction`
策略，此策略表示当运行内存超过最大设置内存时，不淘汰任何数据，但新增操作会报错。此策略为 Redis 默认的内存淘汰策略，此值可通过修改
redis.conf 文件进行修改。

Redis
的内存最大值和内存淘汰策略都可以通过配置文件进行修改，或者是使用命令行工具进行修改。使用命令行工具进行修改的优点是操作简单，成功执行完命令之后设置的策略就会生效，我们可以使用
`confg set xxx` 的方式进行设置，但它的缺点是不能进行持久化，也就是当 Redis
服务器重启之后设置的策略就会丢失。另一种方式就是为配置文件修改的方式，此方式虽然较为麻烦，修改完之后要重启 Redis
服务器才能生效，但优点是可持久化，重启 Redis 服务器设置不会丢失。

### 考点分析

此面试题看似是一个普通问题，但却能考察面试者对于极端问题的关注与理解程度，这恰恰是一个优秀的工程师所具备的特有品质，当然这个问题的回答也可深可浅，如果读者在面试中恰好遇到了此问题，那么就应该乘胜追击把这个问题涉及到的更多知识点进行深入的阐述，以为自己的赢得更大的胜率，和此知识点相关的面试点还有以下这些：

  * Redis 内存淘汰策略有哪些？分别代表什么含义？
  * 内存淘汰策略采用了什么算法？

### 知识扩展

#### 内存淘汰策略

Redis 内存淘汰在 4.0 版本之后一共有 8 种：

  1. **noeviction** ：不淘汰任何数据，当内存不足时，新增操作会报错，Redis 默认内存淘汰策略；
  2. **allkeys-lru** ：淘汰整个键值中最久未使用的键值；
  3. **allkeys-random** ：随机淘汰任意键值;
  4. **volatile-lru** ：淘汰所有设置了过期时间的键值中最久未使用的键值；
  5. **volatile-random** ：随机淘汰设置了过期时间的任意键值；
  6. **volatile-ttl** ：优先淘汰更早过期的键值；
  7. **volatile-lfu** ：淘汰所有设置了过期时间的键值中，最少使用的键值；
  8. **allkeys-lfu** ：淘汰整个键值中最少使用的键值。

其中最后两条（7 和 8）为 4.0 版本新增的内存淘汰策略，从上述描述中可以看出，allkeys-xxx 表示从所有的键值中淘汰数据，而
volatile-xxx 表示从设置了过期键的键值中淘汰数据。

#### 内存淘汰算法

内存淘汰策略决定了内存淘汰算法，从以上八种内存淘汰策略可以看出，它们中虽然具体的实现细节不同，但主要的淘汰算法有两种：LRU 算法和 LFU 算法。

LRU 全称是 Least Recently Used 译为最近最少使用，是一种常用的页面置换算法，选择最近最久未使用的页面予以淘汰。

##### ① LRU 算法实现

LRU 算法需要基于链表结构，链表中的元素按照操作顺序从前往后排列，最新操作的键会被移动到表头，当需要内存淘汰时，只需要删除链表尾部的元素即可。

##### ② 近 LRU 算法

Redis 使用的是一种近似 LRU
算法，目的是为了更好的节约内存，它的实现方式是给现有的数据结构添加一个额外的字段，用于记录此键值的最后一次访问时间，Redis
内存淘汰时，会使用随机采样的方式来淘汰数据，它是随机取 5 个值 (此值可配置) ，然后淘汰最久没有使用的那个。

##### ③ LRU 算法缺点

LRU 算法有一个缺点，比如说很久没有使用的一个键值，如果最近被访问了一次，那么它就不会被淘汰，即使它是使用次数最少的缓存，那它也不会被淘汰，因此在
Redis 4.0 之后引入了 LFU 算法，下面我们一起来看。

LFU 全称是 Least Frequently Used
翻译为最不常用的，最不常用的算法是根据总访问次数来淘汰数据的，它的核心思想是“如果数据过去被访问多次，那么将来被访问的频率也更高”。 LFU
解决了偶尔被访问一次之后，数据就不会被淘汰的问题，相比于 LRU 算法也更合理一些。 在 Redis 中每个对象头中记录着 LFU 的信息，源码如下：

    
    
    typedef struct redisObject {
        unsigned type:4;
        unsigned encoding:4;
        unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                                * LFU data (least significant 8 bits frequency
                                * and most significant 16 bits access time). */
        int refcount;
        void *ptr;
    } robj;
    

在 Redis 中 LFU 存储分为两部分，16 bit 的 ldt(last decrement time) 和 8 bit 的
logc(logistic counter)。

  1. logc 是用来存储访问频次, 8 bit 能表示的最大整数值为 255，它的值越小表示使用频率越低，越容易淘汰；
  2. ldt 是用来存储上一次 logc 的更新时间。

至于 Redis 到底采用的是近 LRU 算法还是 LFU 算法，完全取决于内存淘汰策略的类型配置。

### 总结

本文我们讲了内存淘汰的 2 个重要参数 maxmemory（最大内存）和 maxmemory-policy（内存淘汰策略），当 Redis
的运行内存超过最大值之后会执行 Redis 的内存淘汰策略，从 Redis 4.0 之后 Redis 的内存淘汰策略总共有 8 种，而这 8
种策略中主要包含了 2 种算法，近 LRU 算法和 LFU 算法，希望本文对你理解 Redis 的内存机制有所帮助。

