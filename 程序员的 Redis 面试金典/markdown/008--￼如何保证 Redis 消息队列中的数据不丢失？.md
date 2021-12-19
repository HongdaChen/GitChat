Redis 最常见的业务场景就是缓存读取与存储，而随着时间的推移，有人开始将它作为消息队列来使用了，并且随着 Redis 版本的发展，在
Redis.2.0.0
中新增了发布订阅模式（Pub/Sub）代表着官方开始正式支持消息队列的功能了，直到今天为止还有部分公司在实现轻量级的消息队列时，依然会选择使用 Redis
来实现。并且消息队列的知识点也会作为一个进阶型的面试题经常出现在面试当中。

我们本文的面试题是，什么是消息队列？为什么要用消息队列？Redis 实现消息队列的方式有几种？如何保证 Redis 消息队列中的数据不丢失？

### 典型回答

消息队列（Message
Queue）是一种进程间通信或同一进程的不同线程间的通信方式，它的实现流程是一方会将消息存储在队列中，而另一方则从队列中读取相应的消息，消息队列提供了异步的通信协议，也就是说消息的发送者和接收者无需同时与消息队列进行交互。

消息队列中有几个重要的概念：

  * 生产者：是指发布消息的一方；
  * 消费者：接收消息的一方，也叫订阅者或订阅方；
  * 通道（channel）：也叫频道，它可以理解为某个消息队列的名称，首先消费者先要订阅某个 channel，然后当生产者把消息发送到这个 channel 中时，消费者就可以正常接收到消息了。

