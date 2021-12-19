内存问题 -- OutOfMemoryError 问题是我们在使用 Elasticsearch 过程中遇到的最大问题，Circuit breakers 是
Elasticsearch 用来防止集群出现该问题的解决方案。

Elasticsearch 含多种断路器用来避免因为不合理的操作引起来的
OutOfMemoryError（内存溢出错误）。每个断路器指定可以使用多少内存的限制。
另外，还有一个父级别的断路器，指定可以在所有断路器上使用的内存总量。

### 1 Circuit breadkers 分类

所有的 Circuit breaker 都支持动态配置，例如：

    
    
    curl -XPUT localhost:9200/_cluster/settings -d '{"persistent" : {"indices.breaker.total.limit":"40%"}}'
    curl -XPUT localhost:9200/_cluster/settings -d '{"persistent" : {"indices.breaker.fielddata.limit":"10%"}}'
    curl -XPUT localhost:9200/_cluster/settings -d '{"persistent" : {"indices.breaker.request.limit":"20%"}}'
    curl -XPUT localhost:9200/_cluster/settings -d '{"transient" : {"network.breaker.inflight_requests.limit":"20%"}}'
    curl -XPUT localhost:9200/_cluster/settings -d '{"transient" : {"indices.breaker.accounting.limit":"20%"}}'
    curl -XPUT localhost:9200/_cluster/settings -d '{"transient" : {"script.max_compilations_rate":"20%"}}'
    

#### 1.1 Parent circuit breaker

**1\. 作用**

设置所有 Circuit breakers 可以使用的内存的总量。

**2\. 配置项**

  * `indices.breaker.total.limit`

默认值为 70% JVM 堆大小。

#### 1.2 Field data circuit breaker

**1\. 作用**

估算加载 fielddata 需要的内存，如果超过配置的值就短路。

**2\. 配置项**

  * `indices.breaker.fielddata.limit`

默认值 是 60% JVM 堆大小。

  * `indices.breaker.fielddata.overhead`

所有估算的列数据占用内存大小乘以一个常量得到最终的值。默认值是1.03。

  * `indices.fielddata.cache.size`

该配置不属于 Circuit breaker，但是都与 Fielddata 有关，所以有必要在这里提一下。主要控制 Fielddata Cache
占用的内存大小。

默认值是不限制，Elasticsearch 认为加载 Fielddata 是很重的操作，频繁的重复加载会严重影响性能，所以建议分配足够的内存做 field
data cache。

该配置和 Circuit breaker 配置还有一个不同点是这是一个静态配置，如果修改需要修改集群中每个节点的配置文件，并重启节点。

可以通过 cat nodes api 监控 field data cache 占用的内存空间：

    
    
    curl -sXGET "http://localhost:9200/_cat/nodes?h=name,fielddata.memory_size&v"
    

输出如下（注：存在 `fielddata.memory_size` 为 0 是因为本集群部署了 5 个查询节点，没有存储索引数据）：

    
    
    name    fielddata.memory_size
    node1               224.2mb
    node2               225.5mb
    node3               168.7mb
    node4                    0b
    node5                    0b
    node6               168.4mb
    node7               223.8mb
    node8               150.6mb
    node9               169.5mb
    node10                   0b
    node11              224.7mb
    node12                   0b
    node13                   0b
    

`indices.fielddata.cache.size` 与 `indices.breaker.fielddata.limit` 的 **区别**
：前者是控制 fielddata 占用内存的大小，后者是防止加载过多大的 fielddata 导致 OOM 异常。

#### 1.3 Request circuit breaker

**1\. 作用**

请求断路器允许 Elasticsearch 防止每个请求数据结构（例如，用于在请求期间计算聚合的内存）超过一定量的内存。

**2\. 配置项**

  * `indices.breaker.request.limit`

默认值是 60% JVM 堆大小。

  * `indices.breaker.request.overhead`

所有请求乘以一个常量得到最终的值。默认值是 1。

#### 1.4 In flight circuit breaker

**1\. 作用**

请求中的断路器，允许 Elasticsearch 限制在传输或 HTTP 级别上的所有当前活动的传入请求的内存使用超过节点上的一定量的内存。
内存使用是基于请求本身的内容长度。

