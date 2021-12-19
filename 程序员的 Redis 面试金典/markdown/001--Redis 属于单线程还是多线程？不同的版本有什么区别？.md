Redis 是普及率最高的技术之一，同时也是面试中必问的一个技术模块，所以从今天开始我们将从最热门的 Redis 面试题入手，更加深入的学习和了解一下
Redis。

我们本文的面试题是 Redis 属于单线程还是多线程？

### 典型回答

本文的问题在不同的 Redis 版本下答案是不同的，在 Redis 4.0 之前，Redis 是单线程运行的，但单线程并不意味着性能低，类似单线程的程序还有
Nginx 和 NodeJs 他们都是单线程程序，但是效率并不低。 Redis 的 FAQ（Frequently Asked
Questions，常见问题）也回到过这个问题，具体内容如下：

> Redis is single threaded. How can I exploit multiple CPU / cores?
>
> It's not very frequent that CPU becomes your bottleneck with Redis, as
> usually Redis is either memory or network bound. For instance, using
> pipelining Redis running on an average Linux system can deliver even 1
> million requests per second, so if your application mainly uses O(N) or
> O(log(N)) commands, it is hardly going to use too much CPU.
>
> However, to maximize CPU usage you can start multiple instances of Redis in
> the same box and treat them as different servers. At some point a single box
> may not be enough anyway, so if you want to use multiple CPUs you can start
> thinking of some way to shard earlier.
>
> You can find more information about using multiple Redis instances in the
> Partitioning page.
>
> However with Redis 4.0 we started to make Redis more threaded. For now this
> is limited to deleting objects in the background, and to blocking commands
> implemented via Redis modules. For future releases, the plan is to make
> Redis more and more threaded.

详情请见：<https://redis.io/topics/faq>

他的大体意思是说 Redis 是基于内存操作的，因此他的瓶颈可能是机器的内存或者网络带宽而并非 CPU，既然 CPU
不是瓶颈，那么自然就采用单线程的解决方案了，况且使用多线程比较麻烦。但是在 Redis 4.0 中开始支持多线程了，例如后台删除等功能。

简单来说 Redis 之所以在 4.0 之前一直采用单线程的模式是因为以下三个原因：

  * 使用单线程模型是 Redis 的开发和维护更简单，因为单线程模型方便开发和调试；
  * 即使使用单线程模型也并发的处理多客户端的请求，主要使用的是多路复用（详见本文下半部分）；
  * 对于 Redis 系统来说，主要的性能瓶颈是内存或者网络带宽而并非 CPU。

Redis 在 4.0 中引入了惰性删除（也可以叫异步删除），意思就是说我们可以使用异步的方式对 Redis 中的数据进行删除操作了，例如 `unlink
key` / `flushdb async` / `flushall async` 等命令，他们的执行示例如下：

    
    
    > unlink key # 后台删除某个 key
    > OK # 执行成功
    > flushall async # 清空所有数据
    > OK # 执行成功
    

这样处理的好处是不会导致 Redis 主线程卡顿，会把这些删除操作交给后台线程来执行。

> 小贴士：通常情况下使用 del 指令可以很快的删除数据，而当被删除的 key 是一个非常大的对象时，例如时包含了成千上万个元素的 hash 集合时，那么
> del 指令就会造成 Redis 主线程卡顿，因此使用惰性删除可以有效的避免 Redis 卡顿的问题。

### 考点分析

关于 Redis 线程模型的问题（单线程或多线程）几乎是 Redis 必问的问题之一，但能回答好的人却寥寥无几，大部分的人只能回到上来 Redis
是单线程的以及说出来单线程的众多好处，但对于 Redis 4.0 和 Redis 6.0 中，尤其是 Redis 6.0
中的多线程能回答上来的人少之又少，和这个知识点相关的面试题还有以下这些。

  * Redis 主线程既然是单线程的，为什么还这么快？
  * 介绍一下 Redis 中的多路复用？
  * 介绍一下 Redis 6.0 中的多线程？

### 知识扩展

#### 1.Redis 为什么这么快？

我们知道 Redis 4.0 之前是单线程的，那既然是单线程为什么还能这么快？

Redis 速度比较快的原因有以下几点：

  * 基于内存操作：Redis 的所有数据都存在内存中，因此所有的运算都是内存级别的，所以他的性能比较高；
  * 数据结构简单：Redis 的数据结构比较简单，是为 Redis 专门设计的，而这些简单的数据结构的查找和操作的时间复杂度都是 O(1)，因此性能比较高；
  * 多路复用和非阻塞 I/O：Redis 使用 I/O 多路复用功能来监听多个 socket 连接客户端，这样就可以使用一个线程连接来处理多个请求，减少线程切换带来的开销，同时也避免了 I/O 阻塞操作，从而大大提高了 Redis 的性能；
  * 避免上下文切换：因为是单线程模型，因此就避免了不必要的上下文切换和多线程竞争，这就省去了多线程切换带来的时间和性能上的消耗，而且单线程不会导致死锁问题的发生。