它们的执行流程如下图所示： ![image.png](https://images.gitbook.cn/2020-06-22-092148.png)
使用消息队列有如下好处：

  * 削峰填谷：将某一个时刻急速上升的请求压力转嫁给消息队列来处理，保证应用系统的正常运转；
  * 系统解耦：使用消息队列我们可以将生产者和消费者的耦合代码进行分离，变成两个完全独立的模块，从而更加方便维护与改造；
  * 更加可靠：使用消息队列可以永久性的保留业务数据，保证了数据在传输过程中不会因为意外情况，例如掉电而造成的数据丢失；
  * 扩展性：使用消息队列具备更好的可扩展性，例如用户在完善了个人信息之后，刚开始的需求是增加用户的经验值，有一天产品部门的同学要求完善了个人资料之后不但要增加用户的经验值还要加一定的虚拟金币，那么我们无需改动太多的业务代码，只需要完善了个人信息之后，给增加金币的 channel 中发送一条增加金币的消息即可，即使过两天产品的同学告诉你需要把功能改回去，你也无需改动太多的代码，只需要注释掉发送消息的代码即可，即使要扩展更多的功能也就是非常方便的。

Redis 实现消息队列的方式有四种：

  * 使用 List 方式来实现消息队列，主要使用的是 lpush/rpop 来实现消息的先进先出；
  * ZSet 实现方式，此方式与 List 方式类似，它是使用 zadd 和 zrangebyscore 来实现存入和读取；
  * Redis 自身提供的发布订阅模式，也就是使用 Publisher(发布者) 和 Subscriber(订阅者) 来实现消息队列；
  * 使用 Redis 5.0 版本中提供的 Stream 来实现消息队列，它主要使用的是 xadd/xread 来实现消息的读取和存储。

Redis 想要保住消息队列中的数据不丢失，必须要做到以下两点：

  1. 必须开启消息的持久化功能，负责在 Redis 重启之后消息就会全部丢失；
  2. 需要使用 Stream 中提供的消息确认功能，保证消息能够被正常消费。

### 考点分析

消息队列本身的概念并不难懂，这个就好像买货和卖货一样，最早之前我们卖家和买家是直接面对面进行交易的，但这种方式有很多的局限，比如很难规模话，需要很大的人力成本等，那么随着科技的发展我们可以创建更多的无人超市，而这些无人超市就相当于消息队列中的
channel，卖家相当于生产者，每次只需要负责把货物存放到无人超市就行了，而买家也不用再找卖家直接买货了，只需要每次去无人超市取货就可以了，这样就可以开越来越多的无人超市，即使搞双
11、618 等活动也不怕了，因为那时候就有足够多的无人超市了，这个例子其实就可以用来很好的理解消息队列的核心思路和工作原理，他们之间的关系如下图所示：
![image.png](https://images.gitbook.cn/2020-06-22-092204.png)
理解了消息队列的概念之后，我们重点应该关注的就是消息队列的实现方式了。

和此知识点相关的面试题还有以下这些：

  * 消息队列的几种实现方式在 Java 代码中应该如何实现？
  * 消息队列几种实现方式的优缺点有哪些？

### 消息队列的具体实现

#### 1.List 版实现方式

我们本文将使用 Java 代码来实现相关的示例，在 Java 项目中我们会使用到 Jedis 框架来作为 Redis 的客户端来进行相关的 Redis
操作，在引用了 Jedis 之后，我们实现代码如下：

    
    
    import redis.clients.jedis.Jedis;
    
    public class ListMQExample {
        public static void main(String[] args) throws InterruptedException {
            // 消费者
            new Thread(() -> bConsumer()).start();
            // 生产者
            producer();
        }
        /**
         * 生产者
         */
        public static void producer() throws InterruptedException {
            Jedis jedis = new Jedis("127.0.0.1", 6379);
            // 推送消息
            jedis.lpush("mq", "Hello, List.");
            Thread.sleep(1000);
            jedis.lpush("mq", "message 2.");
            Thread.sleep(2000);
            jedis.lpush("mq", "message 3.");
        }
        /**
         * 消费者（阻塞版）
         */
        public static void bConsumer() {
            Jedis jedis = new Jedis("127.0.0.1", 6379);
            while (true) {
                // 阻塞读
                for (String item : jedis.brpop(0,"mq")) {
                    // 读取到相关数据，进行业务处理
                    System.out.println(item);
                }
            }
        }
    }
    

以上程序的运行结果是：

> 接收到消息：Hello, List.

我们使用无限循环来获取队列中的数据，这样就可以实时的获取相关信息了，其中 brpop() 方法的第一个参数是设置超时时间的，设置 0
表示一直阻塞，也就是说当队列没有数据时，while 循环就会进入休眠状态，当有数据进入队列之后，它才会“苏醒”过来执行读取任务，这样就可以解决 while
循环一直执行消耗系统资源的问题了。

ZSet 的实现方式和 List 实现方式类似，它主要借助的是 zadd 和 zrangebyscore 来实现存入和读取，这里就不再进行赘述了。

#### 2.Pub/Sub 实现方式

此模式是 Redis 提供的发布订阅模式，它的相关实现代码如下。

消费者代码如下：

    
    
    /**
     * 消费者
     */
    public static void consumer() {
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        // 接收并处理消息
        jedis.subscribe(new JedisPubSub() {
            @Override
            public void onMessage(String channel, String message) {
                // 接收消息，业务处理
                System.out.println("频道 " + channel + " 收到消息：" + message);
            }
        }, "channel");
    }
    

生产者代码如下：

    
    
    /**
     * 生产者
     */
    public static void producer() {
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        // 推送消息
        jedis.publish("channel", "Hello, channel.");
    }
    

发布者和订阅者模式运行：

    
    
    public static void main(String[] args) throws InterruptedException {
        // 创建一个新线程作为消费者
        new Thread(() -> consumer()).start();
        // 暂停 0.5s 等待消费者初始化
        Thread.sleep(500);
        // 生产者发送消息
        producer();
    }
    

以上代码运行结果如下：

> 频道 channel 收到消息：Hello, channel.

此模式提供了主题订阅的模式，例如我要订阅日志类的消息队列，它们的命名都是 logXXX，这个时候就需要使用 Redis 提供的另一个功能 `Pattern
Subscribe` 主题订阅，这种方式可以使用 “*” 来匹配多个频道，如下图所示：
![image.png](https://images.gitbook.cn/2020-06-22-092224.png)

它的相关实现代码如下。

主题订阅模式的生产者的代码是一样，只有消费者的代码是不同的，如下所示：

    
    
    /**
     * 主题订阅
     */
    public static void pConsumer() {
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        // 主题订阅
        jedis.psubscribe(new JedisPubSub() {
            @Override
            public void onPMessage(String pattern, String channel, String message) {
                // 接收消息，业务处理
                System.out.println(pattern + " 主题 | 频道 " + channel + " 收到消息：" + message);
            }
        }, "channel*");
    }
    

主题模式运行代码如下：

    
    
    public static void main(String[] args) throws InterruptedException {
        // 主题订阅
        new Thread(() -> pConsumer()).start();
        // 暂停 0.5s 等待消费者初始化
        Thread.sleep(500);
        // 生产者发送消息
        producer();
    }
    

以上代码运行结果如下：

> channel* 主题 | 频道 channel 收到消息：Hello, channel.

#### 3.Stream 实现方式

Stream 实现消息队列的代码如下：

    
    
    import com.google.gson.Gson;
    import redis.clients.jedis.Jedis;
    import redis.clients.jedis.StreamEntry;
    import redis.clients.jedis.StreamEntryID;
    import utils.JedisUtils;
    
    import java.util.AbstractMap;
    import java.util.HashMap;
    import java.util.List;
    import java.util.Map;
    
    public class StreamGroupExample {
        private static final String _STREAM_KEY = "mq"; // 流 key
        private static final String _GROUP_NAME = "g1"; // 分组名称
        private static final String _CONSUMER_NAME = "c1"; // 消费者 1 的名称
        private static final String _CONSUMER2_NAME = "c2"; // 消费者 2 的名称
        public static void main(String[] args) {
            // 生产者
            producer();
            // 创建消费组
            createGroup(_STREAM_KEY, _GROUP_NAME);
            // 消费者 1
            new Thread(() -> consumer()).start();
            // 消费者 2
            new Thread(() -> consumer2()).start();
        }
        /**
         * 创建消费分组
         * @param stream    流 key
         * @param groupName 分组名称
         */
        public static void createGroup(String stream, String groupName) {
            Jedis jedis = JedisUtils.getJedis();
            jedis.xgroupCreate(stream, groupName, new StreamEntryID(), true);
        }
        /**
         * 生产者
         */
        public static void producer() {
            Jedis jedis = JedisUtils.getJedis();
            // 添加消息 1
            Map<String, String> map = new HashMap<>();
            map.put("data", "redis");
            StreamEntryID id = jedis.xadd(_STREAM_KEY, null, map);
            System.out.println("消息添加成功 ID：" + id);
            // 添加消息 2
            Map<String, String> map2 = new HashMap<>();
            map2.put("data", "java");
            StreamEntryID id2 = jedis.xadd(_STREAM_KEY, null, map2);
            System.out.println("消息添加成功 ID：" + id2);
        }
        /**
         * 消费者 1
         */
        public static void consumer() {
            Jedis jedis = JedisUtils.getJedis();
            // 消费消息
            while (true) {
                // 读取消息
                Map.Entry<String, StreamEntryID> entry = new AbstractMap.SimpleImmutableEntry<>(_STREAM_KEY,
                        new StreamEntryID().UNRECEIVED_ENTRY);
                // 阻塞读取一条消息（最大阻塞时间 120s）
                List<Map.Entry<String, List<StreamEntry>>> list = jedis.xreadGroup(_GROUP_NAME, _CONSUMER_NAME, 1,
                        120 * 1000, true, entry);
                if (list != null && list.size() == 1) {
                    // 读取到消息
                    Map<String, String> content = list.get(0).getValue().get(0).getFields(); // 消息内容
                    System.out.println("Consumer 1 读取到消息 ID：" + list.get(0).getValue().get(0).getID() +
                            " 内容：" + new Gson().toJson(content));
                }
            }
        }
        /**
         * 消费者 2
         */
        public static void consumer2() {
            Jedis jedis = JedisUtils.getJedis();
            // 消费消息
            while (true) {
                // 读取消息
                Map.Entry<String, StreamEntryID> entry = new AbstractMap.SimpleImmutableEntry<>(_STREAM_KEY,
                        new StreamEntryID().UNRECEIVED_ENTRY);
                // 阻塞读取一条消息（最大阻塞时间 120s）
                List<Map.Entry<String, List<StreamEntry>>> list = jedis.xreadGroup(_GROUP_NAME, _CONSUMER2_NAME, 1,
                        120 * 1000, true, entry);
                if (list != null && list.size() == 1) {
                    // 读取到消息
                    Map<String, String> content = list.get(0).getValue().get(0).getFields(); // 消息内容
                    System.out.println("Consumer 2 读取到消息 ID：" + list.get(0).getValue().get(0).getID() +
                            " 内容：" + new Gson().toJson(content));
                }
            }
        }
    }
    

以上代码运行结果如下：

> 消息添加成功 ID：1580971482344-0 消息添加成功 ID：1580971482415-0 Consumer 1 读取到消息
> ID：1580971482344-0 内容：{"data":"redis"} Consumer 2 读取到消息 ID：1580971482415-0
> 内容：{"data":"java"}

其中，jedis.xreadGroup() 方法的第五个参数 noAck 表示是否自动确认消息，如果设置 true 收到消息会自动确认 (ack)
消息，否则则需要手动确认。

> 注意：Jedis 框架要使用最新版，低版本 block 设置大于 0 时，会有 bug 抛连接超时异常。

可以看出，同一个分组内的多个 consumer 会读取到不同消息，不同的 consumer 不会读取到分组内的同一条消息。

### 消息队列优缺点分析

Redis 实现消息队列的四种实现方式各有优缺点，首先来说 List 实现方式。

List 优点：

  * 消息可以被持久化，借助 Redis 本身的持久化 (AOF、RDB 或者是混合持久化)，可以有效的保存数据；
  * 消费者可以积压消息，不会因为客户端的消息过多而被强行断开。

List 缺点：

  * 消息不能被重复消费，一个消息消费完就会被删除；
  * 没有主题订阅的功能。

ZSet 优点：

  * 支持消息持久化；
  * 相比于 List 查询更方便，ZSet 可以利用 score 属性很方便的完成检索，而 List 则需要遍历整个元素才能检索到某个值。

ZSet 缺点：

  * ZSet 不能存储相同元素的值，也就是如果有消息是重复的，那么只能插入一条信息在有序集合中；
  * ZSet 是根据 score 值排序的，不能像 List 一样，按照插入顺序来排序；
  * ZSet 没有向 List 的 brpop 那样的阻塞弹出的功能。

Pub/Sub 优点：

  * 提供了普通消息订阅模式和主题订阅模式。

Pub/Sub 缺点：

  * 无法持久化保存消息，如果 Redis 服务器宕机或重启，那么所有的消息将会丢失；
  * 发布订阅模式是“发后既忘”的工作模式，如果有订阅者离线重连之后不能消费之前的历史消息。

Stream 优点：

  * 支持消息持久化；
  * 支持消息消费确认。

Stream 缺点：

  * 实现代码比较繁琐；
  * 没有提供主题订阅的功能。

### 总结

本文我们讲了消息队列的概念，以及消息队列的 3 个重要组成部分：生产者、消费者、通道，还讲了 Redis 中实现消息队列的四种方式，其中 List 和
ZSet 的实现方式比较类似，List 是通过 lpush/rpop 来实现消息存入和读取，而 ZSet 是通过 zadd/zrangebyscore
来实现消息存入和读取，最后我们讲了 Pub/Sub 的实现方式以及 Stream 的实现方式，其中 Pub/Sub 的实现方式支持主题订阅，而 Stream
提供了消息消费确认的功能。