**2\. 配置项**

  * `network.breaker.inflight_requests.limit`

默认值是 100% JVM 堆大小，也就是说该 breaker 最终受 `indices.breaker.total.limit` 配置限制。

  * `network.breaker.inflight_requests.overhead`

所有 (inflight_requests) 请求中估算的常数乘以确定最终估计，默认值是1。

#### 1.5 Accounting requests circuit breaker

**1\. 作用**

估算一个请求结束后不能释放的对象占用的内存。包括底层 Lucene 索引文件需要常驻内存的对象。

**2\. 配置项**

  * `indices.breaker.accounting.limit`

默认值是 100% JVM 堆大小，也就是说该 breaker 最终受`indices.breaker.total.limit`配置限制。

  * `indices.breaker.accounting.overhead`

默认值是1。

#### 1.6 Script compilation circuit breaker

**1\. 作用**

与上面的基于内存的断路器略有不同，脚本编译断路器在一段时间内限制脚本编译的数量。

**2\. 配置项**

  * `script.max_compilations_rate`

默认值是 75/5m。也就是每 5 分钟可以进行 75 次脚本编译。

### 2 Circuit breaker 状态

合理配置 Circuit breaker 大小需要了解当前 breaker 的状态，可以通过 Elasticsearch 的 stats api 获取当前
breaker 的状态，包括配置的大小、当前占用大小、overhead 配置以及触发的次数。

    
    
    curl -sXGET     "http://localhost:9200/_nodes/stats/breaker?pretty"
    

执行上面的命令后，返回各个节点的 Circuit breakers 状态：

    
    
    "breakers" : {
    "request" : {
        "limit_size_in_bytes" : 6442450944,
        "limit_size" : "6gb",
        "estimated_size_in_bytes" : 690875608,
        "estimated_size" : "658.8mb",
        "overhead" : 1.0,
        "tripped" : 0
    },
    "fielddata" : {
        "limit_size_in_bytes" : 11274289152,
        "limit_size" : "10.5gb",
        "estimated_size_in_bytes" : 236500264,
        "estimated_size" : "225.5mb",
        "overhead" : 1.03,
        "tripped" : 0
    },
    "in_flight_requests" : {
        "limit_size_in_bytes" : 32212254720,
        "limit_size" : "30gb",
        "estimated_size_in_bytes" : 18001,
        "estimated_size" : "17.5kb",
        "overhead" : 1.0,
        "tripped" : 0
    },
    "parent" : {
        "limit_size_in_bytes" : 17716740096,
        "limit_size" : "16.5gb",
        "estimated_size_in_bytes" : 927393873,
        "estimated_size" : "884.4mb",
        "overhead" : 1.0,
        "tripped" : 0
        }
    }
    

其中重点需要关注的是 `limit_size` 与 `estimated_size` 大小是否相近，越接近越有可能触发熔断。tripped 数量是否大于
0，如果大于 0 说明已经触发过熔断。

### 3 Circuit breaker 配置原则

Circuit breaker 的目的是防止 **不当的操作** 导致进程出现 OutOfMemoryError
问题。不能由于触发了某个断路器就盲目调大对应参数的设置，也不能由于节点经常发生 OutOfMemoryError
错误就盲目调小各个断路器的设置。需要结合业务合理评估参数的设置。

#### 3.1 不同版 circuit breakers 区别

Elasticsearch 从 2.0 版本开始，引入 Circuit breaker 功能，而且随着版本的变化，Circuit breaker
的类型和默认值也有一定的变化，具体如下表所示：

版本 | Parent | Fielddata | Request | Inflight | Script | Accounting  
---|---|---|---|---|---|---  
2.0-2.3 | 70% | 60% | 40% | 无 | 无 | 无  
2.4 | 70% | 60% | 40% | 100% | 无 | 无  
5.x-6.1 | 70% | 60% | 60% | 100% | 1分钟15次 | 无  
6.2-6.5 | 70% | 60% | 60% | 100% | 1分钟15次 | 100%  
  
从上表中可见，Elasticsearch 也在不断调整和完善 Circuit breaker 相关的默认值，并不断增加不同类型的 Circuit
breaker 来减少 Elasticsearch 节点出现 OOM 的概率。

