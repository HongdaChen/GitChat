“限流”这种事在生活中很常见，比如逢年过节时景点的限流，还有工作日的车辆单双号限流等，有人可能会问为什么要限流？我既然买了车子你还不让我上路开？还有我倒景点买了门票，景点不是能赚更多的钱吗？为什么要限流呢？

其实限流的主要目的就是为了保证整个系统的正常运行，比如以车辆限流为了，它的作用主要有两个，一个是为了保证我们生存空间的资源少受污染，尤其是近几年雾霾已经越来越严重了，如果不采取相应的手段会导致生态系统更加恶化，第二，目前车辆的增长速度已经远远的超过了市政道路的新建速度，尤其是上班的时候大家都在赶时间，如果车流量太大的话就会造成严重的交通拥堵，那么导致的直接后果就是大家上班都会迟到，为了解决这个问题所有需要限行。包括北上广深从几年前已经开始买车要摇号了，其实也是一种限流的手段，目的就是为了更好的保证我们整个系统的正常运行。

回到程序的这个层面也是一样，假设我们的系统只能为 10 万人同时提供购物服务，但是某一天因为老罗带货突然就涌进了 100
万用户，那么导致的直接后果就是服务器瘫痪，谁也甭想买东西了，所以这个时候我们需要“限流”的功能保证先让一部分用户享受购物的服务，而其他用户进行排队等待购物，这样就可以让整个系统正常的运转了。

我们本文的面试题是，使用 Redis 如何实现限流功能？

### 典型回答

我们可以使用 Redis 中的
ZSet（有序集合）加上滑动时间算法来实现简单的限流。所谓的滑动时间算法指的是以当前时间为截止时间，往前取一定的时间，比如往前取 60s 的时间，在这
60s 之内运行最大的访问数为 100，此时算法的执行逻辑为，先清除 60s 之前的所有请求记录，再计算当前集合内请求数量是否大于设定的最大请求数
100，如果大于则执行限流拒绝策略，否则插入本次请求记录并返回可以正常执行的标识给客户端。

