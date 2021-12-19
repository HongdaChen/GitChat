延迟消息队列在我们的日常工作中经常会被用到，比如支付系统中超过 30
分钟未支付的订单，将会被取消，这样就可以保证此商品库存可以释放给其他人购买，还有外卖系统如果商家超过 5
分钟未接单的订单，将会被自动取消，以此来保证用户可以更及时的吃到自己点的外卖，等等诸如此类的业务场景都需要使用到延迟消息队列，又因为它在业务中比较常见，因此这个知识点在面试中也会经常被问到。

我们本文的面试题是，使用 Redis 如何实现延迟消息队列？

### 典型回答

延迟消息队列的常见实现方式是通过 ZSet
的存储于查询来实现，它的核心思想是在程序中开启一个一直循环的延迟任务的检测器，用于检测和调用延迟任务的执行，如下图所示：
![image.png](https://images.gitbook.cn/2020-06-22-091827.png) ZSet
实现延迟任务的方式有两种，第一种是利用 `zrangebyscore`
查询符合条件的所有待处理任务，循环执行队列任务；第二种实现方式是每次查询最早的一条消息，判断这条信息的执行时间是否小于等于此刻的时间，如果是则执行此任务，否则继续循环检测。

**方式一：zrangebyscore 查询所有任务** 此实现方式是一次性查询出所有的延迟任务，然后再进行执行，实现代码如下：

    
    
    import redis.clients.jedis.Jedis;
    import utils.JedisUtils;
    
    import java.time.Instant;
    import java.util.Set;
    
    /**
     * 延迟队列
     */
    public class DelayQueueExample {
        // zset key
        private static final String _KEY = "myDelayQueue";
    
        public static void main(String[] args) throws InterruptedException {
            Jedis jedis = JedisUtils.getJedis();
            // 延迟 30s 执行（30s 后的时间）
            long delayTime = Instant.now().plusSeconds(30).getEpochSecond();
            jedis.zadd(_KEY, delayTime, "order_1");
            // 继续添加测试数据
            jedis.zadd(_KEY, Instant.now().plusSeconds(2).getEpochSecond(), "order_2");
            jedis.zadd(_KEY, Instant.now().plusSeconds(2).getEpochSecond(), "order_3");
            jedis.zadd(_KEY, Instant.now().plusSeconds(7).getEpochSecond(), "order_4");
            jedis.zadd(_KEY, Instant.now().plusSeconds(10).getEpochSecond(), "order_5");
            // 开启延迟队列
            doDelayQueue(jedis);
        }
    
        /**
         * 延迟队列消费
         * @param jedis Redis 客户端
         */
        public static void doDelayQueue(Jedis jedis) throws InterruptedException {
            while (true) {
                // 当前时间
                Instant nowInstant = Instant.now();
                long lastSecond = nowInstant.plusSeconds(-1).getEpochSecond(); // 上一秒时间
                long nowSecond = nowInstant.getEpochSecond();
                // 查询当前时间的所有任务
                Set<String> data = jedis.zrangeByScore(_KEY, lastSecond, nowSecond);
                for (String item : data) {
                    // 消费任务
                    System.out.println("消费：" + item);
                }
                // 删除已经执行的任务
                jedis.zremrangeByScore(_KEY, lastSecond, nowSecond);
                Thread.sleep(1000); // 每秒轮询一次
            }
        }
    }
    

以上程序执行结果如下：

> 消费：order _2 消费：order_ 3 消费：order _4 消费：order_ 5 消费：order_1

**方式二：判断最早的任务**
此实现方式是每次查询最早的一条任务，再与当前时间进行判断，如果任务执行时间大于当前时间则表示应该立即执行延迟任务，实现代码如下：

    
    
    import redis.clients.jedis.Jedis;
    import utils.JedisUtils;
    
    import java.time.Instant;
    import java.util.Set;
    
    /**
     * 延迟队列
     */
    public class DelayQueueExample {
        // zset key
        private static final String _KEY = "myDelayQueue";
    
        public static void main(String[] args) throws InterruptedException {
            Jedis jedis = JedisUtils.getJedis();
            // 延迟 30s 执行（30s 后的时间）
            long delayTime = Instant.now().plusSeconds(30).getEpochSecond();
            jedis.zadd(_KEY, delayTime, "order_1");
            // 继续添加测试数据
            jedis.zadd(_KEY, Instant.now().plusSeconds(2).getEpochSecond(), "order_2");
            jedis.zadd(_KEY, Instant.now().plusSeconds(2).getEpochSecond(), "order_3");
            jedis.zadd(_KEY, Instant.now().plusSeconds(7).getEpochSecond(), "order_4");
            jedis.zadd(_KEY, Instant.now().plusSeconds(10).getEpochSecond(), "order_5");
            // 开启延迟队列
            doDelayQueue2(jedis);
        }
    
        /**
         * 延迟队列消费（方式 2）
         * @param jedis Redis 客户端
         */
        public static void doDelayQueue2(Jedis jedis) throws InterruptedException {
            while (true) {
                // 当前时间
                long nowSecond = Instant.now().getEpochSecond();
                // 每次查询一条消息，判断此消息的执行时间
                Set<String> data = jedis.zrange(_KEY, 0, 0);
                if (data.size() == 1) {
                    String firstValue = data.iterator().next();
                    // 消息执行时间
                    Double score = jedis.zscore(_KEY, firstValue);
                    if (nowSecond >= score) {
                        // 消费消息（业务功能处理）
                        System.out.println("消费消息：" + firstValue);
                        // 删除已经执行的任务
                        jedis.zrem(_KEY, firstValue);
                    }
                }
                Thread.sleep(100); // 执行间隔
            }
        }
    }
    

以上程序执行结果和实现方式一相同，结果如下：

> 消费：order _2 消费：order_ 3 消费：order _4 消费：order_ 5 消费：order_1

其中，执行间隔代码 `Thread.sleep(100)` 可根据实际的业务情况删减或配置。

### 考点分析

延迟消息队列的实现方法有很多种，不同的公司可能使用的技术也是不同的，我上面是从 Redis
的角度出发来实现了延迟消息队列，但一般面试官不会就此罢休，会借着这个问题来问关于更多的延迟消息队列的实现方法，因此除了 Redis
实现延迟消息队列的方式，我们还需要具备一些其他的常见的延迟队列的实现方法。

和此知识点相关的面试题还有以下这些：

  * 使用 Java 语言如何实现一个延迟消息队列？
  * 你还知道哪些实现延迟消息队列的方法？

### 知识扩展

#### Java 中的延迟消息队列

我们可以使用 Java 语言中自带的 DelayQueue 数据类型来实现一个延迟消息队列，实现代码如下：

    
    
    public class DelayTest {
        public static void main(String[] args) throws InterruptedException {
            DelayQueue delayQueue = new DelayQueue();
            delayQueue.put(new DelayElement(1000));
            delayQueue.put(new DelayElement(3000));
            delayQueue.put(new DelayElement(5000));
            System.out.println("开始时间：" +  DateFormat.getDateTimeInstance().format(new Date()));
            while (!delayQueue.isEmpty()){
                System.out.println(delayQueue.take());
            }
            System.out.println("结束时间：" +  DateFormat.getDateTimeInstance().format(new Date()));
        }
    
        static class DelayElement implements Delayed {
            // 延迟截止时间（单面：毫秒）
            long delayTime = System.currentTimeMillis();
            public DelayElement(long delayTime) {
                this.delayTime = (this.delayTime + delayTime);
            }
            @Override
            // 获取剩余时间
            public long getDelay(TimeUnit unit) {
                return unit.convert(delayTime - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
            }
            @Override
            // 队列里元素的排序依据
            public int compareTo(Delayed o) {
                if (this.getDelay(TimeUnit.MILLISECONDS) > o.getDelay(TimeUnit.MILLISECONDS)) {
                    return 1;
                } else if (this.getDelay(TimeUnit.MILLISECONDS) < o.getDelay(TimeUnit.MILLISECONDS)) {
                    return -1;
                } else {
                    return 0;
                }
            }
            @Override
            public String toString() {
                return DateFormat.getDateTimeInstance().format(new Date(delayTime));
            }
        }
    }
    

以上程序执行的结果如下：

> 开始时间：2019-6-13 20:40:38 2019-6-13 20:40:39 2019-6-13 20:40:41 2019-6-13
> 20:40:43 结束时间：2019-6-13 20:40:43

此实现方式的优点是开发比较方便，可以直接在代码中使用，实现代码也比较简单，但它缺点是数据保存在内存中，因此可能存在数据丢失的风险，最大的问题是它无法支持分布式系统。

#### 使用 MQ 实现延迟消息队列

我们使用主流的 MQ 中间件也可以方便的实现延迟消息队列的功能，比如 RabbitMQ，我们可以通过它的 rabbitmq-delayed-message-
exchange 插件来实现延迟队列。

首先我们需要配置并开启 rabbitmq-delayed-message-exchange 插件，然后再通过以下代码来实现延迟消息队列。

配置消息队列：

    
    
    import com.example.rabbitmq.mq.DirectConfig;
    import org.springframework.amqp.core.*;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import java.util.HashMap;
    import java.util.Map;
    
    @Configuration
    public class DelayedConfig {
        final static String QUEUE_NAME = "delayed.goods.order";
        final static String EXCHANGE_NAME = "delayedec";
        @Bean
        public Queue queue() {
            return new Queue(DelayedConfig.QUEUE_NAME);
        }
    
        // 配置默认的交换机
        @Bean
        CustomExchange customExchange() {
            Map<String, Object> args = new HashMap<>();
            args.put("x-delayed-type", "direct");
            //参数二为类型：必须是 x-delayed-message
            return new CustomExchange(DelayedConfig.EXCHANGE_NAME, "x-delayed-message", true, false, args);
        }
        // 绑定队列到交换器
        @Bean
        Binding binding(Queue queue, CustomExchange exchange) {
            return BindingBuilder.bind(queue).to(exchange).with(DelayedConfig.QUEUE_NAME).noargs();
        }
    }
    

发送者实现代码如下：

    
    
    import org.springframework.amqp.AmqpException;
    import org.springframework.amqp.core.AmqpTemplate;
    import org.springframework.amqp.core.Message;
    import org.springframework.amqp.core.MessagePostProcessor;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Component;
    import java.text.SimpleDateFormat;
    import java.util.Date;
    
    @Component
    public class DelayedSender {
        @Autowired
        private AmqpTemplate rabbitTemplate;
    
        public void send(String msg) {
            SimpleDateFormat sf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            System.out.println("发送时间：" + sf.format(new Date()));
    
            rabbitTemplate.convertAndSend(DelayedConfig.EXCHANGE_NAME, DelayedConfig.QUEUE_NAME, msg, new MessagePostProcessor() {
                @Override
                public Message postProcessMessage(Message message) throws AmqpException {
                    message.getMessageProperties().setHeader("x-delay", 3000);
                    return message;
                }
            });
        }
    }
    

从上述代码我们可以看出，我们配置 3s 之后再进行任务执行。

消费者实现代码如下：

    
    
    import org.springframework.amqp.rabbit.annotation.RabbitHandler;
    import org.springframework.amqp.rabbit.annotation.RabbitListener;
    import org.springframework.stereotype.Component;
    import java.text.SimpleDateFormat;
    import java.util.Date;
    
    @Component
    @RabbitListener(queues = "delayed.goods.order")
    public class DelayedReceiver {
        @RabbitHandler
        public void process(String msg) {
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            System.out.println("接收时间：" + sdf.format(new Date()));
            System.out.println("消息内容：" + msg);
        }
    }
    

测试代码如下：

    
    
    import com.example.rabbitmq.RabbitmqApplication;
    import com.example.rabbitmq.mq.delayed.DelayedSender;
    import org.junit.Test;
    import org.junit.runner.RunWith;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.test.context.junit4.SpringRunner;
    
    import java.text.SimpleDateFormat;
    import java.util.Date;
    
    @RunWith(SpringRunner.class)
    @SpringBootTest
    public class DelayedTest {
    
        @Autowired
        private DelayedSender sender;
    
        @Test
        public void Test() throws InterruptedException {
            SimpleDateFormat sf = new SimpleDateFormat("yyyy-MM-dd");
            sender.send("Hi Admin.");
            Thread.sleep(5 * 1000); //等待接收程序执行之后，再退出测试
        }
    }
    

以上程序的执行结果为：

> 发送时间：2020-06-11 20:47:51 接收时间：2018-06-11 20:47:54 消息内容：Hi Admin.

从上述结果中可以看出，当消息进入延迟队列 3s 之后才被正常消费，执行结果符合我的预期，RabbitMQ 成功的实现了延迟消息队列。

### 总结

本文我们讲了延迟消息队列的两种使用场景：支付系统中的超过 30
分钟未支付的订单，将会被自动取消，以此来保证此商品的库存可以正常释放给其他人购买，还有外卖系统如果商家超过 5
分钟未接单的订单，将会被自动取消，以此来保证用户可以更及时的吃到自己点的外卖。并且我们讲了延迟队列的 4 种实现方式，使用 ZSet 的 2
种实现方式，以及 Java 语言中的 DelayQueue 的实现方式，还有 RabbitMQ 的插件 rabbitmq-delayed-message-
exchange 的实现方式。

