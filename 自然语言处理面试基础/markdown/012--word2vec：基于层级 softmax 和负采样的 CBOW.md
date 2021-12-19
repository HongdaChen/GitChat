今天我们来看一下
word2vec，它是自然语言处理中非常重要的概念，是一种用语言模型做词嵌入的算法，目的就是将文字转化为更有意义的向量，进而可以让深度神经网络等模型更好地理解文本数据。

### 词嵌入

词嵌入（Word
Embeddings）就是将文本转化为数字表示，转化成数值向量后才能传递给机器学习等模型进行后续的训练和运算，并应用于情感分析文档分类等诸多任务中。

词嵌入的方法主要有两大类：基于传统频数统计的词嵌入，和基于预测算法的词嵌入。

其中传统的词嵌入方法有：

  1. 计数向量
  2. TF-IDF 向量
  3. 共现矩阵
  4. One-hot 向量

**1\. 计数向量**

就是统计每个单词在各个文档里出现的次数。例如我们有 N
个文档，这些文档里的不重复的单词组成了一个字典，然后构建一个矩阵，矩阵的每一行代表每个文档，每一列代表字典里的每个单词，矩阵的数值是每个单词在各文档里出现的次数，最后每一列就构成了这个词的向量表示：

![](https://images.gitbook.cn/d154b1b0-4e6a-11ea-9e01-3ff6ee0d9989)

**2\. TF-IDF 向量**

在每个文档中有一些单词对理解文档起着很重要的作用，而有一些词例如特别常见的冠词等几乎在每个文档中都存在，这些词对理解文档的意义并不大，所以需要放大有意义的词的作用，减少常用词的影响，TF-
IDF（Term Frequency–Inverse Document Frequency）就是一种这样的方法。

例如我们有这样一个例子，下图是两个文档中同样的几个单词出现的次数：

![](https://images.gitbook.cn/af2ae680-4e61-11ea-bb37-55480bd50c9e)

首先计算每个单词的 TF 值，即这个单词在当前文档中出现的次数/当前文档一共有多少单词。

    
    
    TF(It, Doc1) = 1/8
    TF(It, Doc2) = 1/5
    

再计算每个单词的 IDF = log(N/n)，N 是指一共有 N 个文档，n 是这个单词几个文档中出现过。

    
    
    IDF(It) = log(2/2) = 0
    IDF(NLP) = log(2/1) = 0.301
    

然后将二者相乘，得到每一个单词对每个文档的 TF-IDF：

    
    
    TF-IDF(It, Doc1) = (1/8) * (0) = 0
    TF-IDF(It, Doc2) = (1/5) * (0) = 0
    TF-IDF(NLP, Doc1) = (4/8)*0.301 = 0.15
    

然后构造一个矩阵，每行代表每个文档，每一列代表每个单词，矩阵的数值就是 TF-IDF，同样每一列就是相应单词的词向量表示。

**3\. 共现矩阵**

共现矩阵就是统计在指定大小的窗口中两个词同时出现的次数。它的思想是基于相似的单词所处的上下文也应该是差不多的，例如有两句话“苹果是水果，芒果是水果”，因为“苹果”和“芒果”有相似的上下文“是水果”，所以苹果和芒果是相似的事物，所以用共现来建模这种相似性。

例如我们有三句话：

    
    
    NLP is not easy， NLP is interesting，NLP is amazing
    

首先定义一个窗口长度 c，表示在当前单词的左边取 c 个，右边取 c 个，这里取 c = 2，则共现矩阵为：

![](https://images.gitbook.cn/2ce5a330-4e62-11ea-9e01-3ff6ee0d9989)

其中我们看 `(NLP,is)= 4` 这一对是如何计算的，以 `NLP` 为中心的窗口一共有 3 个，在这 3 个窗口中 `is` 一共出现了 4
次，所以共现次数就为 4，同理可以计算其他的数值：

![](https://images.gitbook.cn/59d0d040-4e62-11ea-9e01-3ff6ee0d9989)

建立好矩阵后，再进行 SVD 分解：

![](https://images.gitbook.cn/6cd129b0-4e62-11ea-8532-814baa40f153)

U 和 S 的点积就得到单词的词向量表示，而 V 是单词的上下文表示。

**4\. One-hot 向量**

用语料库建立一个字典，每个单词在它相应的位置处值为 1，其余地方为 0。

例如语料库为 (I,like,NLP,very,much...)，则 NLP 的 one-hot 向量为 (0,0,1,0,0,...)。

* * *

前面这几种传统的词嵌入方法在数据量很大的时候效率较低，而且没有包含丰富的语义信息，2013 年 Mitolov 在论文 [_Efficient
Estimation of Word Representations in Vector
Space_](https://arxiv.org/pdf/1301.3781.pdf) 中提出了
word2vec，这种方法可以将单词更高效地转化成携带更多信息的向量，之后又在论文 [_Distributed Representations of
Words and Phrases and their
Compositionality_](https://papers.nips.cc/paper/5021-distributed-
representations-of-words-and-phrases-and-their-compositionality.pdf)
中通过负采样等方法对其进行改进。

  1. 由 word2vec 得到的词向量可以用余弦来衡量两个单词之间的距离，就可以知道哪些单词离得近，即哪些单词之间具有相似性，还能知道单词之间的所属关系，如 `Rome - Italy = Beijing - China`。
  2. 这种词向量还携带着单词的语境语义等信息，在下游的各种自然语言处理任务中如情感分析、文档分类、机器翻译等就可以有更好的效果。
  3. 而且这个过程是自动学习出来的，不需要人为的干预。

在使用时可以用谷歌的预训练模型，包含 300 万单词的词向量，这些词向量是用谷歌新闻数据集中大约 1000 亿词训练而得的，也可以在自己的数据集上用
word2vec 模型来训练词向量，本文中就用一部分维基百科数据集来训练模型。

word2vec 有 CBOW（Continuous Bag-of-Words）和 Skip-Gram 两种训练思路。CBOW
是由上下文预测中心词，Skip-Gram 是由中心词预测上下文，今天我们先来看 CBOW。

### 什么是 CBOW

CBOW：Continuous Bag-of-Words 连续词袋模型，是根据上下文来预测中心词的模型。它的输入是 **上下文**
中每个词的词向量，输出是词汇表中所有词的 softmax 概率，然后希望其中输入上下文所对应的 **真实中心词的 softmax 概率是最大的**
，模型的前向计算得出误差后，再通过反向传播和梯度下降法更新模型的参数，通过多次迭代后让模型的预测值尽可能靠近真实值。

![](https://images.gitbook.cn/b1eb5eb0-4e64-11ea-96da-c5d22898644a)

下面详细地看一下 CBOW 的前向计算和反向传播。

#### **前向计算**

我们先来看 input 是一个单词的情况。

![](https://images.gitbook.cn/cc2563c0-4e64-11ea-8532-814baa40f153)

模型中有两个核心矩阵分别是 W 和 W‘：

  * W 是 embedding 矩阵，维度是：`vocab_size * embedding_size`，记作：$W=\\{w_{ki}\\}$。
  * W' 是 context 矩阵，维度是：`embedding_size * vocab_size`，记作：$W'=\\{w'_{ij}\\}$。

输入的每个单词先转化为 one-hot 向量 x，大小是 `vocab_size * 1`。

然后在 embedding 矩阵中找到自己的位置，就是 W 的其中一行，将 x 和嵌入矩阵相乘就可以得到 $h = W^T x = W^T_{(k,.)}
:= V^T_{w_I}$（式 1），维度是 embedding_size * 1。其中我们用 $V_{w_I}$ 来表示输入单词 $w_I$
的词向量。同时可以得到 h 的第 i 个元素 $h_i = \sum_{k=1}^V x_k w_{ki}$（式 2）。

h 与 context 矩阵 W' 相乘，得到一个维度为 `vocab_size * 1` 的向量 $u = W'^T h$（式
3），这个就是模型的预测向量。其中 u 的第 j 个元素 $u_j = W'^T_{(.,j)} h := V'^T_{w_j} h$（式 4）。我们用
$V'_{w_j}$ 来表示预测单词 $w_j$ 的词向量，其中 j 取遍整个词汇表 1~V。

再对 u 应用 softmax 得到概率向量 $y = softmax(u)$。其中每个元素

$$y_j = P(w_j | w_I) = softmax(u_j) = \frac{e^{u_{j}}}{\sum_{j'=1}^V
e^{u_{j'}}}（式 5）$$

有了预测结果后，再来看模型的损失函数是什么。

因为我们的目标是希望模型能够预测出真实的中心词，也就是希望中心词所对应的概率最大，即：

$$max P(w_O | w_I) := max y_{j^*} \iff max (log y_{j^*})$$

令误差

$$ E = - log y_{j^*} = - log P(w_O | w_I) = -u_{j^*}+log(\sum_{j'=1}^V
e^{u_{j'}})（式 6）$$

#### **反向传播**

前面是前向计算我们得到了误差，接下来求两个矩阵的梯度。

首先求 $ \frac{∂ E}{∂ V'_{w_j}} = \frac{∂ E}{∂ u_j} \frac{∂ u_j}{∂ V'_{w_j}} $，其中
$\frac{∂ E}{∂ u_j} = e_j$，详细推导请看下图：

![](https://images.gitbook.cn/e5f20b90-4e65-11ea-bb37-55480bd50c9e)

再由（式 4）得到

$$\frac{∂ u_j}{∂ V'_{w_j}} = h$$

最后

$$ \frac{∂ E}{∂ V'_{w_j}} = \frac{∂ E}{∂ u_j} \frac{∂ u_j}{∂ V'_{w_j}} = e_j h
$$

于是进行梯度更新：

$$V'^{(new)}_{w_j} = V'^{(old)}_{w_j} -\eta \cdot e_j \cdot h, (j=1,...,V) （式
7）$$

再求

$$ \frac{∂ E}{∂ w_{ki}} = \frac{∂ E}{∂ h_i} \frac{∂ h_i}{∂ w_{ki}} $$

由（式 6）和（式 4）可知 E 中包括 1~V 个 $u_j$，且每个 $u_j$ 里都有 $h_i$，可得：

$$ \frac{∂ E}{∂ h_i} = \sum_{j=1}^V \frac{∂ E}{∂ u_j} \frac{∂ u_j}{∂ h_i} =
\sum_{j=1}^V e_j w'_{ij} := EH_i $$

所以有

$$ \frac{∂ E}{∂ w_{ki}} = \frac{∂ E}{∂ h_i} \frac{∂ h_i}{∂ w_{ki}} = EH_i
\cdot x_k $$

进而

$$ \frac{∂ E}{∂ W} = x \cdot EH^T $$

又因为 x 是 one-hot 向量，只有其中一个位置是 1，其余是 0，所以只有输入单词对应的位置上有数值，于是：

$$ \frac{∂ E}{∂ V_{w_I}} = \frac{∂ E}{∂ W_{(k,.)}} = EH^T $$

然后进行梯度更新：

$$V^{(new)}_{w_I} = V^{(old)}_{w_I} -\eta \cdot EH^T （式 8）$$

上面是计算输入为一个单词的时候，如果是 C 个上下文单词，那么：

（式 1）处，

$$ h = \frac{1}{C} W^T (x_1 + ... + x_C) = \frac{1}{C} (V_{w_1} + ... +
V_{w_C}) $$

（式 7）处的公式是一样的，但是注意其中

$$ h = \frac{1}{C} (V_{w_1} + ... + V_{w_C}) $$

（式 8）处多了一个系数：

$$V^{(new)}_{w_I,c} = V^{(old)}_{w_I,c} - \frac{1}{C} \cdot \eta \cdot EH^T,
(c=1,...,C) $$

但是如果直接按照上面的模型做的话，我们有计算 P：

$$p(w_O \vert w_I) = \frac{\exp({v'_{w_O}}^{\top} v_{w_I})}{\sum_{i=1}^V
\exp({v'_{w_i}}^{\top} v_{w_I})}$$

由于词汇表大小通常会达到数百万，这样从隐藏层到输出层的计算量会非常大，所以需要做一些简化，接下来我们来看两种改进方案。

#### **第一种，基于层级 Softmax**

当词汇表 V 很大时，对于每个样本需要遍历所有单词来计算分母，这个计算量是非常大的，于是需要用更高效的方法来计算这个条件概率，层级 Softmax
和负采样就是其中两种常用方法。

这种方法通过建立一个 Huffman 树来对原来的神经网络进行简化：

![](https://images.gitbook.cn/f6443f80-4e66-11ea-8532-814baa40f153)

  1. 当我们输入上下文的词向量后，直接对所有输入的向量求和取平均，来得到一个整体的词向量，这里省去了神经网络的线性变换和激活函数。$x_w = \frac{1}{2c}\sum\limits_{i=1}^{2c}x_i$
  2. 然后在隐藏层和输出层之间，要建立一个 Huffman 树，这个树的根结点是上下文的平均词向量，叶子结点对应的是词汇表的所有词。

这样已知上下文来得到中心词，就是沿着树形结构从根结点走到相应的叶子结点，只通过一条路径来计算出中心词的概率，可以避免计算所有词的 softmax
概率，进而减少计算量。

**1\. 那么什么是 Huffman 树呢？**

Huffman
树是一个二叉树，它的每个叶子结点都带有不同的权重，每个叶子结点的权重乘以根结点到它的步数，再求和，得到的整体带权路径是所有叶结点排列情况中最小的。简称为带权重的路径长度最短二叉树，也称为最优二叉树。

例如，我们有这样一组数据“1,2,5,7”，用下面两种树的结构分别计算带权路径：

![](https://images.gitbook.cn/260c9be0-4e67-11ea-a847-a5aa0d0597f8)

  * 左边：$ 1*2 + 2*2 + 5*2 + 7*2 = 30 $
  * 右边：$ 1*3 + 2*3 + 5*2 + 7*1 = 26 $

可以发现右侧的 Huffman 树得到的结果小于左边的。

此外，Huffman 树还有一个编码机制。例如规定左子树编码为 1，右子树编码为 0，那么上面右图中，权值为“1”叶子结点的 Huffman
编码为：001。

![](https://images.gitbook.cn/3e867a60-4e67-11ea-a873-6386be7e020f)

**2\. 如何构造 Huffman 树呢？**

就是把权重越大的结点放在离根结点越近的地方。

当我们在构造 Huffman
树时，可以将词汇表中每个单词的频数作为权重，出现频数越多的词，放在越靠近根结点的地方。每一次将语料库中剩余的词中，词频最低的两个词组合成一个子树，再在剩下的词中找到频数最少的词与新子树组合起来。

例如，我们有 "I, like, NLP" 3 个结点，结点的权值分别为 (20,8,6)。

首先是最小的“like”和“NLP”合并，得到的子树的根结点权重是 14，然后森林里就剩下 2 棵树了，结点权重分别是 (20,14)，再用“I”和权重为
14 的结点合并，得到了新的子树：

![](https://images.gitbook.cn/589ca3c0-4e67-11ea-8128-358088d96de7)

**3\. 将词汇表构建成 Huffman 树之后，是如何由上下文走到中心词的呢？**

在 Huffman 树的每个结点处，相当于是用逻辑回归做一下二分类，我们定义向右子树走为正例，编码为 0，概率为

$$P(+) = \sigma(x_w^T\theta) = \frac{1}{1+e^{-x_w^T\theta}}$$

向左子树走为负例，编码为 1，概率为

$$P(-) = 1 - P(+) = 1 - \sigma(x_w^T\theta) $$

哪个概率大就向哪边走。

上面的例子“I like NLP”这句话，想要得到“NLP”这个词，那么它的路径应该是 右右，即第一次判断希望 P(+) 概率大，第二次也是 P(+)
概率大，将所有判断的概率累乘起来，这样就得到了“NLP”的概率。

![](https://images.gitbook.cn/9c2acc70-4e67-11ea-a873-6386be7e020f)

我们的目标也正是希望得到的这个“NLP”的概率是最大的。

**4\. 经过上面这个过程，我们来看模型的目标函数**

先来定义一些符号：

  * $x_w$：输入上下文词向量的平均词向量
  * $l_w$：从根结点到中心词 w 的路径上包含的结点个数
  * $\theta_{j}^w$：从根结点到中心词 w 的路径上，每个非叶子结点逻辑回归的参数
  * $d_{j}^w$：是每个结点的编码，即 右边为 0，左边为 1

每一次用于判断向左走向右走的概率形式为：

$$P(d_j^w|x_w, \theta_{j-1}^w)= \begin{cases} \sigma(x_w^T\theta_{j-1}^w)&
{d_j^w=0}\\\ 1- \sigma(x_w^T\theta_{j-1}^w) & {d_j^w = 1} \end{cases}$$

将所有结点处的逻辑回归累乘起来就得到了目标函数：

$$\prod_{j=2}^{l_w}P(d_j^w|x_w, \theta_{j-1}^w) = \prod_{j=2}^{l_w}
[\sigma(x_w^T\theta_{j-1}^w)]
^{1-d_j^w}[1-\sigma(x_w^T\theta_{j-1}^w)]^{d_j^w}$$

再进一步将目标函数转化为对数似然函数：

$$L= log \prod_{j=2}^{l_w}P(d_j^w|x_w, \theta_{j-1}^w) =
\sum\limits_{j=2}^{l_w} ((1-d_j^w) log [\sigma(x_w^T\theta_{j-1}^w)] + d_j^w
log[1-\sigma(x_w^T\theta_{j-1}^w)])$$

我们希望在给定上下文的情况下，得到目标中心词的概率 p 越大越好，也就是要让这个似然函数越大越好，那我们就可以用梯度上升法来进行求解，即求对数似然函数对参数
$\theta$ 和词向量 x 的偏导。

对 $\theta$ 求偏导：

$\begin{align} \frac{\partial L}{\partial \theta_{j-1}^w} & =
(1-d_j^w)\frac{(\sigma(x_w^T\theta_{j-1}^w)(1-\sigma(x_w^T\theta_{j-1}^w)}{\sigma(x_w^T\theta_{j-1}^w)}x_w
- d_j^w \frac{(\sigma(x_w^T\theta_{j-1}^w)(1-\sigma(x_w^T\theta_{j-1}^w)}{1-
\sigma(x_w^T\theta_{j-1}^w)}x_w \end{align}$ $=
(1-d_j^w)(1-\sigma(x_w^T\theta_{j-1}^w))x_w -
d_j^w\sigma(x_w^T\theta_{j-1}^w)x_w$ $=
(1-d_j^w-\sigma(x_w^T\theta_{j-1}^w))x_w $

对 $x_w$ 求偏导，这里 $x_w$ 是输入词向量求和平均后得到的：

$$\frac{\partial L}{\partial x_w} =
\sum\limits_{j=2}^{l_w}(1-d_j^w-\sigma(x_w^T\theta_{j-1}^w))\theta_{j-1}^w$$

然后进行梯度更新，因为是梯度上升，所以用＋号，$\theta$ 的更新公式：

$$\theta_{j-1}^w = \theta_{j-1}^w + \eta
(1-d_j^w-\sigma(x_w^T\theta_{j-1}^w))x_w$$

x 的更新公式：因为刚才求得的是 $x_w$ 整体的梯度，这里我们用下面这个式子对每个词向量都进行更新，来保证每个词向量的更新都是沿着整体趋势走的。

$$x_{w_i}= x_{w_i} +\eta
\sum\limits_{j=2}^{l_w}(1-d_j^w-\sigma(x_w^T\theta_{j-1}^w))\theta_{j-1}^w
\;(i =1,2..,2c)$$

**总结一下基于 Huffman 树的 CBOW 的流程就是：**

  1. 已知上下文的词向量，计算出平均词向量
  2. 根据词汇表中各单词的词频，建立一个 Huffman 树
  3. 根据 Huffman 树写出叶子结点的中心词的目标函数，再转化为对数似然函数
  4. 计算对数似然函数对参数 $\theta$，词向量 $x_w$ 的梯度
  5. 用梯度上升法来更新 $\theta$ 和 $x$，直到收敛

#### **第二种，基于负采样**

当语料库很大时，中心词的词频又很低，Huffman 树会形成很多二分类结点，这样计算复杂度还是很高，于是有另一种解决方案，叫做负采样，即采样一些负例。

负采样的思想就是我们在计算上下文与目标词对的得分时，加一些噪声数据，损失函数变成两部分，真实单词对的得分和噪声的分数：

![](https://images.gitbook.cn/10ce6af0-4e68-11ea-96da-c5d22898644a)

这样在每次优化参数时，只需要关注损失函数中涉及到的词，并且将 softmax 函数换成 sigmoid 函数，这样就不需要将所有单词都计算一遍了。

当目标中心词与模型输入的上下文是实实在在配对的一组样本时，这个中心词就叫 **正例** ，用 1 表示，那么取一些和这个上下文无关的词，就是 **负例**
，用 0 表示。例如，“我饿了想吃饭”这句话中，“饭”就是正例，“拳头”就是负例。

负采样的目的是给模型训练的数据加一些噪音，避免数据不平衡。因为如果模型训练的数据都是正例，那么它只需要定义一个最简单的一直返回 1
的模型也可以达到很高的准确率。

**1\. 如何选择负样本？**

负样本就去词汇表里采样选取，我们可以很容易想到的方法是生成一个随机数，取其相对应的词。

**如果在采样时还希望频次越高的词越容易被选为负样本时要怎么做呢？**

我们可以设词汇表的大小为 V，将一段长度为 1 的线段分成 V
份，这样每份对应词汇表中的一个词。其中每一份的长度由下面这个公式决定，这样每份的长度就和相应词的频数相关：

$$ len(w) = \frac{count(w)^{3/4}}{\sum\limits_{u \in vocab} count(u)^{3/4}} $$

**于是高频词对应的线段长，低频词对应的线段短，频次越高的词就越容易被取出来。**

这之后再随机生成 0-1 之间的几个数，每个数所处的线段所对应的词就是采样出来的负例。

采样后，这次不需要构造 Huffman 树，而是直接用 sigmoid 函数进行分类， **我们的目标是想要最大化这个式子** ：

$$\prod_{i=0}^{neg}P(context(w_0), w_i) =
\sigma(x_{w_0}^T\theta^{w_0})\prod_{i=1}^{neg}(1-
\sigma(x_{w_0}^T\theta^{w_i}))$$

第一项 sigmoid 是中心词 w 正例的概率，后面几项中的 sigmoid 是负例 u 的概率，1 - 负例的概率 =
正例的概率，这个式子的意思是希望最后的结果是朝着正确的趋势发展的。这也就是我们的目标函数， **我们希望给定上下文后，预测出中心词的概率越大越好。**

接下来写出似然函数：

$$\prod_{i=0}^{neg} \sigma(x_{w_0}^T\theta^{w_i})^{y_i}(1-
\sigma(x_{w_0}^T\theta^{w_i}))^{1-y_i}$$

再转化为对数似然函数：

$$L = \sum\limits_{i=0}^{neg}y_i log(\sigma(x_{w_0}^T\theta^{w_i})) + (1-y_i)
log(1- \sigma(x_{w_0}^T\theta^{w_i}))$$

然后求对数似然函数对参数 $\theta$ 的偏导：

$$\begin{align} \frac{\partial L}{\partial \theta^{w_i} } &= y_i(1-
\sigma(x_{w_0}^T\theta^{w_i}))x_{w_0}-(1-y_i)\sigma(x_{w_0}^T\theta^{w_i})x_{w_0}
\\\ & = (y_i -\sigma(x_{w_0}^T\theta^{w_i})) x_{w_0} \end{align}$$

对 x 的偏导：

$$\frac{\partial L}{\partial x^{w_0} } = \sum\limits_{i=0}^{neg}(y_i
-\sigma(x_{w_0}^T\theta^{w_i}))\theta^{w_i}$$

再用梯度上升法进行梯度更新，不断迭代，得到参数和词向量的最优解。

**总结一下基于负采样的 CBOW 流程是：**

  1. 对于每一组训练样本，负采样出负例词
  2. 写出目标函数，再转化为对数似然函数
  3. 计算对数似然函数对参数 $\theta$，词向量 $x_w$ 的梯度
  4. 用梯度上升法来更新 $\theta$ 和 $x_w$，直到收敛

### 应用 CBOW

下面我们用 TensorFlow 建立 CBOW 模型训练词向量。

这里使用的数据来自 [Evan
Jones](https://www.evanjones.ca/software/wikipedia2text.html)，是他提取的部分维基百科的数据，有大概三百万个单词。

模型的训练主要有下面几大步骤：

  1. 数据预处理
  2. 建立字典
  3. 定义模型
  4. 训练模型
  5. 结果可视化

#### **1\. 导入所需要的库**

    
    
    %matplotlib inline
    import collections
    import math
    import numpy as np
    import os
    import random
    import tensorflow as tf
    import bz2
    from matplotlib import pylab
    from six.moves import range
    from six.moves.urllib.request import urlretrieve
    from sklearn.manifold import TSNE
    from sklearn.cluster import KMeans
    import nltk                             # 用来做预处理
    nltk.download('punkt')
    import operator                            # 用来对字典里的词进行排序
    from math import ceil
    import csv
    

#### **2\. 下载 Wikipedia 数据**

    
    
    url = 'http://www.evanjones.ca/software/'
    
    def maybe_download(filename, expected_bytes):
      if not os.path.exists(filename):
        print('Downloading file...')
        filename, _ = urlretrieve(url + filename, filename)
      statinfo = os.stat(filename)
      if statinfo.st_size == expected_bytes:
        print('Found and verified %s' % filename)
      else:
        print(statinfo.st_size)
        raise Exception(
          'Failed to verify ' + filename + '. Can you get to it with a browser?')
      return filename
    
    filename = maybe_download('wikipedia2text-extracted.txt.bz2', 18377035)
    

或者在[这里下载](https://github.com/amolnayak311/nlp-with-
tensorflow/blob/master/wikipedia2text-extracted.txt.bz2)，将数据保存到本地文件夹，就可以省略这一步。

#### **3\. 读取数据，并用 NLTK 进行预处理**

读取第一个文本文件，转化为一个单词列表，经过预处理后，有 3360286 个单词，我们可以打印看一下 words 里面的数据：

    
    
    def read_data(filename):
      with bz2.BZ2File(filename) as f:
    
        data = []
        file_size = os.stat(filename).st_size
        chunk_size = 1024 * 1024                                         # 每次读取 1 MB 的数据
        print('Reading data...')
        for i in range(ceil(file_size//chunk_size)+1):
            bytes_to_read = min(chunk_size,file_size-(i*chunk_size))
            file_string = f.read(bytes_to_read).decode('utf-8')
            file_string = file_string.lower()                            # 转化为小写字母
            file_string = nltk.word_tokenize(file_string)                
            data.extend(file_string)
      return data
    
    words = read_data('wikipedia2text-extracted.txt.bz2')
    print('Data size %d' % len(words))
    print('Example words (start): ',words[:10])
    print('Example words (end): ',words[-10:])
    
    
    
    Data size 3360286
    Example words (start):  ['propaganda', 'is', 'a', 'concerted', 'set', 'of', 'messages', 'aimed', 'at', 'influencing']
    Example words (end):  ['favorable', 'long-term', 'outcomes', 'for', 'around', 'half', 'of', 'those', 'diagnosed', 'with']
    

#### **4\. 建立字典**

在这里我们要建立 2 个字典，dictionary 是为了将文本转化为数字，data 就是文本转化为数字的结果，reverse_dictionary
是在模型训练结束后将数字转化为文本，方便我们查看结果。

假设我们输入的文本是 "I like to go to school"：

  * dictionary：将单词映射为 ID，例如 `{I:0, like:1, to:2, go:3, school:4}`
  * reverse_dictionary：将 ID 映射为单词，例如 `{0:I, 1:like, 2:to, 3:go, 4:school}`
  * count：计数每个单词出现了几次，例如 `[(I,1),(like,1),(to,2),(go,1),(school,1)]`
  * data：将读取的文本数据转化为 ID 后的数据列表，例如 `[0, 1, 2, 3, 2, 4]`

此外，我们用 UNK 代表比较少见的单词：

    
    
    vocabulary_size = 50000                                                             # 将词汇表大小限制在 50000
    
    def build_dataset(words):
      count = [['UNK', -1]]
      count.extend(collections.Counter(words).most_common(vocabulary_size - 1))            # 选择 50000 个最常见的单词，其他的用 UNK 代替
      dictionary = dict()
    
      for word, _ in count:                                                                # 为每个 word 建立一个 ID，保存到一个字典里
        dictionary[word] = len(dictionary)
    
      data = list()
      unk_count = 0
    
      for word in words:                                                                # 将文本数据中所有单词都换成相应的 ID，得到 data
        if word in dictionary:                                                            # 不在字典里的单词，用 UNK 代替
          index = dictionary[word]
        else:
          index = 0  # dictionary['UNK']
          unk_count = unk_count + 1
        data.append(index)
    
      count[0][1] = unk_count
    
      reverse_dictionary = dict(zip(dictionary.values(), dictionary.keys())) 
    
      assert len(dictionary) == vocabulary_size                                            # 确保字典和词汇表的大小一样
    
      return data, count, dictionary, reverse_dictionary
    
    data, count, dictionary, reverse_dictionary = build_dataset(words)
    print('Most common words (+UNK)', count[:5])
    print('Sample data', data[:10])
    del words  
    
    
    
    Most common words (+UNK) [['UNK', 69215], ('the', 226881), (',', 184013), ('.', 120944), ('of', 116323)]
    Sample data [1721, 9, 8, 16471, 223, 4, 5165, 4456, 26, 11590]
    

#### **5\. 生成批次数据**

这一步是用来生成 CBOW 所需要的数据，我们知道 CBOW 是通过上下文预测中心词，所以需要构造出这样的输入。

首先定义 window_size，是指每个中心词的前面和后面每一侧所要取的词数，例如取 `window_size = 2`。

模型的输入数据为 batch 即上下文，如：

    
    
    [['propaganda', 'is', 'concerted', 'set'], ['is', 'a', 'set', 'of'], ['a', 'concerted', 'of', 'messages'], ['concerted', 'set', 'messages', 'aimed'], ['set', 'of', 'aimed', 'at'], ['of', 'messages', 'at', 'influencing'], ['messages', 'aimed', 'influencing', 'the'], ['aimed', 'at', 'the', 'opinions']]
    

模型的预测目标为 labels 即中心词，如：

    
    
    ['a', 'concerted', 'set', 'of', 'messages', 'aimed', 'at', 'influencing']
    

下面的 generate_batch_cbow 就是为了得到模型的输入和目标输出，其中 batch 的维度为 `(batch_size,
window_size*2)`，labels 的维度为 `(batch_size, 1)`。

    
    
    data_index = 0
    
    def generate_batch_cbow(batch_size, window_size):
    
        global data_index                                                                # 每次读取一个数据集后增加 1
    
        span = 2 * window_size + 1                                                         # span 是指中心词及其两侧窗口的总长度
    
        batch = np.ndarray(shape=(batch_size,span-1), dtype=np.int32)                    # batch 为上下文数据集，一共有 span - 1 个列
        labels = np.ndarray(shape=(batch_size, 1), dtype=np.int32)                        # labels 为中心词
    
        buffer = collections.deque(maxlen=span)                                            # buffer 用来存储 span 内部的数据
    
        for _ in range(span):
            buffer.append(data[data_index])
            data_index = (data_index + 1) % len(data)
    
        # 批次读取数据，对于每一个 batch index 遍历 span 的所有元素，填充到 batch 的各个列中
        for i in range(batch_size):
            target = window_size  # 在 buffer 的中心是中心词
            target_to_avoid = [ window_size ] # 因为只需要考虑中心词的上下文，这时不需要考虑中心词
    
            # 将选定的中心词加入到 avoid_list 下次时用
            col_idx = 0
            for j in range(span):
                # 建立批次数据时忽略中心词
                if j==span//2:
                    continue
                batch[i,col_idx] = buffer[j] 
                col_idx += 1
            labels[i, 0] = buffer[target]
    
            # 每次读取一个数据点，需要移动 span 一次，建立一个新的 span
            buffer.append(data[data_index])
            data_index = (data_index + 1) % len(data)
    
        return batch, labels
    
    for window_size in [1,2]:
        data_index = 0
        batch, labels = generate_batch_cbow(batch_size=8, window_size=window_size)
        print('\nwith window_size = %d:' % (window_size))
        print('    batch:', [[reverse_dictionary[bii] for bii in bi] for bi in batch])
        print('    labels:', [reverse_dictionary[li] for li in labels.reshape(8)])
    
    
    
    with window_size = 1:
        batch: [['propaganda', 'a'], ['is', 'concerted'], ['a', 'set'], ['concerted', 'of'], ['set', 'messages'], ['of', 'aimed'], ['messages', 'at'], ['aimed', 'influencing']]
        labels: ['is', 'a', 'concerted', 'set', 'of', 'messages', 'aimed', 'at']
    with window_size = 2:
        batch: [['propaganda', 'is', 'concerted', 'set'], ['is', 'a', 'set', 'of'], ['a', 'concerted', 'of', 'messages'], ['concerted', 'set', 'messages', 'aimed'], ['set', 'of', 'aimed', 'at'], ['of', 'messages', 'at', 'influencing'], ['messages', 'aimed', 'influencing', 'the'], ['aimed', 'at', 'the', 'opinions']]
        labels: ['a', 'concerted', 'set', 'of', 'messages', 'aimed', 'at', 'influencing']
    

#### **6\. 设置超参数**

    
    
    batch_size = 128                             
    embedding_size = 128             # embedding 向量的维度
    
    window_size = 2                 # 中心词的左右两边各取 2 个
    
    valid_size = 16                 # 随机选择一个 validation 集来评估单词的相似性
    
    valid_window = 50                # 从一个大窗口随机采样 valid 数据
    
    # 选择 valid 样本时, 取一些频率比较大的单词，也选择一些适度罕见的单词
    valid_examples = np.array(random.sample(range(valid_window), valid_size))            
    valid_examples = np.append(valid_examples,random.sample(range(1000, 1000+valid_window), valid_size),axis=0)
    
    num_sampled = 32                 # 负采样的样本个数
    

#### **7\. 定义输入输出**

    
    
    tf.reset_default_graph()
    
    # 用于训练的上下文数据，有 2*window_size 列
    train_dataset = tf.placeholder(tf.int32, shape=[batch_size,2*window_size])            
    # 用于训练的中心词
    train_labels = tf.placeholder(tf.int32, shape=[batch_size, 1])                        
    # Validation 不需要用 placeholder，因为前面已经定义了 valid_examples
    valid_dataset = tf.constant(valid_examples, dtype=tf.int32)    
    

#### **8\. 定义模型的参数**

定义一个 embedding 层和神经网络的参数。

    
    
    embeddings = tf.Variable(tf.random_uniform([vocabulary_size, embedding_size], -1.0, 1.0,dtype=tf.float32))
    
    # Softmax 的权重和 Biases
    softmax_weights = tf.Variable(tf.truncated_normal([vocabulary_size, embedding_size],
                     stddev=0.5 / math.sqrt(embedding_size),dtype=tf.float32))
    softmax_biases = tf.Variable(tf.random_uniform([vocabulary_size],0.0,0.01))
    

#### **9\. 定义模型**

在前文理论部分我们知道了 CBOW 的输入要先做一下平均，在这一步中：

  * 首先定义一个 lookup 函数，为每一批的输入数据获取相应的嵌入向量。
  * 然后定义 stacked_embedings 就是整个窗口中单词的词向量矩阵，它的维度是 `[batch_size，embedding_size，2 * window_size]`。
  * mean_embeddings 就是平均后的词向量，它的维度是 `[batch_size，embedding_size]`。
  * 再定义负采样损失函数 tf.nn.sampled _softmax_ loss，接收嵌入向量和之前定义的神经网络参数。

    
    
    # 为 input 的每一列做 embedding lookups
    # 然后求平均，得到大小为 embedding_size 的词向量
    stacked_embedings = None
    print('Defining %d embedding lookups representing each word in the context'%(2*window_size))
    for i in range(2*window_size):
        embedding_i = tf.nn.embedding_lookup(embeddings, train_dataset[:,i])        
        x_size,y_size = embedding_i.get_shape().as_list()
        if stacked_embedings is None:
            stacked_embedings = tf.reshape(embedding_i,[x_size,y_size,1])
        else:
            stacked_embedings = tf.concat(axis=2,values=[stacked_embedings,tf.reshape(embedding_i,[x_size,y_size,1])])
    
    assert stacked_embedings.get_shape().as_list()[2]==2*window_size
    print("Stacked embedding size: %s"%stacked_embedings.get_shape().as_list())
    mean_embeddings =  tf.reduce_mean(stacked_embedings,2,keepdims=False)
    print("Reduced mean embedding size: %s"%mean_embeddings.get_shape().as_list())
    
    # 每次用一些负采样样本计算 softmax loss, 
    # 输入是训练数据的 embeddings
    # 用这个 loss 来优化 weights, biases, embeddings
    loss = tf.reduce_mean(
        tf.nn.sampled_softmax_loss(weights=softmax_weights, biases=softmax_biases, inputs=mean_embeddings,
                               labels=train_labels, num_sampled=num_sampled, num_classes=vocabulary_size))
    
    
    
    Defining 4 embedding lookups representing each word in the context
    Stacked embedding size: [128, 128, 4]
    Reduced mean embedding size: [128, 128]
    

其中
[tf.nn.sampled_softmax_loss](https://www.tensorflow.org/api_docs/python/tf/nn/sampled_softmax_loss)
就是采用了负采样的损失函数，如果大家对其数学原理感兴趣可以看[这篇文章](https://www.tensorflow.org/extras/candidate_sampling.pdf)，这个损失函数中会自动从词汇表中选择
num_sampled 个与目标中心词不相关的单词作为负样本，用比较简单的公式来表示这个损失函数的话：

![](https://images.gitbook.cn/9e2f6420-4e69-11ea-b2e1-7d26d62747f1)

#### **10\. 定义优化算法**

这里选择 Adagrad：

    
    
    optimizer = tf.train.AdagradOptimizer(1.0).minimize(loss)
    

#### **11\. 定义计算单词相似性的函数**

用余弦距离来计算样本的相似性：

    
    
    norm = tf.sqrt(tf.reduce_sum(tf.square(embeddings), 1, keepdims=True))
    normalized_embeddings = embeddings / norm
    valid_embeddings = tf.nn.embedding_lookup(normalized_embeddings, valid_dataset)
    similarity = tf.matmul(valid_embeddings, tf.transpose(normalized_embeddings))
    

#### **12\. 训练 CBOW 模型**

    
    
    num_steps = 100001
    cbow_losses = []
    
    with tf.Session(config=tf.ConfigProto(allow_soft_placement=True)) as session:        # ConfigProto 可以提供不同的配置设置
    
        tf.global_variables_initializer().run()                                            # 初始化变量
        print('Initialized')
    
        average_loss = 0
    
        # 训练 Word2vec 模型 num_step 次
        for step in range(num_steps):
    
            # 生成一批数据
            batch_data, batch_labels = generate_batch_cbow(batch_size, window_size)
    
            # 运行优化算法，计算损失
            feed_dict = {train_dataset : batch_data, train_labels : batch_labels}
            _, l = session.run([optimizer, loss], feed_dict=feed_dict)
    
            # 更新平均损失变量
            average_loss += l
    
            if (step+1) % 2000 == 0:
                if step > 0:
                    average_loss = average_loss / 2000
                    # 计算一下平均损失
                cbow_losses.append(average_loss)
                print('Average loss at step %d: %f' % (step+1, average_loss))
                average_loss = 0
    
            # 评估 validation 集的单词相似性
            if (step+1) % 10000 == 0:
                sim = similarity.eval()
                # 对验证集的每个用于验证的中心词，计算其 top_k 个最近单词的余弦距离，
                for i in range(valid_size):
                    valid_word = reverse_dictionary[valid_examples[i]]
                    top_k = 8                                                             # 近邻的数目
                    nearest = (-sim[i, :]).argsort()[1:top_k+1]
                    log = 'Nearest to %s:' % valid_word
                    for k in range(top_k):
                        close_word = reverse_dictionary[nearest[k]]
                        log = '%s %s,' % (log, close_word)
                    print(log)
        cbow_final_embeddings = normalized_embeddings.eval()
    
    
    np.save('cbow_embeddings',cbow_final_embeddings)
    
    with open('cbow_losses.csv', 'wt') as f:
        writer = csv.writer(f, delimiter=',')
        writer.writerow(cbow_losses)
    
    
    
    ...
    Average loss at step 92000: 2.097807
    Average loss at step 94000: 2.097366
    Average loss at step 96000: 2.105257
    Average loss at step 98000: 2.099399
    Average loss at step 100000: 2.106109
    Nearest to for: wildcard, winning, lysander, uhl, jet-bomber, 1761., mccain, luftflotte,
    Nearest to but: which, however, though, although, erudite, theophany, shabery, arose,
    Nearest to its: their, his, pressed, bayous, transylvania, rewrites, unattainability, nbc,
    Nearest to ): tolkāppiyam, analogue, belleforest, longevity, ironically, hillsborough, 1½, corpses,
    Nearest to 's: s, ’, rapping, gerald, grandsons, bigger, fervent, ancyra,
    Nearest to :: hettangian, ;, land-mass, vinegar, 1404, pelageya, begs, johnson-sirleaf,
    Nearest to not: never, n't, haemophilus, hoping, still, prescot, 78.6, 1644.,
    Nearest to to: 1½, will, might, towards, ticket, would, toward, could,
    Nearest to be: remain, being, been, stipulations, lead, olson, ottawa-carleton, exist,
    Nearest to was: is, were, became, copacabana, been, pentiums, remains, be,
    Nearest to with: tinged, farmington, featuring, vice-presidential, clarify, surveying, puddle, deep-seated,
    Nearest to and: nicos, pollstar, or, samantha, mihály, cornea, fended, iaea,
    Nearest to has: have, had, having, is, substitutes, contains, consists, agraristas,
    Nearest to on: upon, palomar, undetected, carranza, transfiguration, spits, hiroshi, standalone,
    Nearest to it: she, he, itself, realizing, irae, macs, hitler, aztlán,
    Nearest to were: are, was, auditing, reshaped, ny, constitute, being, bulldozed,
    

在结果中我们可以看到与验证样本集相近的前 8 个单词，例如与 but 相近的有 which、however、though、although，与 it
相近的有 she、he、itself，与 were 相近的有 are、was。

#### **13\. 可视化 embedding 的结果**

这一步我们会用到 t-SNE 算法（t-Distributed Stochastic Neighbor
Embedding），它是一种用于可视化高维数据的方法，由 Laurens van der Maatens 和 Geoffrey Hinton 在 2008
年提出。

**1\. 首先找到聚集在一起的单词** ，其他一些离散的单词就不显示出来了。

    
    
    def find_clustered_embeddings(embeddings,distance_threshold,sample_threshold):
    
        # 计算余弦距离
        cosine_sim = np.dot(embeddings,np.transpose(embeddings))
        norm = np.dot(np.sum(embeddings**2,axis=1).reshape(-1,1),np.sum(np.transpose(embeddings)**2,axis=0).reshape(1,-1))
        assert cosine_sim.shape == norm.shape
        cosine_sim /= norm
    
        np.fill_diagonal(cosine_sim, -1.0)
    
        argmax_cos_sim = np.argmax(cosine_sim, axis=1)
        mod_cos_sim = cosine_sim
        # 找到每次循环的最大值，来计数是否超过阈值
        for _ in range(sample_threshold-1):
            argmax_cos_sim = np.argmax(cosine_sim, axis=1)
            mod_cos_sim[np.arange(mod_cos_sim.shape[0]),argmax_cos_sim] = -1
    
        max_cosine_sim = np.max(mod_cos_sim,axis=1)
    
        return np.where(max_cosine_sim>distance_threshold)[0]
    

**2\. 用 Scikit-Learn 计算词嵌入的 t-SNE 可视化。**

    
    
    num_points = 1000 # 用一个大的样本空间来构建 T-SNE，然后用余弦相似性对其进行修剪
    
    tsne = TSNE(perplexity=30, n_components=2, init='pca', n_iter=5000)
    
    print('Fitting embeddings to T-SNE. This can take some time ...')
    
    selected_embeddings = cbow_final_embeddings[:num_points, :]
    two_d_embeddings = tsne.fit_transform(selected_embeddings)
    
    print('Pruning the T-SNE embeddings')
    
    # 修剪词嵌入，只取超过相似性阈值的 n 个样本，使可视化变得整齐一些
    
    selected_ids = find_clustered_embeddings(selected_embeddings,.25,10)
    two_d_embeddings = two_d_embeddings[selected_ids,:]
    
    print('Out of ',num_points,' samples, ', selected_ids.shape[0],' samples were selected by pruning')
    

> Fitting embeddings to T-SNE. This can take some time ... Pruning the T-SNE
> embeddings Out of 1000 samples, 577 samples were selected by pruning

**3\. 用 Matplotlib 画出 t-SNE 的结果。**

    
    
    def plot(embeddings, labels):
    
      n_clusters = 20                                                                     # clusters 的数量
    
      label_colors = [pylab.cm.Spectral(float(i) /n_clusters) for i in range(n_clusters)]        # 为每个 cluster 自动分配颜色
    
      assert embeddings.shape[0] >= len(labels), 'More labels than embeddings'
    
      # 定义 K-Means
      kmeans = KMeans(n_clusters=n_clusters, init='k-means++', random_state=0).fit(embeddings)
      kmeans_labels = kmeans.labels_
    
      pylab.figure(figsize=(15,15))  
    
      # 画出所有 embeddings 和他们相应的单词
      for i, (label,klabel) in enumerate(zip(labels,kmeans_labels)):
        x, y = embeddings[i,:]
        pylab.scatter(x, y, c=label_colors[klabel])    
    
        pylab.annotate(label, xy=(x, y), xytext=(5, 2), textcoords='offset points',
                       ha='right', va='bottom',fontsize=10)
    
      pylab.show()
    
    words = [reverse_dictionary[i] for i in selected_ids]
    plot(two_d_embeddings, words)
    

![](https://images.gitbook.cn/71240f70-4e6a-11ea-96da-c5d22898644a)

从这个图中我们可以比较看到几个非常明显的结果，模型将所有的数字 0~9 聚在了一起，将月份聚在了一起，将介词聚成了一堆，还有
king、chief、president 这种意思相近的词也聚在了一起。

* * *

今天我们介绍了 CBOW 和层级 Softmax，负采样两种改进思路，下次我们将介绍 word2vec 的另一种训练方法 Skip-
Gram，以及结合共现矩阵和 word2vec 二者优点的 GloVe。

参考文献：

  * Xin Rong，[ _word2vec Parameter Learning Explained_](https://arxiv.org/pdf/1411.2738.pdf)
  * Morin and Bengio (2005)，[ _Hierarchical Probabilistic Neural Network Language Model_](https://www.iro.umontreal.ca/~lisa/pointeurs/hierarchical-nnlm-aistats05.pdf)
  * Mikolov et al. (2013)，[ _Distributed Representations of Words and Phrases and their Compositionality_](https://papers.nips.cc/paper/5021-distributed-representations-of-words-and-phrases-and-their-compositionality.pdf)
  * Thushan Ganegedara， _Natural Language Processing with TensorFlow_