滑动时间窗口如下图所示： ![image.png](https://images.gitbook.cn/2020-06-11-072100.png)
其中每一小个表示 10s，被红色虚线包围的时间段则为需要判断的时间间隔，比如 60s 秒允许 100 次请求，那么红色虚线部分则为 60s。

下面我们通过具体的代码来实现一下滑动时间算法，我们借助 Jedis 包来操作 Redis，实现在 pom.xml 添加 Jedis 框架的引用，配置如下：

    
    
    <!-- https://mvnrepository.com/artifact/redis.clients/jedis -->
    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>3.3.0</version>
    </dependency>
    

具体的 Java 实现代码如下：

    
    
    import redis.clients.jedis.Jedis;
    
    public class RedisLimit {
        // Redis 操作客户端
        static Jedis jedis = new Jedis("127.0.0.1", 6379);
    
        public static void main(String[] args) throws InterruptedException {
            for (int i = 0; i < 15; i++) {
                boolean res = isPeriodLimiting("java", 3, 10);
                if (res) {
                    System.out.println("正常执行请求：" + i);
                } else {
                    System.out.println("被限流：" + i);
                }
            }
            // 休眠 4s
            Thread.sleep(4000);
            // 超过最大执行时间之后，再从发起请求
            boolean res = isPeriodLimiting("java", 3, 10);
            if (res) {
                System.out.println("休眠后，正常执行请求");
            } else {
                System.out.println("休眠后，被限流");
            }
        }
    
        /**
         * 限流方法（滑动时间算法）
         * @param key      限流标识
         * @param period   限流时间范围（单位：秒）
         * @param maxCount 最大运行访问次数
         * @return
         */
        private static boolean isPeriodLimiting(String key, int period, int maxCount) {
            long nowTs = System.currentTimeMillis(); // 当前时间戳
            // 删除非时间段内的请求数据（清除老访问数据，比如 period=60 时，标识清除 60s 以前的请求记录）
            jedis.zremrangeByScore(key, 0, nowTs - period * 1000);
            long currCount = jedis.zcard(key); // 当前请求次数
            if (currCount >= maxCount) {
                // 超过最大请求次数，执行限流
                return false;
            }
            // 未达到最大请求数，正常执行业务
            jedis.zadd(key, nowTs, "" + nowTs); // 请求记录 +1
            return true;
        }
    }
    

以上程序的执行结果为：

> 正常执行请求：0 正常执行请求：1 正常执行请求：2 正常执行请求：3 正常执行请求：4 正常执行请求：5 正常执行请求：6 正常执行请求：7
> 正常执行请求：8 正常执行请求：9 被限流：10 被限流：11 被限流：12 被限流：13 被限流：14 休眠后，正常执行请求

### 考点分析

使用 ZSet 的方式加上滑动时间的算法固然可以实现简单的限流，但是这个解决方案存在一定的问题，比如当我们允许 60s 的最大访问次数为 1000w
的时候，此时如果使用 ZSet 的方式就会占用大量的空间用来存储请求的记录信息，并且它的判断和添加属于两条 Redis
执行指令是非原子单元的，所以使用它可以出现问题。那有没有更好的实现 Redis 限流的方法呢？限流的算法还有哪些？

### 知识扩展

#### 限流算法

文章的开头我们讲了滑动时间算法，它是属于实现限流的算法之一，其他的常见限流算法还有以下两个：

  * 漏桶算法
  * 令牌算法

#### 漏桶算法

漏桶算法的灵感源于漏斗，如下图所示：
![image.png](https://images.gitbook.cn/2020-06-11-072101.png)
滑动时间算法有一个问题就是在一定范围内，比如 60s 内只能有 10 个请求，当第一秒时就到达了 10 个请求，那么剩下的 59s
只能把所有的请求都给拒绝掉，而漏桶算法可以解决这个问题。

漏桶算法类似于生活中的漏斗，无论上面的水流倒入漏斗有多大，也就是无论请求有多少，它都是以均匀的速度慢慢流出的。当上面的水流速度大于下面的流出速度时，漏斗会慢慢变满，当漏斗满了之后就会丢弃新来的请求;当上面的水流速度小于下面流出的速度的话，漏斗永远不会被装满，并且可以一直流出。

漏桶算法的实现步骤是，先声明一个队列用来保存请求，这个队列相当于漏斗，当队列容量满了之后就放弃新来的请求，然后重新声明一个线程定期从任务队列中获取一个或多个任务进行执行，这样就实现了漏桶算法。

#### 令牌算法

在令牌桶算法中有一个程序以某种恒定的速度生成令牌，并存入令牌桶中，而每个请求需要先获取令牌才能执行，如果没有获取到令牌的请求可以选择等待或者放弃执行，如下图所示：

![image.png](https://images.gitbook.cn/2020-06-11-072102.png) 我们可以使用 Google
开源的 guava 包，很方便的实现令牌桶算法，首先在 pom.xml 添加 guava 引用，配置如下：

    
    
    <!-- https://mvnrepository.com/artifact/com.google.guava/guava -->
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>28.2-jre</version>
    </dependency>
    

具体实现代码如下：

    
    
    import com.google.common.util.concurrent.RateLimiter;
    
    import java.time.Instant;
    
    /**
     * Guava 实现限流
     */
    public class RateLimiterExample {
        public static void main(String[] args) {
            // 每秒产生 10 个令牌（每 100 ms 产生一个）
            RateLimiter rt = RateLimiter.create(10);
            for (int i = 0; i < 11; i++) {
                new Thread(() -> {
                    // 获取 1 个令牌
                    rt.acquire();
                    System.out.println("正常执行方法，ts:" + Instant.now());
                }).start();
            }
        }
    }
    

以上程序的执行结果为：

> 正常执行方法，ts:2020-05-15T14:46:37.175Z 正常执行方法，ts:2020-05-15T14:46:37.237Z
> 正常执行方法，ts:2020-05-15T14:46:37.339Z 正常执行方法，ts:2020-05-15T14:46:37.442Z
> 正常执行方法，ts:2020-05-15T14:46:37.542Z
>
> 正常执行方法，ts:2020-05-15T14:46:37.640Z
>
> 正常执行方法，ts:2020-05-15T14:46:37.741Z
>
> 正常执行方法，ts:2020-05-15T14:46:37.840Z
>
> 正常执行方法，ts:2020-05-15T14:46:37.942Z
>
> 正常执行方法，ts:2020-05-15T14:46:38.042Z
>
> 正常执行方法，ts:2020-05-15T14:46:38.142Z

从以上结果可以看出令牌确实是每 100ms 产生一个，而 acquire() 方法为阻塞等待获取令牌，它可以传递一个 int
类型的参数，用于指定获取令牌的个数。它的替代方法还有 tryAcquire()，此方法在没有可用令牌时就会返回 false 这样就不会阻塞等待了。当然
tryAcquire() 方法也可以设置超时时间，未超过最大等待时间会阻塞等待获取令牌，如果超过了最大等待时间，还没有可用的令牌就会返回 false。

### 更好的限流方案

前面我们将了滑动时间算法的实现代码，但它的缺点是占用内存空间大，并且是非原子操作；而令牌算法为程序级别的单机限流实现方案，有没有更好的分布式限流解决方案呢？

答案是有的，它就是 Redis 4.0 提供的 Redis-Cell 模块，该模块使用的是漏斗算法，并且提供了原子的限流指令，而且依靠 Redis
这个天生的分布式程序就可以实现完美的限流了。

Redis-Cell 实现限流的方法也很简单，只需要使用一条指令 cl.throttle 即可，使用示例如下：

    
    
    > cl.throttle mylimit 15 30 60
    1）（integer）0 # 0 表示获取成功，1 表示拒绝
    2）（integer）15 # 漏斗容量
    3）（integer）14 # 漏斗剩余容量
    4）（integer）-1 # 被拒绝之后，多长时间之后再试（单位：秒）-1 表示无需重试
    5）（integer）2 # 多久之后漏斗完全空出来
    

其中 15 为漏斗的容量，30 / 60s 为漏斗的速率。

### 总结

本文我们讲了实现限流的 3 种常见算法：滑动时间算法、漏桶算法和令牌算法，其中滑动时间算法是借助 Redis 中的 ZSet
实现的，但它的缺点是占用内存大，且不是原子操作；而令牌算法最简单的实现方式是 Google 开源的 guava 包，但它是单机版的限流方案，而 Redis-
Cell 则可以实现分布式原子级别的限流。