> **注：** 顺便提一下，Elasticsearch 7.0 增加了 `indices.breaker.total.use_real_memory`
> 配置项，可以更加精准的分析当前的内存情况，及时防止 OOM 出现。虽然该配置会增加一点性能损耗，但是可以提高 JVM
> 的内存使用率，增强了节点的保护机制。

#### 3.2 默认值的问题

Elasticsearch 对于 Circuit breaker 的默认值设置的都比较激进、乐观的，尤其是对于 6.2（不包括
6.2）之前的版本，这些版本中没有 **accounting circuit breaker** ，节点加载打开的索引后，Lucene
的一些数据结构需要常驻内存， **Parent circuit breakeredit** 配置成堆的 70%，很容易发生 OOM。

#### 3.3 配置思路

不同的业务场景，不同的数据量，不同的硬件配置，Circuit breaker 的设置应该是有差异的。 配置的过大，节点容易发生 OOM
异常，配置的过小，虽然节点稳定，但是会经常出现触发断路的问题，导致一部分合理应用无法完成。这里我们介绍下在配置时需要考虑的问题。

  * 1\. 了解 Elasticsearch 内存分布

**Circuit breaker** 最主要的作用就是防止节点出现 OOM 异常，所以，掌握 Elasticsearch 中都有哪些组件占用内存是配置好
Circuit breaker 的第一步。

> 具体参见本课程中《常见问题之-内存问题》一章。

  * 2\. Parent circuit breaker 配置

前面提到 Elasticsearch 6.2 之前的版本是不包含 accounting requests circuit breaker
的，所以需要根据自己的数据特点，评估 Lucene segments 占用的内存量占 JVM heap 、index buffer、Query
Cache、Request Cache 占用的内存的百分比，并用 70% 减去评估出的值作为 parent circuit breaker 的值。 对于
6.2 以后的版本，不需要减掉 Lucene segments 占用的百分比。

  * 3\. Fielddata circuit breaker 配置

在 Elasticsearch 引入 `doc_values` 后，我们十分不建议继续使用 fielddata，一是 feilddata
占用内存过大，二是在 text 字段上排序和聚合没有意义。Fielddata 占用的内存应该仅限于 Elasticsearch 在构建 global
ordinals 数据结构时占用的内存。

有一点注意的是，Elasticsearch 只有在单个 shard 包含多个 segments 时，才需要构建 global
ordinals，所以对于不再更新的索引，尽量将其 merge 到一个 segments，这样在配置 Fielddata circuit breaker
时只需要评估还有可能变化的索引占用的内存即可。

Fielddata circuit breaker 应该略高于 `indices.fielddata.cache.size`， 防止老数据不能从 Cache
中清除，新的数据不能加载到 Cache。

  * 4\. Accounting requests circuit breaker 配置

根据自己的数据的特点，合理评估出 Lucene 占用的内存百分比，并在此基础上上浮 5% 左右。

  * 5\. Request circuit breaker 配置

大数据量高纬度的聚合查询十分消耗内存，需要评估业务的数据量和聚合的维度合理设置。建议初始值 20% 左右，然后出现 breaker
触发时，评估业务的合理性再适当调整配置，或者通过添加物理资源解决。

  * 6\. In flight requests circuit breaker 配置

该配置涉及传输层的数据传输，包括节点间以及节点与客户端之间的通信，建议保留默认配置 100%。

### 总结

本章简要介绍了 Elasticsearch 不同版本中的 breaker 及其配置，并结合我们的经验，给出了一点配置思路。是否能合理配置 Circuit
breaker 是保证 Elasticsearch 能否稳定运行的关键， 由于 Elasticsearch 的节点恢复时间成本较高，提前触发 breaker
要好于节点 OOM。

7.x 之前的版本中，大部分的 breaker 在计算内存使用量时都是估算出来的，这就造成很多时候 breaker 不生效，或者过早介入，7.x 之后
breaker 可以根据实际使用量来计算占用空间，比较精确的控制熔断。

下一章节我们介绍 **如何对集群进行压测** ，这是业务上线的一个必经过程。

