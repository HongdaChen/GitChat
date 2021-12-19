当初要研究关联规则挖掘算法，就是为了解决 **地鼠商城购物车页** 采用协同过滤效果一般的问题，既然对 FPGrowth
算法原理已经掌握得很好了，学它当然是为了用它，那么应该如何构造数据？得到频繁模式后又如何应用到推荐系统中？在掌握了算法的使用之后，作为 IT
工作者，忍不住要一探究竟：FPGrowth 到底是如何实现的？

这些问题，将会在本篇得到答案。

本篇代码的软件环境：Spark 2.4.0。

### 一句话需求

提高地鼠商城购物车页的推荐指标。

> 需求一句话，开发把班加！

### 数据源

假设我们拥有这样一张数据表：

user | item | action_type | session_id | timestamp | day  
---|---|---|---|---|---  
用户 ID | 物品 ID | 行为类型 | 会话 ID | 行为时间戳 | 分区日期  
  
表字段说明：

  * 用户 ID：用户 ID
  * 物品 ID：物品 ID
  * 行为类型：这里需要注意一下，我们要优化的是购物车页的指标，需要找到关联的物品而不是相似的物品，所以我们只采用用户的加车、收藏和购买数据
  * 会话 ID：地鼠商城 App 的每条数据都有 session_id 这么一个字段，用户从打开 App 开始直到关闭 APP 止，session_id 是唯一的。再次打开 App 时会重新生成一个 session_id
  * 行为时间戳：顾名思义
  * 分区日期：当天的日期

关联规则挖掘需要 Transaction，所以我们就需要定义什么才算是 1 个 Transaction。

> **Transaction** ：3 天内用户的所有加车、收藏和购买行为作为 1 个 Transaction。

当然读者可能会有疑问：为什么不以一个 session_id 内用户的行为作为 1 个 Transaction，而要使用 3 天的时间呢？

这是因为地鼠商城（可以泛化到一般的电商）的用户行为特别的稀疏，更别提 加车、收藏和购买 这种又更稀疏的用户行为了，如果单单以 1 次 session_id
或者 1 天 的用户行为作为 Transaction 的话，那么有很大的可能会导致大部分的 Transaction 中含有的物品个数为 1
个，而这对于关联规则挖掘算法来说基本等同于无用数据。当然，读者们可以根据自己的数据来设定到底是用 1 周还是 3 天 还是 1 天的数据。

下面这张图展示了 Transaction 的生成过程，为了演示清楚，只用一个用户——小明：

