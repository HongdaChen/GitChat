Learning To Rank（LTR、L2R），翻译过来就是 **排序学习**
。当我初次接触到它的时候，就充满了强烈的好奇心：它与普通的建模方式有何不同？它怎么学习先后顺序的？背后的理论基础是不是门槛很高啊？它怎么预测呢？损失函数又是什么？评测指标是不是
NDCG？（根据上一篇的内容，有了 **预测函数** 和 **损失函数** ，基本上模型就有“学习”的能力了）。

本篇，咱们就来看看利用 Listwise 方法（Listwise Approach）的 ListNet$^1$ 是如何实现 Learning To Rank
的。

想了解 Pointwise 和 Pairwise 的同学可以查阅有关文献$^2$，本文的重点在 Listwise。前两者可以认为是 Listwise
的特殊形式：当 list 的长度是 1 时就是 Pointwise，长度是 2 时就是 Pairwise。

### 什么是 Listwise

首先介绍两个概念：pv id 和 relevance。

#### **PV ID**

Page View 简称 PV，表示一次页面浏览，PV ID 就是用来标识某次 PV 事件，它是唯一的，翻页时 PV ID
会发生变化。如下图所示，假设用户打开了首页-猜你喜欢场景，看到了 ABCDEF 6 个物品，那么这就是一次 PV。当该用户往上滑动屏幕发生 **翻页**
时，可能又看到 GHIJ 4 个物品，此时的 PV ID 则会发生变化。

