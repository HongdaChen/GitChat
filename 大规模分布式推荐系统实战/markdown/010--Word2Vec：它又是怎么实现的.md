通过前面两篇文章，相信你已经对 Word2Vec 这种简约不简单的算法的理论有了比较透彻的理解，那么这一篇，咱们来看看理论是如何通过代码来落地的。

作者代码的 Git 地址在[这里](https://github.com/tmikolov/word2vec)，是从 C 写的，这篇文章剖析的是 Spark
实现，逻辑与 C 的实现完全一致，只不过是分布式版本。Spark
版本的地址在[这里](https://github.com/hibayesian/spark-word2vec)，之所以没有用 Spark 官方的
Word2Vec 实现，是因为此版本实现了 Negative Sampling，而官方的版本是基于 Huffman Tree。但是这个版本有少许
Bug，我也对这些 Bug 进行了修复，本文中的代码在[这里](https://github.com/recsys-4-everyone/dist-
recsys/tree/main/word2vec)。

> 代码中关于 Huffman Tree 的部分均被自动忽略。

### 代码逻辑

首先来理一下 Word2Vec 源码的逻辑，了解 Word2Vec 原理的同学现在看到这张思维导图时，应该会心一笑——Nothing Special!

![](https://images.gitbook.cn/dfd1e5e0-2764-11eb-b804-ef8651c2512a)

瞧，代码里几乎每一步都是我们熟知的，但是在实现上，作者也是花了很多心思，考虑了诸多性能问题，这在本文的后面会谈到。

大体的逻辑如此，我们来看看作者如何将理论落地的，再回顾下 Word2Vec 的数据格式。

一个个句子由空格分隔的词组成：

sentence  
---  
词 11 词 12 词 13 …… 词 1N  
词 21 词 22 词 23 …… 词 2M  
  
### 参数说明

代码中的模型超参如下：

  * stepSize：学习率
  * minCount：词的最小出现次数，出现次数小于 minCount 的会被过滤掉
  * maxIter：训练迭代次数
  * numPartitions：Spark 的分区数
  * seed：产生随机数的种子
  * vectorSize：词向量长度
  * windowSize：滑动窗口大小，同一个 window 里的词处在同一个上下文中
  * maxSentenceLength：句子的最大长度，超过词长度的句子会被截断
  * cbow：模型结构是 CBOW 还是 Skip-Gram，1 表示 CBOW，0 表示 Skip-Gram
  * hs：按照 Huffman 树来训练模型，如果设置了 hs > 0，就不能设置 negative，本文只考虑为 0 的情况
  * negative：按照负采样的方式来训练模型，设置为 0 表示不采用负采样，大于 0 表示负采样的个数
  * sample：Sub Sampling 对应的概率值

Word2Vec 运行程序如下：

    
    
    import org.apache.spark.ml.embedding._
    
    val word2Vec = new Word2Vec()
          .setInputCol("sentence")   // 数据表中对应句子的列名
          .setVectorSize(vectorSize)
          .setMaxIter(maxIter)
          .setNegative(5)
          .setHS(0)
          .setSample(1E-3)
          .setMaxSentenceLength(1000)
          .setNumPartitions(200)
          .setSeed(77L)
          .setStepSize(0.025)
          .setWindowSize(windowSize)
          .setMinCount(minCount)
    
    val model = word2Vec.fit(document) // document 就是数据集，只有一列，名为 "sentence"
    

由此可以看出，最主要的还是这个 fit 方法。

接下来，我们就按照思维导图上的逻辑顺序，对着源码来一步步探究 fit 方法到底做了什么。

fit 方法完整路径：

    
    
    org.apache.spark.mllib.embedding.Word2Vec#fit
    

### 为每个词编号

进入 fit 后，引入眼帘的便是 learnVocab 方法，此方法的作用就是这一小节的小标题——为每个词编号。

    
    
    // S 是数据类型，可以简单地理解成一个字符串数组，表示一个句子
    private def learnVocab[S <: Iterable[String]](dataset: RDD[S]): Unit = {
      val words = dataset.flatMap(x => x) // RDD[句子] 变成 RDD[词]
    
      vocab = words.map(w => (w, 1))
        .reduceByKey(_ + _) // spark word count ... 统计每个词出现的次数
        .filter(_._2 >= minCount) // 过滤掉次数小于 minCount 的词
        .map(x => VocabWord( // 将词和其对应的次数组装到一类里，便于管理
          x._1, // 词
          x._2, // 词的次数
          new Array[Int](MAX_CODE_LENGTH), // 忽略不计，用于 Huffman 树训练
          new Array[Int](MAX_CODE_LENGTH), // 同上
          0)) // 同上
        .collect()
        .sortWith((a, b) => a.cn > b.cn) // 按照词出现的次数降序排列
    
      vocabSize = vocab.length // vocab 中的元素此时是没有重复的，length 得到词汇表大小
      require(vocabSize > 0, "The vocabulary size should be > 0. You may need to check " +
        "the setting of minCount, which could be large enough to remove all your words in sentences.")
    
      var a = 0
      while (a < vocabSize) { // 为每个 word 分配编号
        vocabHash += vocab(a).word -> a // vocabHash 存储了 word 到编号的映射关系
        trainWordsCount += vocab(a).cn // 顺便统计下数据集中一共有多少词（包含重复的）
        a += 1
      }
      logInfo(s"vocabSize = `$vocabSize, trainWordsCount = $`trainWordsCount")
    } // 结束，是不是 so easy ...
    

思维导图中的第 1 步就完成了。

### 变量、参数初始化

fit 方法再调用 learnVocab 完毕之后，继续往下走，会看到 createBinaryTree 方法，同样，我们对忽略之，因为这只有在采用
Huffman 树训练时才用的上。

继续走，看到这样一行代码：

    
    
    if (negative > 0) { // 如果采用负采样的方式训练，嗯，那我们会进到这个分支里来，进去看看
      initUnigramTable() // 初始化 UnigramTable(?), UnigramTable? 这是啥...
    } else if (hs == 0) {
      throw new RuntimeException(s"negative and hs can not both be equal to 0.")
    }
    

还记得负采样的概率公式吗？再次贴出来唤醒大家的记忆：

$$P(w_i) = \frac{f(w_i)^{3/4}} {\sum_{j=0}^Vf(w_j)^{3/4}}$$

UnigramTable 作用正在于此：生成负采样的概率表。我们来看看它到底是干什么的。

#### **负采样概率表**

    
    
    private def initUnigramTable(): Unit = {
      var a = 0
      val power = 0.75
      var trainWordsPow = 0.0
      table = new Array[Int](tableSize) // tableSize 大小为 1E8 也就是 1 亿
    
      while (a < vocabSize) {
        trainWordsPow += Math.pow(vocab(a).cn, power) // trainWordsPow (1) 式中的分母
        a += 1
      }
    
      var i = 0
      // 第一个词的负采样概率，因为 vocab 是按照 cn 降序排列的，所以第一个词的概率最大
      var d1 = Math.pow(vocab(i).cn, power) / trainWordsPow 
      a = 0
      // 下面这个 while 循环中的逻辑，我们用一个具体的例子来说明
      while (a < tableSize) { 
        table(a) = i
        if (a.toDouble / tableSize > d1) { // 1
          i += 1
          d1 += Math.pow(vocab(i).cn, power) / trainWordsPow // 2
        }
        if (i >= vocabSize) {
          i = vocabSize - 1
        }
        a += 1
      }
    }
    

上面这段代码片段的核心在于 while 循环，先把代码丢到一边，看看下面这张图：

![](https://images.gitbook.cn/8e932440-2765-11eb-9825-558323e50dbd)

> 问：随机从这 10 个方格中抽取 1 个元素，抽到 A 的概率是多少？
>
> 答：0.4。
>
> 问：抽到 B 的概率呢?
>
> 答：0.3。

假设上图中的 A、B、C、D 就是词，在数据集中出现的次数分别是 4、3、2、1，tableSize 设为 10。

则上述片段中 while 之前各变量的值如下：

> trainWordsPow = $4^{0.75} + 3^{0.75} + 2^{0.75} + 1^{0.75}$ =
> 7.789727012208396
>
> d1 = 2.8284271247461903
>
> tableSize = 10
>
> vocabSize = 3

那么，下面问题的答案又是多少呢？

> 问：想让 A 被抽到的概率等于 $4^{0.75} / (4^{0.75} + 3^{0.75} + 2^{0.75} + 1^{0.75}) =
> 0.363$，那么 A 必须在 table 里出现多少次呢？
>
> 答：这个问题有点侮辱我的智商啊，当然是 $\lceil 0.363 * 10 \rceil = 4$ 啦。如果 tableSize = 1000，那么
> A 要出现 363 次。

现在，能理解 while 循环中的逻辑了吗？

是的，就是将 词 $w$ 在 table 中重复 $m$ 次，使得 $m = \lceil tableSize * \frac{f(w)^{3/4}}
{\sum_{j=0}^Vf(w_j)^{3/4}} \rceil $，这是通过 $a.toDouble / tableSize > d1$
来实现的，当该条件不满足时，词 $w$ 一直在 table 中重复，一旦条件满足，则换成下一个词，继续重复。

> a / tableSize 可以看作均匀分布在 a 处的累积概率，d1 可以看作负采样概率分布在 i 处的累积概率。

通过这种方式生成负采样概率表 table 之后，随机从 table 表中抽出词 $w$，其概率为 $ \frac{f(w)^{3/4}}
{\sum_{j=0}^Vf(w_j)^{3/4}}$，满足要求！

#### **sigmoid 函数优化**

sigmoid 函数如下：

$$sigmoid(x) = \frac{1}{1+e^{-x}}$$

这个公式虽然写起来很简单，用起来也很简单，但是跑起来就不简单了，因为 $e$ 这个无理数的存在，导致计算机在计算时的性能差点意思。

$$sigmoid(x) = \frac{1}{1+e^{-x}} \\\linear(x) = \frac{1}{1+x}$$

上述两个函数，各循环执行 1000000 次，sigmoid 耗时 260 毫秒，linear 耗时 20 毫秒。（不同的机器可能耗时不一样）

作者为了提高 sigmoid 的性能，提前计算好 $x \in [-6,6]$ 对应的所有结果。优化思路如下。

1\. 将 [-6, 6] 平均分成 1000 等份，每等份编号 0 到 999，使用 expTable（数组结构）存储 $x$ 对应的 sigmoid
值，$x$ 与其对应的编号 $i$ 的关系为：

$$\begin{align}x &= \frac {6 - (-6)} {1000} i - 6 = (\frac{2i}{1000} - 1)
\times 6 \\\i &= \lfloor(x + 6) \times \frac{1000}{2 \times 6} \rfloor
\\\expTable[i] &= \frac{1}{1+e^{-x}} \end{align}$$

2\. 优化后的 sigmoid 伪代码如下:

    
    
    optimizedSigmoid(x) = {
      if(x > 6) return 1;
      if(x < -6) return -1;
      i = int((x + 6) * 1000/(x * 6));
      return expTable[i];
    }
    

瞧，经过这一系列变换，sigmoid 函数的时间复杂度也变成 O(1) 了。

#### **初始化 W 和 W'**

    
    
    val syn0Global =
      Array.fill[Float](vocabSize * vectorSize)((initRandom.nextFloat() - 0.5f) / vectorSize) // W,初始化参数为[-0.5,0.5]的均匀分布
    val syn1Global = new Array[Float](vocabSize * vectorSize) // 忽略,用于 Huffman 树训练
    val syn1NegGlobal = new Array[Float](vocabSize * vectorSize) // W',初始化参数为 0
    

好了，至此一些准备工作就做完了，接下来就进入到了训练阶段了。

### 训练、保存模型

#### **Sub Sampling**

首先看看，如何对高频词进行 Sub Sampling：

    
    
    var sentencePosition = 0
    while (sentencePosition < sentence.length && sentencePosition < MAX_SENTENCE_LENGTH) {
      val word = sentence(sentencePosition) // 取出当前词
      if (sample > 0) { // 进行 sub sampling
        val ran = (Math.sqrt(bcVocab.value(word).cn / (sample * trainWordsCount)) + 1) *
          (sample * trainWordsCount) / bcVocab.value(word).cn // 这个公式眼熟不?
        if (ran >= random.nextFloat()) { // ran 是该词被保留的概率
          sen(sentenceLength) = word
          sentenceLength += 1
        } // ran 小于随机数，则丢弃该词
      } else { // 不做 Sub Sampling，保留句子中的每个词
        sen(sentenceLength) = word
        sentenceLength += 1
      }
      sentencePosition += 1
    }
    
    
    
    if (wordCount - lastWordCount > 10000) {
      lwc = wordCount
      alpha =
        learningRate * (1 - numPartitions * wordCount.toDouble / (trainWordsCount + 1))
      if (alpha < learningRate * 0.0001) alpha = learningRate * 0.0001
      logInfo("wordCount = " + wordCount + ", alpha = " + alpha)
    } // 这里是每个 Spark 分区，每训练 10000 个词，学习率就减小
    

#### **参数更新**

在看参数更新这部分之前，有必要将上一篇文章中的符号与代码中的变量名称对应起来。

回顾一下上一篇的部分内容，输出词的预测概率：

$$p(w_j|w_0) = σ(v'^T_{w_j} \cdot v_{w0}) \in(0,1) \tag{1}$$

输出层到隐藏层的参数更新：

$$\begin{align}\frac{\partial E} {\partial v'_w} &=
\begin{cases}(\sigma(v'^T_{w_{o,c}} \cdot v_{w_0}) - 1)v_{w_0}, & w = w_{o,c}
\\\\[1ex](\sigma(v'^T_w \cdot v_{w_0}) - 0)v_{w_0}, & otherwise\end{cases}
\tag{2}\\\ \end{align}$$

$$v'^{new}_w = v'^{old}_{w} - \eta\frac{\partial E} {\partial v'_w} \tag{3}$$

隐藏层到输入层的参数更新：

$$\begin{align}\frac{\partial E} {\partial v_w} &=
\begin{cases}(\sigma(v'^T_{w_{o,c}} \cdot v_{w_0}) - 1)v'_{w_{o,c}}, & w =
w_{o,c} \\\\[1ex](\sigma(v'^T_w \cdot v_{w_0}) - 0)v'_{w}, &
otherwise\end{cases} \\\\\tag{4}\end{align}$$

$$v^{new}_w = v^{old}_{w} - \eta\frac{\partial E} {\partial v_w} \tag{5}$$

其中 $w_0$ 是输入词， $w_{o,c}$ 是上下文词，也即周围词，其他的 $w$ 均为负采样词。

下面的代码片段已经删除了不必要的 CBOW 和 Huffman 的训练部分，只保留 Skip-Gram 和 Negative Sampling。

    
    
    sentencePosition = 0
    while (sentencePosition < sentenceLength) {
      val word = sen(sentencePosition) // 当前词
      val b = random.nextInt(window) // 窗口
      val neu1 = new Array[Float](vectorSize) // negative sampling
    
      // Train Skip-gram
      var a = b
      while (a < window * 2 + 1 - b) {
        if (a != window) {
          // window 的设计：取输入词左边 window - b 个，右边 window + b 个
          // 而不是我们常见的：输入词左边右边各 window 个
          val c = sentencePosition - window + a 
          if (c >= 0 && c < sentenceLength) {
            val lastWord = sen(c) // 上下文词，对应上述公式(2)(3)中的 w(o,c)
            val l1 = lastWord * vectorSize // 上下文词向量的起始位置在数组中的偏移量
            val neu1e = new Array[Float](vectorSize) // 对应公式 (4)
    
            if (negative > 0) {
              // Negative sampling
              var target = -1
              var label = -1
              var d = 0
              while (d < negative + 1) {
                var isContinued = false
                if (d == 0) {
                  target = word
                  label = 1 // (上下文词, 输入词, 1)
                } else {
                  target = bcTable.value(random.nextInt(tableSize)) // 负采样随机抽取
                  if (target == word) {
                    isContinued = true
                  }
                  if (!isContinued.equals(true)) {
                    label = 0 // (上下文词, 负采样词, 0)
                  }
                }
                if (!isContinued.equals(true)) {
                  val l2 = target * vectorSize // 目标词（输入/负采样词）在数组中的偏移量
                  val f = blas.sdot(vectorSize, syn0, l1, 1, syn1Neg, l2, 1) // 公式(1)
                  var g = 0.0
                  // 优化后的 sigmoid 计算方法 ===begin===
                  if (f > MAX_EXP) {
                    g = (label - 1) * alpha 
                  } else if (f < -MAX_EXP) {
                    g = (label - 0) * alpha
                  } else {
                    val ind = ((f + MAX_EXP) * (EXP_TABLE_SIZE / MAX_EXP / 2.0)).toInt
                    g = (label - expTable.value(ind)) * alpha
                  }
                  // 优化后的 sigmoid 计算方法 ===end===
                  blas.saxpy(vectorSize, g.toFloat, syn1Neg, l2, 1, neu1e, 0, 1) // 在 v'w 更新之前先计算公式(4)
                  blas.saxpy(vectorSize, g.toFloat, syn0, l1, 1, syn1Neg, l2, 1) // 公式(3), 更新 v'w
                  syn1NegModify(target) += 1
                }
                d += 1
              }
            }
    
            syn0Modify(lastWord) += 1
            blas.saxpy(vectorSize, 1.0f, neu1e, 0, 1, syn0, l1, 1) // 公式(5)，完成一次词向量的更新
          }
        }
        a += 1
      }
      sentencePosition += 1
    }
    

### 小结

看完了源码觉得如何？

当初我在看的时候，给我的第一反应就是，理论和落地差别太大了。在实现理论时有太多的性能方面的因素需要考虑。

不管怎么说，Word2Vec 绝对是一个值得仔细咀嚼的算法，这其中的：

  * 多分类转二分类
  * 负采样（Negative Sampling）、下采样（Sub Sampling）技术
  * 工程上的优化等

都是作为 Coder 来说很宝贵的学习对象。

希望亲爱的读者们好好阅读 Word2Vec 源码，不管是 C 的版本还是 Spark 版，其本质是相同的。

到现在，我们也终于将 Word2Vec 刨根问底，翻了个底朝天。可是，我们好像一直没说一个问题——它到底怎么用？词向量怎么用在电商里呢？

不要着急，希望大家在吃透词向量 1、2、3 之后，再去阅读《词向量 4：应用》。

下篇见。