![Transaction](https://images.gitbook.cn/4c4b5c10-2756-11eb-be76-5555cd71111f)

上图中，数字代表日期，Transaction 时间窗口设为 3
天，绿色日期代表小明在那天有加车/收藏/购买行为。将三天内的所有行为物品放在一起，就形成了一个 Transaction。

其他用户的处理逻辑同小明。

好了，Transaction 处理完毕，也就是意味着算法需要的数据准备好了，如下表所示。

| transactions | | ----------- | | A B C D E F G | | H I J K L M N | | O P Q R
S T | | U V W X Y Z |

### 运行 FPGrowth

数据存储在 HIVE 表中，表名：recsys.data_fpgrowth。

首先，读取数据：

    
    
    val data = spark.sqlContext.sql("select transactions from recsys.data_fpgrowth")
        .rdd
        .map(r => {
            /* transactions 以 空格分隔 的字符串 */
            val items = r.getAs[String]("transactions").split("\\s+")
            items
        })
    

最小支持度 minSupport 可以根据不同的需求设置不同的值，比如 transaction 有 1 千万个，如果设置 minSupport 为
1%，那么就要求有效物品的出现次数至少为 1 千万 * 1% = 10 万。这个阈值恐怕会过滤掉 90% 的物品。

所以 minSupport 的设置可以看物品出现的次数分布来确定。

地鼠商城的 minSupport 设置为 100/10000000，即十万分之一。

    
    
    val transactionsCount: Long = data.count
    val fpg = new FPGrowth()
        .setMinSupport(minSupport)
        .setNumPartitions(spark.sparkContext.defaultParallelism)
    val model = fpg.run(data) /* so easy ☆☆☆☆☆ */ 
    

瞧，我们在上一篇花费了巨大篇幅来解释 FPGrowth 原理，结果一行代码运行了起来。

其实这里的 model 最主要的是得到了频繁模式/项集，还记得 Apriori 算法那篇文章里，我们是根据什么指标去选择 X
的最关联物品的？对了，lift。

    
    
    /* itemSet 的类型是 FreqItemset，含有两个成员变量 items 和 freq */
    /* 前者对应的是这个频繁模式含有的元素，后者对应的是这个频繁模式在数据集中出现的次数 */
    /* 这里统计一下频繁模式中 1 项集元素（其实就是有效物品），及其出现的个数，不知道读者们还记不记得，计算 lift 需要它 */
    val itemFrequents = model.freqItemsets.filter(itemSet => itemSet.items.length == 1)
        .map(itemSet => (itemSet.items.head, itemSet.freq))
        .collectAsMap()
    
    /* 这里开始生成关联规则，minConfidence 和 minSupport 一样，需要根据数据去设置，地鼠商城的数据设置为 0.1 */
    model.generateAssociationRules(minConfidence)
    .filter(rule =>                                   // 1
            rule.antecedent.length == 1 &&
            itemFrequents.contains(rule.consequent.head))
        .map(rule => {
            val X = rule.antecedent.head
            val Y = rule.consequent.head
            val confidenceX2Y = rule.confidence
            val supportY = itemFrequents(Y).toDouble / transactionsCount
            val lift = confidenceX2Y / supportY
            (X, Y, lift)                         // 2
        })
        .toDF("item1", "item2", "lift")
    

说明：

1\. 这里的 rule 的类型为 Rule，含有 4 个成员变量：

  * antecedent：即 X -> Y 中的 X
  * consequent：即 X -> Y 中的 Y
  * freqUnion：(X, Y) 同时出现在数据集中的次数
  * freqAntecedent：X 出现在数据集中的次数
  * 这里之所以限定 X 长度为 1，是因为我们想得到 [啤酒 -> 尿布]，而不是 [(啤酒, 牛奶) -> 尿布]。
    * 因为线上需要根据用户的行为去推荐，比如用户对啤酒产生行为，那么我们就可以推出尿布。
    * 而 [(啤酒, 牛奶) -> 尿布] 这条关联规则要求用户同时对啤酒和牛奶都产生行为，才会推出尿布，无疑这样的用户会少很多，X 的长度越长，能命中的用户就越少。所以我们只保留 X 长度为 1 的关联规则。

2\. 这里计算 lift 的方法与 Apriori 那篇文章中是一模一样的，已经忘记的读者可以回看熟悉一下。

这段代码运行结束后，就会生成我们需要的最终的关联物品表：

item1 | item2 | lift  
---|---|---  
主物品 | 关联物品 | 提升度  
  
把这张表存储在 HBase/Redis 中，当用户的购物车/收藏/订单页中包含 item1 时，可以推荐 N 个最关联的 item2 作为推荐列表。

推荐逻辑与协同过滤如出一辙，就不赘述了。

### 源码分析

下面我们就来看看 Spark 的 FPGrowth 实现。主要看 org.apache.spark.mllib.fpm.FPGrowth，mllib
库的实现更好理解。

源码分析需要对 FPGrowth 的算法原理特别熟悉，否则代码里的逻辑不好理解，有还不清楚的读者可以再回去看看上一篇文章。

FPGrowth 执行步骤：

  * 第一步，统计物品次数，去除不满足 minSupport 的物品；
  * 第二步，构建 FP 树，根据 transactions 构建 FP 树；
  * 第三步，条件 FP 树与频繁模式，根据 FP 树生成条件 FP 树并得到频繁模式。

咱们从上面标记 ☆☆☆☆☆ 的 org.apache.spark.mllib.fpm.FPGrowth#run 函数开始。

    
    
    /* org.apache.spark.mllib.fpm.FPGrowth#run */
    /* Item 是一种数据类型，表示物品，在我们这里就是 String（物品 ID） */
    def run[Item: ClassTag](data: RDD[Array[Item]]): FPGrowthModel[Item] = {
        if (data.getStorageLevel == StorageLevel.NONE) {
          logWarning("Input data is not cached.")
        }
        val count = data.count() // transaction 数目
        val minCount = math.ceil(minSupport * count).toLong // 得到 minSupport
        val numParts = if (numPartitions > 0) numPartitions else data.partitions.length
        val partitioner = new HashPartitioner(numParts)
        /* 对应 第一步 */
        val freqItems = genFreqItems(data, minCount, partitioner)
        /* 对应 第二三步 */
        val freqItemsets = genFreqItemsets(data, minCount, freqItems, partitioner)
        new FPGrowthModel(freqItemsets)
      }
    

#### **统计物品次数**

    
    
    /* org.apache.spark.mllib.fpm.FPGrowth#genFreqItems */
    private def genFreqItems[Item: ClassTag](
          data: RDD[Array[Item]],
          minCount: Long,
          partitioner: Partitioner): Array[Item] = {
        data.flatMap { t =>
          val uniq = t.toSet // t 就是 transaction, 可以看到 transaction 里的物品/元素 必须唯一
          if (t.length != uniq.size) {
            throw new SparkException(s"Items in a transaction must be unique but got ${t.toSeq}.")
          }
          t
        }.map(v => (v, 1L)) // 这里开始统计每个物品出现的次数
          .reduceByKey(partitioner, _ + _)
          .filter(_._2 >= minCount) // 将次数小于 minSupport * #transaction 的物品过滤掉
          .collect() // 得到的结果类型是 Array[(Item, Count)]
          .sortBy(-_._2) // _2 也就是每个物品出现的次数(Count)，按照 Count 从大到小排序 
          .map(_._1) // _1 也就是 Item，也就得到了所有满足 minSupport 的物品，且按照出现次数从大到小排序
         /* 结束，与上一篇中的第一步完全一致 */
    }
    

#### **FP 树**

在进入第二三步之前，我们需要知道 FP 树 这种数据结构在 Spark 中是怎么实现的，这样才能看 FP 树和条件 FP 树的建立时不至于一头雾水。

上一篇咱们知道 FP 树是长这样的：

![](https://images.gitbook.cn/cd37d0b0-2756-11eb-b5c5-e14bfa43e9db)

org.apache.spark.mllib.fpm.FPTree，我们从上图来与代码一一对应。

  * 上图中的每个节点，由 org.apache.spark.mllib.fpm.FPTree.Node 来抽象。 Node 可以表示节点之间的双亲和孩子的关系（parent children）。
  * 上图中节点与节点之间的虚线连接的关系（并不是 sibling 的关系），由 org.apache.spark.mllib.fpm.FPTree.Summary 来抽象。

有了这两个类，一棵 FPTree 就能完整地表达出来了。

**Node**

    
    
    /* org.apache.spark.mllib.fpm.FPTree.Node */
    /* Node 类记录每个节点的信息 */
    class Node[T](val parent: Node[T]) extends Serializable {
        /* 节点中的 Item 表示，比如上图中的 B */
        var item: T = _ 
        /* 节点中 Item 在本路径中出现的次数，比如 B:5，表示路径中 B 出现了 5 次 */
        var count: Long = 0L 
        /* 记录每个节点的孩子节点，比如 B:5 节点的 children 包含了 C:3 和 D:1 */
        val children: mutable.Map[T, Node[T]] = mutable.Map.empty 
        /* 通过 parent 是否为 NULL 来判断是否根节点 */
        def isRoot: Boolean = parent == null
    }
    

**Summary**

    
    
    /* org.apache.spark.mllib.fpm.FPTree.Summary */
    /* Summary 记录每个物品在树中的连接情况 */
    private class Summary[T] extends Serializable {
        /* 物品(不是节点)在整个数据集中出现的次数，比如 C 物品出现 3 + 1 + 2 = 6 次 */
        var count: Long = 0L
        /* 链表记录本物品在树中的虚线连接情况，比如对于 C 物品，nodes 里的内容应该是 C:3 -> C:1 -> C:2 */
        val nodes: ListBuffer[Node[T]] = ListBuffer.empty
    }
    

掌握了这两个类的作用之后，我们就来正式介绍 FPTree。

**FPTree**

**add**

    
    
    /* org.apache.spark.mllib.fpm.FPTree */
    /* FPTree 最为核心的两个方法：add 和 extract */
    /* add 方法用于 FPTree 的构建 */
    /* extract 方法用于抽取前缀路径和生成条件 FP 树 */
    
    /* org.apache.spark.mllib.fpm.FPTree#add */
    // t 是一次 transaction, count 为该 transaction 出现的次数
    def add(t: Iterable[T], count: Long = 1L): this.type = {
        require(count > 0)
        var curr = root // 0
        curr.count += count // 1
        t.foreach { item =>
            val summary = summaries.getOrElseUpdate(item, new Summary) // 2
            summary.count += count // 3
            val child = curr.children.getOrElseUpdate(item, {
                val newNode = new Node(curr) // 4
                newNode.item = item
                summary.nodes += newNode // 5
                newNode
            }) 
            child.count += count // 6
            curr = child // 7
        }
        this
    }
    

add 方法实现了上一篇中“FPGrowth 算法举例第二步：构建 FP 树”，读者们还记得那组构建图吗？

下面，我们再次以具体的例子来分析 add 方法的执行过程。

假设有这样一个 transaction 数据集：

Transaction | Items  
---|---  
T1 | AB  
T2 | BC  
  
我们要将 T1 - AB 添加到 FPTree 中。

插入过程如下：

![](https://images.gitbook.cn/21681780-2757-11eb-b5c5-e14bfa43e9db)

图中 `// 0` 表示执行上述代码片段中 `// 0` 处的语句执行后的效果，`// 1`、`//2`……类似。

继续插入 T2 - BC 到 FP 树中：

![](https://images.gitbook.cn/418782d0-2757-11eb-a3fe-4f57d7580ecb)

通过这样的图文展示，聪明的你对于 FP 树建立的源码已经掌握一二了。

**extract**

通过 add 方法，FP 树已经建立了，接下来的工作就是抽取条件 FP 树了，而这正是
org.apache.spark.mllib.fpm.FPTree#extract 的职责所在。

那么，这里就交给读者一个小小的任务，结合上一篇条件 FP 树的生成步骤，自己对照着源码去分析其逻辑。

相信这并不能难倒你！

### 优化

通过运行 FPGrowth 的程序我们会发现，它会生成类似如下的关联规则：

> [AB] -> C
>
> [ABC] -> D
>
> [B] -> C
>
> ……

但是我们只想要类似 [B] -> C 这样的规则，这样在做推荐时可以很方便的根据 B 就能推出 C 了，我们当然可以通过“运行
FPGrowth”这一节中所使用的方法，过滤掉 antecedent 长度不为 1 的关联规则。

但是如果我们希望 FPGrowth 在建立条件 FP 树时就实现这样的功能，岂不美哉？那么该如何实现呢？

交给读者们去思考吧，其实并不难。

> 答案就在 extract 方法中，在建立条件 FP 树时，我们可以让递归提前结束，不让 FPGrowth 挖掘太深！

### 总结

好啦，FPGrowth 已经花了很大的篇幅，该到结束的时候了。

总结下 FPGrowth：

  * FPGrowth 只需要对数据库扫描 2 次：一次统计物品个数，一次建树。
  * 算法在运行过程中递归生成条件 FP 树，所以内存开销大。

当购物车/收藏页/订单页等场景需要做推荐时，关联规则挖掘是一件很好的工具。

> 关联规则与协同过滤：
>
>   * 关联规则关心的是 与 X 经常一起出现的物品是什么；协同过滤关心的是 与 X 相似的物品是什么。
>   * 关联规则用于用户已经购买/收藏/加购过 X 时，根据 X 去做推荐；协同过滤用于用户还在浏览 X 尚未产生购买意向时，根据 X 去做推荐。
>

FPGrowth 上线到地鼠商城的购物车/收藏/订单页之后，果然指标大涨，想来我又能休息一段时间了。

下篇，我们好好聊聊“神秘”的 embedding。

咱们，下篇见。

本篇代码地址在[这里](https://github.com/recsys-4-everyone/dist-
recsys/tree/main/fpgrowth)。