![](https://images.gitbook.cn/393e0e70-58e7-11eb-8e97-939f5104d3bb)

#### **Relevance**

Relevance 在 **第 17 篇** 讲解 NDCG 时已经做了说明，在这里再回忆一下。

Relevance
常用在信息检索中，表示当用户输入一个检索条件时，返回的检索结果与检索条件的匹配相关程度，一般值越大相关程度越高。扩展到推荐系统中，Relevance
可以用来表示用户对物品的喜好程度，比如上图左边，用户在一次 PV 中看到了 ABCDEF 6 件物品，但是点击了 D，那么可以认为在该用户眼里，D 的
Relevance 是高于其他物品的。

在电商中，一般会将 Relevance 划分为：

  * 0：曝光未点击
  * 1：点击
  * 2：加车/收藏
  * 3：下单/购买

如果是视频/文章领域，则可以划分为：

  * 0：曝光未点击
  * 1：播放/浏览
  * 2：点赞/收藏
  * 3：下载/分享

不同的业务对应不同的划分方法，不过本质上都遵循相同的标准：用户行为意图越明显，则分值越高。

#### **Listwise**

那么所谓的 Listwise Approach 到底怎么理解呢？

一般情况下，一条数据就是一条训练实例（instance），但是在 Listwise LTR 中，一个 PV ID
下的所有数据才构成一条训练实例，也就是说训练时它的输入是一个 List，因此得名。如下图所示，标明了 Listwise LTR 的 **一条**
输入数据对应的输出，其中的 Relevance 正是我们希望预测的 label。

![](https://images.gitbook.cn/7aed16e0-58e7-11eb-8ca0-ad2e0df56a32)

模型的输入为一个 PV ID 下的所有样本，这些样本构成了一条训练实例。实例中每条样本都有一个
relevance（label），特征（features）经过模型之后都会得到一个预测得分（score），所以每条训练实例的预测输出也是一个
List，List 中含有预测值（scores）和真实值（relevances）。问题随之而来，如何设计一个损失函数，可以通过 **这个预测 List**
计算出损失值从而进行梯度更新实现模型的“学习”功能呢？

这也正是文献$^1$ 的精华所在，它将 score 和 relevance
与概率联系了起来，然后通过概率分布去计算损失。接下来，我们先来讲述这种联系是如何做到的，论文中提到了两种方式：permutation probability
和 top one probability。

上图中的模型输入，其实与普通模型的输入别无二致，只不过我们将同一个 PV ID 下的数据集中在了一起进行训练，让模型更好的捕获用户的意图。

### Permutation probability

先提一个问题：红黄蓝三个球，按顺序一字排开，请问有多少种排法？每种排法的概率是多少？

首先解答有多少种排法：很简单，$A_3^3$，$3!$。

每种排法的概率：也很简单，每种排法的概率都是 $\frac{1}{3!}=\frac{1}{6}$。

如果现在题目做稍微的改动：还是红黄蓝三个球排列，但是每个球都有了分值，红色球的分值为 1.5，黄色球的分值为 1.0，蓝色球的分值为
0.5，此时每种排法的概率又是多少呢？

假设 $\pi$ 是 $n$ 个元素全排列中的一个排列，$\phi(·)$ 是正的单调递增函数。如果 $n$ 个元素都有各自的分值（score），那么排列
$\pi$ 出现的概率为：

$$P_s(\pi) =
\prod\limits_{j=1}^n\frac{\phi(s_{\pi(j)})}{\sum_{k=j}^n\phi(s_{\pi(k)})}
\tag{1}$$

式中，$s_{\pi(j)}$ 是排列 $\pi$ 中第 $j$ 个位置的元素对应的 score，计算出所有的排列出现的概率就得到了 permutation
probability。

现在回到刚才的问题，红黄蓝三个球的排法依然不变，还是 $A_3^3$，全排列如下所示：

排列 | 组合  
---|---  
$\pi_1$ | ![](https://images.gitbook.cn/cc7b3b90-58e7-11eb-86e7-51557e1a6508)  
$\pi_2$ | ![](https://images.gitbook.cn/d3c8e460-58e7-11eb-86e7-51557e1a6508)  
$\pi_3$ | ![](https://images.gitbook.cn/da9ffda0-58e7-11eb-b90b-2f717fa43dd4)  
$\pi_4$ | ![](https://images.gitbook.cn/e18f32c0-58e7-11eb-90b2-f32bf5b015be)  
$\pi_5$ | ![](https://images.gitbook.cn/e86d9f00-58e7-11eb-90b2-f32bf5b015be)  
$\pi_6$ | ![](https://images.gitbook.cn/f02183b0-58e7-11eb-9c81-71e9be275b25)  
  
以 $\pi_1$ 为例，根据公式 $(1)$
来计算其出现的概率（$s_{红球}=1.5$，$s_{黄球}=1.0$，$s_{蓝球}=0.5$，$\phi(x) = e^x$）。

$$\begin{aligned}P_s(\pi_1) &=
\prod\limits_{j=1}^3\frac{\phi(s_{\pi_1(j)})}{\sum_{k=j}^3\phi(s_{\pi_1(k)})}
\\\&=\frac{e^{s_{红球}}}{e^{s_{红球}}+e^{s_{黄球}}+e^{s_{蓝球}}} \times
\frac{e^{s_{黄球}}}{e^{s_{黄球}}+e^{s_{蓝球}}} \times \frac{e^{s_{蓝球}}}{e^{s_{蓝球}}}
\\\&= 0.3153\end{aligned}$$

    
    
    # python 3.6
    # 公式(1) 对应的代码实现 
    import math
    
    def permutation_probability(scores):
        sum_exp_score = 0
        probability = 1
        for score in reversed(scores):
            cur_exp_score = math.exp(score)
            sum_exp_score += cur_exp_score
            probability *= (cur_exp_score / sum_exp_score)
        return probability
    

同理，可以得到所有排列对应的概率：

排列 | 组合 | 概率  
---|---|---  
$\pi_1$ | <img
src="https://images.gitbook.cn/cc7b3b90-58e7-11eb-86e7-51557e1a6508"
style="zoom: 67%;" /> | $0.3153$ |  |  
$\pi_2$ | <img
src="https://images.gitbook.cn/d3c8e460-58e7-11eb-86e7-51557e1a6508"
style="zoom:67%;" /> | $0.1912$ |  |  
$\pi_3$ | <img
src="https://images.gitbook.cn/da9ffda0-58e7-11eb-b90b-2f717fa43dd4"
style="zoom:67%;" /> | $0.1160$ |  |  
$\pi_4$ | <img
src="https://images.gitbook.cn/e18f32c0-58e7-11eb-90b2-f32bf5b015be"
style="zoom:67%;" /> | $0.0703$ |  |  
$\pi_5$ | <img
src="https://images.gitbook.cn/e86d9f00-58e7-11eb-90b2-f32bf5b015be"
style="zoom:67%;" /> | $0.0826$ |  |  
$\pi_6$ | <img
src="https://images.gitbook.cn/f02183b0-58e7-11eb-9c81-71e9be275b25"
style="zoom:67%;" /> | $0.2246$ |  |  
  
注意到概率最大的 $\pi_1$ 和最小的 $\pi_4$，会得出一个有趣的结论：按照 score 降序排列时，概率最大；按照 score
升序排列时，概率最小。

按照上述逻辑，可以分别计算单个 List 的预测值（scores）和真实值（relevancies）的 permutation
probability。然后再将计算出的两组 permutation probability
输入到某个损失函数中，就得到了损失值，这样便可以进行后续的梯度更新等步骤，实现模型的“学习”能力。

但是这种概率的计算方式有严重的缺点，导致其不可用，因为假设 List 中有 $n$ 个样本，那么 permutation probability
的计算量居然是 $n!$，也就是只要 List 中的样本数超过 11，那么计算量就轻松上亿了，而这还仅仅是一条训练实例。

可见，permutation probability 只能停留在理论阶段。不过 ListNet
的作者以此为基础，提出了另外一个可以实际落地的概率模型——top one probability。

### Top one probability

top one probability 的定义是：对于 List 中的元素 $j$，它的 top one probability 等于 $j$ 排在
List 首位的概率。

公式为：

$$P_s(j) = \sum_{\pi(1)=j,\pi \in \Omega_n}P_s(\pi)$$

元素 $j$ 的 top one probability 等于 permutation probability 中第一个元素为 $j$ 的组合的概率之和。

以上述红黄蓝球为例，红球的 top one probability 为 $P(\pi_1) + P(\pi_2)=0.5065$。

这是否说明计算 top one probability 需要提前计算 permutation probability 呢？

当然不是啦。ListNet 的作者给出了计算公式，完全不用计算 permutation probability。

$$P_s(j) = \frac{\phi(s_j)}{\sum_{k=1}^{n}\phi(s_k)} \tag{2}$$

式中，$s_j$ 是 元素 $j$ 的得分。可以看到时间复杂度变成了 $O(n)$。

如果 $\phi(x) = e^x$，那么 $P_s(j)$ 就转化成了 $softmax$ 函数。公式$(2)$的推导请查阅文献[^1]的附录 $C$。

我们可以从另外一个角度来理解 top one probability：红黄蓝的三个球，分值都是
1，按顺序一字排开，红色球排在第一位的概率是多少？黄色和蓝色球排在第一位的概率又分别是多少呢？

答案很简单，都是 $\frac{1}{A_3^1}=\frac{1}{3}$。

扩展一下，若三个球的分值不确定是否一样时，则概率变成了 $P_s(红球) =
\frac{e^{s_{红球}}}{e^{s_{红球}}+e^{s_{黄球}}+e^{s_{蓝球}}}$。显然，分值一样时，$P_s(红球)
=P_s(黄球)=P_s(蓝球)= \frac{1}{3}$。

    
    
    # python 3.6
    # 公式(2) 对应的代码实现 
    import math
    
    def top_one_probability(scores):
        sum_exp_score = sum([math.exp(score) for score in scores])
        rank_first_score = math.exp(scores[0])
        return rank_first_score / sum_exp_score
    

有了 top one probability 这个概率分布模型，就可以按照如下步骤计算损失了：

  1. 将 relevances 转化为概率分布，即真实概率分布
  2. 将 scores 转化为概率分布，即预测概率分布
  3. 根据两个分布差异计算损失值——交叉熵

### 损失函数

交叉熵的计算公式为：

$$cross\\_entropy = -true\\_prob\\_distribution * log \
(pred\\_prob\\_distribution)$$

将上述计算损失的逻辑用原始的 Python 代码实现。

实际生产中我们使用 TensorFlow 实现本文中的所有代码：

    
    
    import math
    # top one probability
    def softmax(scores):
        sum_exp = sum([math.exp(score) for score in scores])
        return [math.exp(score) / sum_exp for score in scores]
    
    def cross_entropy(truths, preds):
        assert truths is not None, 'truths none.'
        assert preds is not None, 'preds none.'
        assert len(truths) == len(preds), 'truths len: {}, preds len: {}'.format(len(truths), len(preds))
        size = len(truths)
        loss = 0.0
        for i in range(size):
            loss += -truths[i] * math.log(preds[i])
        return loss
    
    # 真实值
    relevances = [0, 1, 2]
    # 预测值
    scores = [1, 4, 6]
    
    # 1.将 relevances 转化为概率分布
    true_top1_dist = softmax(relevances) #[0.0900, 0.2447, 0.6652]
    
    # 2.将 scores 转化为概率分布
    pred_top1_dist = softmax(scores) #[0.0059, 0.1185, 0.8756]
    
    # 3.根据两个分布差异计算损失值
    loss = cross_entropy(true_top1_dist, pred_top1_dist) #1.0724
    
    # ... 梯度更新
    

关于计算两个概率分布的差异，还有 $KL$ 散度这么一个概念，不过在实际中，我们还是使用交叉熵多一点。在**某种特定**的前提下，$KL$
散度和交叉熵本质上优化的是同样的目标，而这个前提在监督学习中基本都可以满足，感兴趣的读者可以自行研究，就不在此拓展了。

至此，ListNet 有了 **预测函数** 和 **损失函数** ，这样就可以使其拥有“学习元素顺序”的功能了。

那么，这个 Listwise 的 LTR 模型 ListNet 应该如何实现？

别着急，在后续的章节介绍完 **TensorFlow** 之后，我们再来看看这些千奇百怪的 Net 是如何被一一实现出来的。

### 总结

Learning To Rank
技术天生适合在推荐系统中发挥作用，它可以学习到元素与元素之间的相对顺序关系，一般用在召回场景以及不在乎精确预测值的排序场景。

本篇中的内容大都是文献$^1$ 的提炼，它描述了如何将 **相关性** 转变为 **概率分布**
，从而可以使用概率模型对排序学习建模。有两种方式可以实现这种转变：permutation probability 和 top one
probability. 前者因为计算量太过巨大只能存在理论中，后者在前者的基础上极大的简化了概率的计算，可以在工程中很好的落地。

有了 **真实值** 的概率分布和 **预测值** 的概率分布这两个分布之后，就可以使用交叉熵或者 $KL$
散度去计算分布差异得到损失值，然后通过梯度更新让模型拥有了“学习顺序”的能力。

这些基础细节的掌握，对于后续我们使用 **TensorFlow** 来实现 ListNet 是必不可少的。

下一篇，我们将进入到深度学习领域，看看它的常规结构以及搭建 TensorFlow 模型必须掌握的特征处理工具 Feature Column。

  * 注释 1：[Learning to Rank: From Pairwise Approach to Listwise Approach](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-2007-40.pdf)
  * 注释 2：[A Short Introduction to Learning to Rank](http://times.cs.uiuc.edu/course/598f14/l2r.pdf)