官方使用基准测试的结果是，单线程的 Redis 吞吐量可以达到 10W/每秒，如下图所示：
![image.png](https://images.gitbook.cn/2020-06-11-072103.png)

#### 2.I/O 多路复用

套接字的读写方法默认情况下是阻塞的，例如当调用读取操作 read
方法时，缓冲区没有任何数据，那么这个线程就会阻塞卡在这里，直到缓冲区有数据或者是连接被关闭时，read 方法才可以返回，线程才可以继续处理其他业务。

但这样显然降低了程序的整体执行效率，而 Redis 使用的就是非阻塞的 I/O，这就意味着 I/O
的读写流程不再是阻塞的，读写方法都是瞬间完成并返回的，也就是他会采用能读多少读多少能写多少写多少的策略来执行 I/O 操作，这显然更符合我们对性能的追求。

但这种非阻塞的 I/O
依然存在一个问题，那就是当我们执行读取数据操作时，有可能只读取了一部分数据，同样写入数据也是这种情况，当缓存区满了之后我们的数据还没写完，剩余的数据何时写何时读就成了一个问题。

而 I/O 多路复用就是解决上面的这个问题的，使用 I/O 多路复用最简单的实现方式就是使用 select 函数，此函数为操作系统提供给用户程序的 API
接口，是用于监控多个文件描述符的可读和可写情况的，这样就可以监控到文件描述符的读写事件了，当监控到相应的事件之后就可以通知线程处理相应的业务了，这样就保证了
Redis 读写功能的正常执行了。

I/O 多路复用执行流程如下图所示：
![image.png](https://images.gitbook.cn/2020-06-11-072105.png)

> 小贴士：现在的操作系统已经很少使用 select 函数了，改为调用 epoll（linux）和 kqueue（MacOS）等函数了，因为 select
> 函数在文件描述符特别多时性能非常的差。

#### 3.Redis 6.0 多线程

Redis 单线程的优点很明显，不但降低了 Redis
内部的实现复杂性，也让所有操作都可以在无锁的情况下进行操作，并且不存在死锁和线程切换带来的性能和时间上的消耗，但缺点也很明显，单线程的机制导致 Redis
的 QPS（Query Per Second，每秒查询率）很难得到有效的提高。

Redis 4.0 版本中虽然引入了多线程，但此版本中的多线程只能用于大数据量的异步删除，然而对于非删除操作的意义并不是很大。

如果我们使用多线程就可以分摊 Redis 同步读写 I/O 的压力，以及充分的利用多核 CPU 的资源，并且可以有效的提升 Redis 的 QPS。在
Redis 中虽然使用了 I/O 多路复用，并且是基于非阻塞 I/O 进行操作的，但 I/O 的读和写本身是堵塞的，比如当 socket
中有数据时，Redis 会通过调用先将数据从内核态空间拷贝到用户态空间，再交给 Redis
调用，而这个拷贝的过程就是阻塞的，当数据量越大时拷贝所需要的时间就越多，而这些操作都是基于单线程完成的。

因此在 Redis 6.0 中新增了多线程的功能来提高 I/O 的读写性能，他的主要实现思路是将主线程的 IO
读写任务拆分给一组独立的线程去执行，这样就可以使多个 socket 的读写可以并行化了，但 Redis 的命令依旧是由主线程串行执行的。

需要注意的是 Redis 6.0 默认是禁用多线程的，可以通过修改 Redis 的配置文件 redis.conf 中的 `io-threads-do-
reads` 等于 `true` 来开启多线程，完整配置为 `io-threads-do-reads
true`，除此之外我们还需要设置线程的数量才能正确的开启多线程的功能，同样是修改 Redis 的配置，例如设置 `io-threads 4` 表示开启 4
个线程。

> 小贴士：关于线程数的设置，官方的建议是如果为 4 核的 CPU，建议线程数设置为 2 或 3，如果为 8 核 CPU 建议线程数设置为
> 6，线程数一定要小于机器核数，线程数并不是越大越好。

关于 Redis 的性能，Redis 作者 antirez 在 RedisConf 2019 分享时曾提到，Redis 6 引入的多线程 I/O
特性对性能提升至少是一倍以上。国内也有人在阿里云使用 4 个线程的 Redis 版本和单线程的 Redis 进行比较测试，发现测试的结果和 antirez
给出的结论基本吻合，性能基本可以提高一倍。

### 总结

本文我们介绍了 Redis 在 4.0 之前单线程依然很快的原因：基于内存操作、数据结构简单、多路复用和非阻塞 I/O、避免了不必要的线程上下文切换，在
Redis 4.0 中已经添加了多线程的支持，主要体现在大数据的异步删除功能上，例如 unlink key、flushdb async、flushall
async 等，Redis 6.0 新增了多线程 I/O 的读写并发能力，用于更好的提高 Redis 的性能。

