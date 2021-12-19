上一篇我们讲了 Redis 内存用完之后的内存淘汰策略，它主要是用来出来异常情况下的数据清理，而本文讲的是 Redis
的键值过期之后的数据处理，讲的是正常情况下的数据清理，但面试者常常会把两个概念搞混，以至于和期望的工作失之交臂。我们本文的职责之一就是帮读者朋友搞清楚二者的区别，相信看完本文你就会对二者的概念有一个本质上的认识。

我们本文的面试题是，Redis 如何处理已过期的数据？

### 典型回答

在 Redis 中维护了一个过期字典，会将所有已经设置了过期时间的键值全部存储到此字典中，例如我们使用设置过期时间的命令时，命令如下：

    
    
    127.0.0.1:6379> set mykey java ex 5
    OK
    

此命令表示 5s 之后键值为 `mykey:java` 的数据将会过期，其中 `ex` 是 `expire` 的缩写，也就是过期、到期的意思。

过期时间除了上面的那种字符类型的直接设置之外，还可以使用 `expire key seconds` 的方式直接设置，示例如下：

    
    
    127.0.0.1:6379> set key value
    OK
    127.0.0.1:6379> expire key 100 # 设置 100s 后过期
    (integer) 1
    

获取键值的执行流程是，当有键值的访问请求时 Redis
会先判断此键值是否在过期字典中，如果没有表示键值没有设置过期时间（永不过期），然后就可以正常返回键值数据了；如果此键值在过期字典中则会判断当前时间是否小于过期时间，如果小于则说明此键值没有过期可以正常返回数据，反之则表示数据已过期，会删除此键值并且返回给客户端
`nil`，执行流程如下图所示： ![image.png](https://images.gitbook.cn/2020-06-11-072044.png)
这是键值数据的方法流程，同时也是过期键值的判断和删除的流程。

### 考点分析

本文的面试题考察的是你对 Redis 的过期删除策略的掌握，在 Redis 中为了平衡空间占用和 Redis
的执行效率，采用了两种删除策略，上面的回答不完全对，因为他只回答出了一种过期键的删除策略，和此知识点相关的面试题还有以下这些：

  * 常用的删除策略有哪些？Redis 使用了什么删除策略？
  * Redis 中是如何存储过期键的？

### 知识扩展

#### 删除策略

常见的过期策略，有以下三种：

  * 定时删除
  * 惰性删除
  * 定期删除

##### 1.定时删除

在设置键值过期时间时，创建一个定时事件，当过期时间到达时，由事件处理器自动执行键的删除操作。

##### ① 优点

保证内存可以被尽快的释放

##### ② 缺点

在 Redis 高负载的情况下或有大量过期键需要同时处理时，会造成 Redis 服务器卡顿，影响主业务执行。

#### 2.惰性删除

不主动删除过期键，每次从数据库获取键值时判断是否过期，如果过期则删除键值，并返回 null。

##### ① 优点

因为每次访问时，才会判断过期键，所以此策略只会使用很少的系统资源。

##### ② 缺点

系统占用空间删除不及时，导致空间利用率降低，造成了一定的空间浪费。

##### ③ 源码解析

Redis 中惰性删除的源码位于 src/db.c 文件的 expireIfNeeded 方法中，源码如下：

    
    
    int expireIfNeeded(redisDb *db, robj *key) {
        // 判断键是否过期
        if (!keyIsExpired(db,key)) return 0;
        if (server.masterhost != NULL) return 1;
        /* 删除过期键 */
        // 增加过期键个数
        server.stat_expiredkeys++;
        // 传播键过期的消息
        propagateExpire(db,key,server.lazyfree_lazy_expire);
        notifyKeyspaceEvent(NOTIFY_EXPIRED,
            "expired",key,db->id);
        // server.lazyfree_lazy_expire 为 1 表示异步删除（懒空间释放），反之同步删除
        return server.lazyfree_lazy_expire ? dbAsyncDelete(db,key) :
                                             dbSyncDelete(db,key);
    }
    // 判断键是否过期
    int keyIsExpired(redisDb *db, robj *key) {
        mstime_t when = getExpire(db,key);
        if (when < 0) return 0; /* No expire for this key */
        /* Don't expire anything while loading. It will be done later. */
        if (server.loading) return 0;
        mstime_t now = server.lua_caller ? server.lua_time_start : mstime();
        return now > when;
    }
    // 获取键的过期时间
    long long getExpire(redisDb *db, robj *key) {
        dictEntry *de;
        /* No expire? return ASAP */
        if (dictSize(db->expires) == 0 ||
           (de = dictFind(db->expires,key->ptr)) == NULL) return -1;
        /* The entry was found in the expire dict, this means it should also
         * be present in the main dict (safety check). */
        serverAssertWithInfo(NULL,key,dictFind(db->dict,key->ptr) != NULL);
        return dictGetSignedIntegerVal(de);
    }
    

所有对数据库的读写命令在执行之前，都会调用 expireIfNeeded 方法判断键值是否过期，过期则会从数据库中删除，反之则不做任何处理。

惰性删除执行流程，如下图所示： ![内存过期策略-
惰性删除执行流程.png](https://images.gitbook.cn/2020-06-11-072046.png)

#### 3.定期删除

每隔一段时间检查一次数据库，随机删除一些过期键。 Redis 默认每秒进行 10 次过期扫描，此配置可通过 Redis 的配置文件 redis.conf
进行配置，配置键为 `hz` 它的默认值是 `hz 10` 。 需要注意的是：Redis
每次扫描并不是遍历过期字典中的所有键，而是采用随机抽取判断并删除过期键的形式执行的。

##### ① 定期删除流程

  1. 从过期字典中随机取出 20 个键；
  2. 删除这 20 个键中过期的键；
  3. 如果过期 key 的比例超过 25% ，重复步骤 1。

同时为了保证过期扫描不会出现循环过度，导致线程卡死现象，算法还增加了扫描时间的上限，默认不会超过 25ms。

定期删除执行流程，如下图所示： ![内存过期策略-执行流程
2.png](https://images.gitbook.cn/2020-06-11-072047.png)

##### ② 优点

通过限制删除操作的时长和频率，来减少删除操作对 Redis 主业务的影响，同时也能删除一部分过期的数据减少了过期键对空间的无效占用。

##### ③ 缺点

内存清理方面没有定时删除效果好，同时没有惰性删除使用的系统资源少。

##### ④ 源码解析

在 Redis 中定期删除的核心源码在 src/expire.c 文件下的 activeExpireCycle 方法中，源码如下：

    
    
    void activeExpireCycle(int type) {
        static unsigned int current_db = 0; /* 上次定期删除遍历到的数据库 ID */
        static int timelimit_exit = 0;      /* Time limit hit in previous call? */
        static long long last_fast_cycle = 0; /* 上一次执行快速定期删除的时间点 */
        int j, iteration = 0;
        int dbs_per_call = CRON_DBS_PER_CALL; // 每次定期删除，遍历的数据库的数量
        long long start = ustime(), timelimit, elapsed;
        if (clientsArePaused()) return;
        if (type == ACTIVE_EXPIRE_CYCLE_FAST) {
            if (!timelimit_exit) return;
            // ACTIVE_EXPIRE_CYCLE_FAST_DURATION 是快速定期删除的执行时长
            if (start < last_fast_cycle + ACTIVE_EXPIRE_CYCLE_FAST_DURATION*2) return;
            last_fast_cycle = start;
        }
        if (dbs_per_call > server.dbnum || timelimit_exit)
            dbs_per_call = server.dbnum;
        // 慢速定期删除的执行时长
        timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;
        timelimit_exit = 0;
        if (timelimit <= 0) timelimit = 1;
        if (type == ACTIVE_EXPIRE_CYCLE_FAST)
            timelimit = ACTIVE_EXPIRE_CYCLE_FAST_DURATION; /* 删除操作的执行时长 */
        long total_sampled = 0;
        long total_expired = 0;
        for (j = 0; j < dbs_per_call && timelimit_exit == 0; j++) {
            int expired;
            redisDb *db = server.db+(current_db % server.dbnum);
            current_db++;
            do {
                // .......
                expired = 0;
                ttl_sum = 0;
                ttl_samples = 0;
                // 每个数据库中检查的键的数量
                if (num > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP)
                    num = ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP;
                // 从数据库中随机选取 num 个键进行检查
                while (num--) {
                    dictEntry *de;
                    long long ttl;
                    if ((de = dictGetRandomKey(db->expires)) == NULL) break;
                    ttl = dictGetSignedInteger
                    // 过期检查，并对过期键进行删除
                    if (activeExpireCycleTryExpire(db,de,now)) expired++;
                    if (ttl > 0) {
                        /* We want the average TTL of keys yet not expired. */
                        ttl_sum += ttl;
                        ttl_samples++;
                    }
                    total_sampled++;
                }
                total_expired += expired;
                if (ttl_samples) {
                    long long avg_ttl = ttl_sum/ttl_samples;
                    if (db->avg_ttl == 0) db->avg_ttl = avg_ttl;
                    db->avg_ttl = (db->avg_ttl/50)*49 + (avg_ttl/50);
                }
                if ((iteration & 0xf) == 0) { /* check once every 16 iterations. */
                    elapsed = ustime()-start;
                    if (elapsed > timelimit) {
                        timelimit_exit = 1;
                        server.stat_expired_time_cap_reached_count++;
                        break;
                    }
                }
                /* 每次检查只删除 ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4 个过期键 */
            } while (expired > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4);
        }
        // .......
    }
    

activeExpireCycle 方法在规定的时间，分多次遍历各个数据库，从过期字典中随机检查一部分过期键的过期时间，删除其中的过期键。

这个函数有两种执行模式，一个是快速模式一个是慢速模式，体现是代码中的 timelimit 变量，这个变量是用来约束此函数的运行时间的。快速模式下
timelimit 的值是固定的，等于预定义常量 ACTIVE _EXPIRE_ CYCLE _FAST_ DURATION，慢速模式下，这个变量的值是通过
1000000*ACTIVE _EXPIRE_ CYCLE _SLOW_ TIME_PERC/server.hz/100 计算的。

如果只使用惰性删除会导致删除数据不及时造成一定的空间浪费，又因为 Redis
本身的主线程是单线程执行的，如果因为删除操作而影响主业务的执行就得不偿失了，为此 Redis 需要制定多个过期删除策略：惰性删除加定期删除的过期策略，来保证
Redis 能够及时并高效的删除 Redis 中的过期键。

#### 过期键

过期键存储在 redisDb 结构中，它的源码位于 src/server.h 文件中：

    
    
    // 源码基于 Redis 5.x
    typedef struct redisDb {
        dict *dict;                 /* 数据库键空间，存放着所有的键值对 */
        dict *expires;              /* 键的过期时间 */
        dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
        dict *ready_keys;           /* Blocked keys that received a PUSH */
        dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
        int id;                     /* Database ID */
        long long avg_ttl;          /* Average TTL, just for stats */
        list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
    } redisDb;
    

过期键数据结构如下图所示：
![微信截图_20191116185218.png](https://images.gitbook.cn/2020-06-11-072049.png)

### 总结

本文我们讲了三种常见的删除策略：定时删除、惰性删除、定期删除，其中定时删除比较消耗系统性能，惰性删除不能及时的清理过期数据从而导致了一定的空间浪费，为了兼顾存储空间和性能，Redis
采用了惰性删除加定期删除的组合删除策略，我们还通过 Redis 的源码分析了 Redis 各个删除策略的执行流程。当我们明白了 Redis
的过期删除知识之后，再去理解它与 Redis 内存淘汰的区别就显得非常容易了。

