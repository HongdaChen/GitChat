在一头扎入到具体的原理 $^1$ 中去之前，我们先再来回顾一下 Word2Vec 样本的生成过程。

### 样本生成

为了方便演示，语料库仅有一句话为：

    
    
    无论精神多么独立的人，感情却总是在寻找一种依附，寻找一种归宿。
    

分词后的词汇表为（形式为：序号 词汇）

    
    
    0 无论 1 精神 2 多么 3 独立 4 人 5 感情 6 总是 7 依附 8 归宿 9 寻找
    

由上一篇文章我们知道，Word2Vec 的输入为一个词，输出是输入周围的词。而这个“周围”就是由 **窗口** 来定义的。

例如，如果 **窗口长度** （Windows Length）是 1，那么“多么”就与“精神”在同一个窗口内，与“无论”就不在一个窗口内。

如果 **窗口长度** 是 2，那么“多么”就与“无论”在同一个窗口内了。

处在同一个窗口内的词又被叫做处在同一个 **上下文** （Context）中。

**窗口** 的概念如下图所示：

> Word2Vec 均基于 Skip-Gram，对 CBOW 感兴趣的同学可以自行搜索。

![](https://images.gitbook.cn/689b0440-2762-11eb-be76-5555cd71111f)

此时输入为“独立”，因为窗口长度是 2，生成的训练样本为:

输入 | 输出  
---|---  
独立 | 精神  
独立 | 精神  
独立 | 精神  
独立 | 精神  
  
用同样的方法可以得到所有的训练输入输出。

好了，明白样本的生成逻辑之后，咱们可以对于 Word2Vec 的参数更新过程一探究竟了。

强烈建议读者们看一看这三篇 paper：

  * [Efficient Estimation of Word Representation in Vector Space, 2013](https://arxiv.org/pdf/1301.3781.pdf)
  * [Distributed Representations of Sentences and Documents, 2014](http://papers.nips.cc/paper/5021-distributed-representations-of-words-and-phrases-and-their-compositionality.pdf)
  * [Enriching Word Vectors with Subword Information, 2016](https://www.mitpressjournals.org/doi/abs/10.1162/tacl_a_00051)

### Skip-Gram

如下如所示，是 Skip-Gram 模型的结构：

![](https://images.gitbook.cn/b7c2fe60-2762-11eb-9825-558323e50dbd)

所用到的符号说明如下：

  * $V$ 是词汇表中词的个数。
  * $C$ 是 context（上下文）长度，例如上一小节的窗口长度是 2 时，对应的上下文长度就是 4。
  * $x_i$ 是输入的 one-hot 表示，对应上一小节的“独立”。$y_{c,j}$ 是输出的 one-hot 表示，$i,j \in [0,V), c \in [0, 2*win\\_len]$。
  * $W_{V \times N}$ 是输入层到隐藏层的矩阵，称为输入矩阵。
  * $W'_{N \times V}$ 是隐藏层到输出层的矩阵，称为输出矩阵。
  * 每个词有两个向量表示，一个来自 $W_{V \times N}$ 的行向量，一个来自 $W'_{N \times V} $ 的列向量，模型最终产出的词向量来自于前者。

这里之所以有 $C$ 个输出，是因为这是 Skip-Gram 模型，由中心词 去预测周围词。例如 输入为“独立”，输出就是 4 个词。

好，熟悉了符号之后，我们就开始 **公式推导** 之旅。

> 从模型结构可以看出，Skip-Gram 隐藏层的激活函数是——线性激活函数。

### 前向传播

#### **输入层到隐藏层**

$\boldsymbol x$ 对应的是输入词的 one-hot 表示，也即 $\boldsymbol x = \\{x_1, x_2, ...,
x_V\\}$，向量中的元素：

$$x_i =\begin{cases}1, & i = k \\\\[1ex]0, & otherwise\end{cases}$$

隐藏层的向量表示 $v_{w_0} = x^T W$，显然 $v_{w0}$ 是矩阵 $W$ 的第 $k$ 行。

我们定义 $w_0$ 为输入词，周围词为 $context(w_0)$，用矩阵运算简单说明 $v_{w_0}$ 的具体由来。

$$\begin{align}\boldsymbol x &= \begin{bmatrix}x_1 \\\ x_2 \\\ \vdots \\\ x_k
\\\ \vdots \\\ x_V \end{bmatrix}\end{align}$$

$$\begin{align}W_{V \times N} &= \begin{bmatrix}w_{11} & w_{12} & \cdots &
w_{1N} \\\ w_{21} & w_{22} & \cdots & w_{2N} \\\ \vdots & \vdots &\ddots &
\vdots \\\w_{k1} & w_{k2} & \cdots & w_{kN} \\\ \vdots & \vdots &\ddots &
\vdots \\\w_{V1} & w_{V2} & \cdots & w_{VN} \\\ \end{bmatrix}\end{align}$$

$$\begin{align}\boldsymbol v_{w_0} &= x^TW \\\&= \begin{bmatrix}x_1 & x_2 &
\cdots & x_k & \cdots & x_V \end{bmatrix} \begin{bmatrix}w_{11} & w_{12} &
\cdots & w_{1N} \\\ w_{21} & w_{22} & \cdots & w_{2N} \\\ \vdots & \vdots
&\ddots & \vdots \\\w_{k1} & w_{k2} & \cdots & w_{kN} \\\ \vdots & \vdots
&\ddots & \vdots \\\w_{V1} & w_{V2} & \cdots & w_{VN} \\\ \end{bmatrix}\\\& =
\begin{bmatrix}x_kw_{k1} & x_kw_{k2} & \cdots & x_kw_{kN} \end{bmatrix} \\\& =
\begin{bmatrix}w_{k1} & w_{k2} & \cdots & w_{kN} \end{bmatrix} \tag{1}
\end{align}$$

#### **隐藏层到输出层**

虽然有 $y_{1,j}, y_{2,j}, \cdots, y_{C,j}$ 个输出，但是其实它们都共享 $W'_{N \times V}$，所以就简化成
$y_j$，表示输入是 $w_0$ 时预测词是 $w_j$ 的概率。

$y_j$ 的计算也是异常的简单：

$$y_j = p(w_j|w_0) = σ(v'^T_{w_j} \cdot v_{w0}) \in(0,1) \tag{2}$$

其中，$v'_{wj}$ 是词 $w_j$ 在矩阵 $W'$ 中的列向量，$v_{w0}$ 是词 $w_0$ 在矩阵 $W$ 中的行向量。别弄错了哦。

对于 $w_j \in context(w_0)$ 我们希望 $y_j$ 越接近 1 越好，对于 $w_j \notin context(w_0)
$（负采样得到的词），$y_j$ 越接近 0 越好，即 $1 - y_j$ 越接近 1 越好。

你想到了什么？是的，这是典型的逻辑回归，此时的多分类问题已经转化成了二分类问题。

对于 1 个周围词 $w_{o,c}$，negative 个负采样词 ${w_i}, i \in [1, negative]$ ，希望最大化以下公式：

$$\begin{align}LL &= \sigma(v'^T_{w_{o,c}} \cdot v_{w_0})
\prod_{i=1}^{negative}(1-σ(v'^T_{w_i} \cdot v_{w0})) \\\\\end{align}$$

根据最大似然，很容易可以得到损失函数：

$$\begin{align}E &= -log \ LL \\\& = -log \ [\sigma(v'^T_{w_{o,c}} \cdot
v_{w_0}) \prod_{i=1}^{negative}(1-\sigma(v'^T_{w_i} \cdot v_{w0}))]\\\& =
-(log \ \sigma(v'^T_{w_{o,c}} \cdot v_{w_0}) +\sum_{i=1}^{negative}log\
(1-\sigma(v'^T_{w_i} \cdot v_{w0}))) \end{align} \tag{4}$$

现在有了 loss，就可以对模型中的参数进行梯度计算，并更新了。

### 反向传播

#### **输出层到隐藏层**

首先来看看矩阵 $W'$ 如何更新。

根据公式 $(4)$，对 $v'_w$ 求导，得：

$$\frac{\partial E} {\partial v'_w} = \begin{cases}(\sigma(v'^T_{w_{o,c}}
\cdot v_{w_0}) - 1)v_{w_0}, & w = w_{o,c} \\\\[1ex](\sigma(v'^T_w \cdot
v_{w_0}) - 0)v_{w_0}, & otherwise\end{cases}$$

由此，假设学习率为 $\eta$，则 $v'_w$ 的更新公式为：

$$v'^{new}_w = v'^{old}_{w} - \eta\frac{\partial E} {\partial v'_w} \tag{5}$$

#### **隐藏层到输入层**

再来看看 $W$ 如何更新。

$$\frac{\partial E} {\partial v_w} = \begin{cases}(\sigma(v'^T_{w_{o,c}} \cdot
v_{w_0}) - 1)v'_{w_{o,c}}, & w = w_{o,c} \\\\[1ex](\sigma(v'^T_w \cdot
v_{w_0}) - 0)v'_w, & otherwise\end{cases}$$

由此，假设学习率为 $\eta$，则 $v_w$ 的更新公式为：

$$v^{new}_w = v^{old}_{w} - \eta\frac{\partial E} {\partial v_w} \tag{6}$$

### Negative Sampling

上一篇文章，咱们只是粗略直观地说了一下为什么要负采样，没有从理论上说明。理论说明可以由公式 $(4)$ 得到，第 2
项的计算量太大了，也就是每次更新一个参数就要计算 $V$ 次累加，如果 $V$ 在百万、千万或者亿级，那么 Word2Vec 只能呆坐在地上，啥也做不了了。

[这篇 paper](http://papers.nips.cc/paper/5021-distributed-representations-of-
words-and-phrases-and-their-compositionality.pdf) 告诉我们，可以不计算 $V$
次，如果数据集足够多，只需选择 2~5 个词作为负例就可以了。而这 2~5 个词是从 $V$ 个词中概率随机出来的。具体地，在每次训练时，词 $w_i$
被选作负例的概率 $P(w_i)$ 为：

$$P(w_i) = \frac{f(w_i)^{3/4}} {\sum_{j=0}^Vf(w_j)^{3/4}}$$

其中，$f(w_i)$ 是词 $w_i$ 在数据集中出现的次数。

你要问为啥会设计出这么一个函数，作者在 paper 中说这是他试出来，效果最好的一个。

### Sub-Sampling$^2$

同样上一篇我们也提到了，出现次数比较多的词会被概率保留作为训练样本，那么这个保留概率又是多少呢？

> paper 中给出的概率为 $P(w_i) = \sqrt\frac{t}{f(w_i)}$。

在代码实现时，作者实现的与 paper 中并不一致，做了些许修改，我想代码中的实现应该更能代表作者的意图吧。

$$P(w_i) = (\sqrt{\frac{z(w_i)}{0.001}} + 1) \cdot \frac{0.001}{z(w_i)}$$

其中，$z(w_i)$ 是词 $w_i$ 在数据集中出现的次数除以数据集中词的总个数(不去重)。比如第一小节中的“寻找”出现了 2 次，数据集中词的总个数
10 个，那么 $z(w_i)$ 就是 0.2。

来看一下这个函数的曲线图：

![](https://images.gitbook.cn/e7551310-2763-11eb-b5c5-e14bfa43e9db)

  * $P(w_i)$ = 1 时，$z(w_i)$ < 0.0016，也就是说只有占比大于 0.0016 的词才会在训练时可能会被丢弃。
  * $P(w_i)$ = 0.5 时，$z(w_i)$ = 0.00745，即一半的可能性会被丢弃。
  * 当 $z(w_i)$ = 1 时，$P(w_i)$ = 0.033，如果整个数据集就一个词$w_i$，其被丢弃的概率为 96.7%。

### 小结

本篇稍微有点理论性，不过并不难懂，Word2Vec 作为词向量的开始鼻祖，三大 paper 绝对值得一读。

我们首先给出了前向传播的公式推导，得到 loss 后，再反向传播得到最终的词向量矩阵 $W$ 的更新公式。

从现在来看 Word2Vec 是最最基本的网络结构，只有 1 层隐藏层，且激活函数为线性函数。如此简单的结构却给业界带来了深刻久远的影响。

我们也应该关注 Word2Vec 在实现时做的一些 tricks，比如 Negative Sampling 和 Sub
Sampling。掌握这么做的原因是什么。

下一篇，咱们来看看基于 Spark 的 Skip-Gram Word2Vec 实现，并探讨其在电商中如何应用。

咱们，下篇见。

* * *

注释 1：本文很多内容来自于[这篇文章](http://www.1-4-5.net/~dmm/ml/how_does_word2vec_work.pdf)

注释 2：[Word2Vec 作者源码](https://github.com/tmikolov/word2vec)

