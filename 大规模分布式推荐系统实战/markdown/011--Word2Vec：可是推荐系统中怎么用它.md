理论讲完了，源码解析也结束了，终于到了用它的时候了。铺垫了这么一大堆，终归是为了让它派上用场。

在推荐系统中，Word2Vec 一般会用在“找相似”中——找到与用户行为商品相似的商品，比如，用户点了商品 A，那么就给该用户推荐与商品 A 相似的商品。

那么，相似商品从哪里来呢？

是的，相似商品就根据 Word2Vec 给每个商品生成的向量计算得来。

### 数据源

电商中的 Word2Vec 一般会被称为 Item2Vec，顾名思义，就是将一个个商品表征成一个个向量。

Word2Vec 以句子（sentence）为单位，去学习其组成元素（word）的向量。而运用在电商系统中，将用户在一段时间内的足迹（访问轨迹）作为一个
sentence，而组成 sentence 的一个个 商品 ID 作为 word。这样便会得到如下数据集：

用户 | 访问轨迹  
---|---  
用户 1 | 商品 ID11, 商品 ID12,…, 商品 ID1N  
用户 1 | 商品 ID21, 商品 ID22,…, 商品 ID2M  
…… | ……  
用户 U | 商品 IDU1, 商品 IDU2,…, 商品 IDUK  
  
注：一个用户的轨迹出现在多行数据中是很正常的，比如前后行为时间相差十天半个月，这两个行为就不应该属于同一个“句子”。

用户的访问轨迹必须要按照时间顺序从先到后排列，因为 Word2Vec 的训练过程其实是与词的顺序是很大的关系。

如果大家还记得第 7 篇关于 FpGrowth 算法的数据源生成逻辑的话，应该会觉得 Word2Vec
数据源生成方式与之有点类似，但是又不完全一样，希望你可以从中得到启发。

训练数据的生成直接决定了算法结果的质量，所以不同的业务一定要按照自己的规则/经验去设置相应的超参、聚合相应的数据，生成训练数据是整个算法最重要的一步，请务必重视。

### 运行 Item2Vec$^1$

默认超参如下：

    
    
    vectorSize = 100  // 商品向量长度
    learningRate = 0.025 // 学习率 
    numPartitions = 1 // spark 分区数
    numIterations = 1 // 训练迭代次数
    minCount = 5 // 商品在数据集中出现的最少次数，少于此则过滤掉
    maxSentenceLength = 1000 // 用户行为轨迹最长保留 1000 个
    window = 5 // 窗口大小，也就是 中心词 左边 5 个，右边 5 个
    negative = 5 // 负采样个数
    sample = 1E-3 // 下采样概率（到现在应该分得清 Sub Sampling 和 Negative Sampling 了）
    

一般需要根据自己的数据集修改的超参如下：

    
    
    vectorSize = 100  // 商品向量长度，数据量大时可以设置大一点 
    numPartitions = 1 // spark 分区数，看数据量大小设置
    maxIter = 1 // 训练迭代次数，看数据量大小设置，可以初始设为 10
    minCount = 5 // 商品在数据集中出现的最少次数，根据数据集中商品次数的分布情况来确定
    negative = 5 // 负采样个数，根据作者的 paper（distributed representations of words and phrases and theircompositionality）描述，大数据量可以设置为 2~5，小数据量可以设置 5~20
    

运行 Word2Vec 的代码也超级简单，寥寥几行代码就跑起来了$^2$。

    
    
    val corpus = spark.sqlContext.sql("select user_id, item_ids from rec.data_w2v")
    val trainingData = corpus.map(r => r.getAs[String]("item_ids").split(",")).toDF("ids")
    
    //Learn a mapping from words to Vectors.
    val word2Vec = new Word2Vec()
      .setInputCol("ids") 
      .setVectorSize(vectorSize) 
      .setMaxIter(maxIter) 
      .setNegative(negative)
      .setHS(0) 
      .setSample(sample) 
      .setMaxSentenceLength(maxSentenceLength) 
      .setNumPartitions(numPartitions) 
      .setStepSize(learningRate) 
      .setWindowSize(windowSize) 
      .setMinCount(minCount)
    
    val model = word2Vec.fit(trainingData)
    
    val w2v = model.getVectors.rdd
      .map(r => {
        val word = r.getAs[String]("word") // 词，此处就是 itemId
        val vector = r.getAs[org.apache.spark.ml.linalg.Vector]("vector").toDense // embedding
        (word, vector)
      })
    

这就得到了每个商品的 embedding，是不是直呼 so easy？

确实，每种算法最耗时的在于数据分析和数据处理，真正产出代码所花费的时间可能只占了整个开发过程的 5%~10%。

现在有了商品的 embedding 了，那么怎么用来找每个商品的相似商品呢？

Bug：

1\. Spark Word2Vec 代码 Bug，参见
[JIRA](https://github.com/apache/spark/commit/6dff114ddb3de7877625b79bea818ba724ccd22d)，Bug
会导致词向量中的元素有时候会产生巨大的值从而导致这个向量根本没法使用，解决办法就是在计算过程中对词向量进行归一化，这个 Bug 在 Spark 2.4
中解决了。

2\. 使用 Negative Sampling 方式训练的 Spark Word2Vec 代码小
Bug，[地址在此](https://github.com/hibayesian/spark-
word2vec/blob/master/src/main/scala/org/apache/spark/ml/embedding/Word2Vec.scala)，注意到

    
    
    final val negative = new IntParam(this, "negative", "Use negative sampling method to train the model", ParamValidators.inArray(Array(0, 1)))
    

作者限定了 Negative 取值只能是 0 和 1，应该把这个限制条件去掉。因为这个参数意思是负样本的个数，并不仅仅是“是否采用 Negative
Sampling 训练模型”的意思。

上述 Bug 均已经在本文的代码中解决掉了。

### 找相似

一般有几种衡量相似度的指标——Jaccard，欧式距离，夹角余弦等。在计算向量的相似度时，通常采用夹角余弦 Cosine Similarity。

向量 $\vec{x}$ 与向量 $\vec{y}$ 的余弦相似度公式为：

$$cosine\ similarity<\vec{x}, \vec{y}> = \frac{\vec{x} \cdot \vec{y}}{\lvert
\vec{x} \rvert \lvert \vec{y} \rvert}$$

所以对于 M 个商品，要求每个商品的 topN 个最相似的商品，伪代码如下：

    
    
    # start
    map = {}
    for i from 1 to M:
      heapI: 大小为 N 的小顶堆
      for j from i+1 to M:
        if i == j: 
          continue
        similarityIJ = cosine similarity(vectorI, vectorJ)
        if heapI is not empty and similarity > heapI.head:
          headI.dequeue()
          headI.enqueue(j)
        else:
          heapI.enqueue(j)
      end for
      map[i] = heapI
    end for
    # end
    

很容易知道，其算法时间复杂度 $O(M^2)$：

    
    
    import org.apache.spark.ml.embedding.Word2VecModel
    import org.apache.spark.ml.linalg.Vectors
    import org.apache.spark.sql.{DataFrame, SparkSession}
    
    import scala.collection.mutable
    
    object Word2Vec {
      private val topN = 100
      private val ordering: Ordering[(String, Double)] = new Ordering[(String, Double)] {
        override def compare(self: (String, Double), that: (String, Double)): Int = {
          that._2.compareTo(self._2)
        }
      }
    
      def cosineSimilarity(spark: SparkSession, model: Word2VecModel): DataFrame = {
        val w2v = model.getVectors.rdd
          .map(r => {
            val word = r.getAs[String]("word")
            val vector = r.getAs[org.apache.spark.ml.linalg.Vector]("vector").toDense
    
            (word, vector)
          }).collect()
    
        val broadcast = spark.sparkContext.broadcast(w2v)
    
        import spark.implicits._
    
        model.getVectors.rdd.map(r => {
          val word = r.getAs[String]("word")
          val vector = r.getAs[org.apache.spark.ml.linalg.Vector]("vector").toDense
          (word, vector)
        }).flatMap { case (word1, vec1) =>
          val heap = new mutable.PriorityQueue[(String, Double)]()(ordering) // 构造小顶堆
          broadcast.value.filter { case (word2, _) => word2 != word1 }
            .foreach { case (word2, vec2) =>
              val dotProduct = vec1.toArray.zip(vec2.toArray).map { case (v1, v2) => v1 * v2 }.sum
              val norms = Vectors.norm(vec1, 2) * Vectors.norm(vec2, 2)
              val thisSim = math.abs(dotProduct) / norms
              if (heap.size < topN) heap.enqueue((word2, thisSim))
              else {
                heap.head match {
                  case (_, minSim) if minSim < thisSim =>
                    heap.dequeue()
                    heap.enqueue((word2, thisSim))
                  case _ => // 啥也不做
                }
              }
            }
    
          heap.toArray.map { case (word2, sim) => (word1, word2, sim) }
        }.toDF("item1", "item2", "sim")
      }
    }
    

### 应用

得到了商品相似度表之后，相信聪明的你已经看出来了。是的，与前几篇文章中说到的 **协同过滤** 、 **关联规则** 一样，根据用户最近的 N
个历史行为（这里的行为可以是点击、加车、收藏和购买等）商品，查表（Redis/HBase……）得到了相似商品，作为召回集的一部分加入到用户的推荐列表中。

这部分就不再赘述了。

### 小结

至此，关于 Word2Vec
就聊得差不多了，上线后，在地鼠商城的推荐系统中表现良好，不仅能推荐出很相似的商品，甚至有时候会有点小惊喜，比如点击奶粉会推玩具这样让人意想不到的结果。指标上去了，BOSS
很开心，算法的同学也学到了很多知识，每个人都很 Happy。

总结一下，Word2Vec 是一个简约不简单的算法，模型结构异常简单，但是其背后所蕴含的智慧值得每个算法人员的研究。

  * Negative Sampling：负采样的概率计算，很有启发性，在以后的工作中如果遇到海量求 softmax 我们是不是也可以考虑使用这种技术呢？ 
  * Sub Sampling：下采样的概率计算。
  * 参数的更新，包括 $W$ 和 $W'$ 的更新方式，重中之重。
  * 性能、性能、性能：再怎么强调也不为过，包括作者实现的 sigmoid 的近似处理以及在实现负采样时采用的概率 table 都是很巧妙的设计。当我们在海量的数据中徜徉的时候，速度是我们第一考虑的因素，如果一个模型很厉害，但是训练速度像乌龟一样慢，这是不能接收的。
  * 将商品 ID 作为词，用户浏览轨迹作为文档，采用 Item2Vec 的方式训练出每个商品对应的 embedding。$^3$

好了，Word2Vec 就到这里了 (?) 想得美，难道就没有要改进的地方吗？有，肯定有。

### 优化

抛开 Word2Vec 训练过程中的超参调优这种模型的优化，Word2Vec 在应用过程中的优化空间也是非常大的。

首先，在“找相似”这一小节，会惊人的发现，计算相似度的时间复杂度居然达到了
$O(M^2)$，试问谁能接收如此慢的运行时间。假如商品个数达到百万千万级别级别，那么仅仅找相似这一步会耗费漫长的时间。有没有可能让这一步更快呢？而且一旦有新商品进来的话，
**找相似** 这一步好像要重新跑一遍，想想都觉得心累。

再者，在“应用”这一小节，我们需要查表获得行为商品的 topN 个相似商品作为用户的推荐列表，那么有没有可能摆脱这张相似商品表呢？

关于上面两个问题，都是有答案的，有经验的读者应该立刻就想到了
ANN——近似最近邻。是的，这种技术专门运用于给定某个向量，求与其尽可能最相似的向量。可以让我们 **找相似**
这一步更加的迅速，也可以让我们摆脱商品相似表，同时也支持动态增加、删除节点。

那么，这种技术到底是什么样的呢？

下一篇，让我们先来介绍一个最最基础的 ANN 算法——LSH，局部敏感哈希。

下篇见。

* * *

注释 1：所有 Word2Vec 理论、代码只考虑基于 Negative Sampling 的 Skip-Gram

注释 2：完整代码详见 [GitHub](https://github.com/recsys-4-everyone/dist-
recsys/tree/main/word2vec)

注释 3：在原生的 Word2Vec 基础上，又进化出了 node2vec 和 Graph Embedding 等。有兴趣的同学可以一探究竟。

