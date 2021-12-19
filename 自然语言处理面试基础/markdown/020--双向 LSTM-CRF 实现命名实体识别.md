今天我们来学习 NER——Named entity
recognition，命名实体识别，即识别出文档中具有特定意义的实体，例如人名、地名、机构名、专有名词等。

命名实体识别主要用来提取结构化信息，是信息提取、问答系统、句法分析、机器翻译等应用领域的重要工具。例如，可以用来自动识别简历中的电子邮件、电话号码、学位信息；可以从法律、金融和医疗等领域的文档中提取重要的实体，用于后续的分类和搜索等；用于识别新闻中的人物、机构、地点等标签，进而自动做文档分类；用
NER
来推荐与实体相关的内容，例如当用户浏览了一篇文章后，可以向他推荐具有相似实体的其他文章；还可以用于客服系统，当用户们写评论抱怨产品时，可以自动提取其中的地点、产品等实体信息，然后分配给相应的部门处理等等。

实现 NER 同样也有基于规则的传统方法，机器学习算法，还有深度学习算法，目前 state of art 的模型列表如下图所示：

![](https://images.gitbook.cn/3549f500-7426-11ea-8fa3-2df515022d48)

[数据来源](http://nlpprogress.com/english/named_entity_recognition.html)

今天我们先来学习最基础的 [BiLSTM-CRF](https://arxiv.org/pdf/1508.01991.pdf)，ELMo、BERT
我们会在后面的课程中慢慢深入。

### BiLSTM-CRF 模型的原理

为了简化说明，这里我们假设只有 **人物和机构** 两类实体，于是有下面五种实体标签：

    
    
    B-Person
    I-Person
    B-Organization
    I-Organization
    O
    

其中 B 的意思是 beginning，比如一个人名的第一个字母，I 的意思是 inside，如一个人名中的其他字母，O 为非实体。

#### **首先看一下 BiLSTM-CRF 模型的结构**

如下图所示：

![](https://images.gitbook.cn/6f004790-7426-11ea-bf4b-4f99c7eed580)

[图片来源](https://www.researchgate.net/publication/336113245_Ontology-
Based_Healthcare_Named_Entity_Recognition_from_Twitter_Messages_Using_a_Recurrent_Neural_Network_Approach)

**模型的输入是一个句子** ，如

$$s = w_0,w_1,w_2,w_3,w_4$$

其中 $w_0$、$w_1$ 的实际标签是人物实体，$w_3$ 是机构实体， **我们的目标是识别出这三个单词的正确实体标签** 。

  * 首先需要将单词转化为词向量，然后输入给 BiLSTM-CRF 模型。
  * 在模型中，要先经过 BiLSTM 层，这一层的输出是每个单词在所有标签上的得分，这个得分叫做 **发射分数** 。如 $w_0$ 的得分可能是 1.5（B-Person）、0.9（I-Person）、0.1（B-Organization）、0.08（I-Organization）、0.05（O）。
  * 将发射分数输入给 CRF 层，输出是 **得分最高** 的那个标签序列。
  * 除了发射分数，模型中还会用到一个分数叫做 **转移分数** ，是指标签与标签之间的转移概率。标签之间的转移分数可以构造出一个转移矩阵，这个转移矩阵是 BiLSTM-CRF 模型的一个参数矩阵，训练开始时进行初始化，训练过程中这些分数会自动更新。

#### **模型中 CRF 层的作用**

BiLSTM 的输出就已经是每个单词在所有标签上的得分，之所以再加上一个 CRF 层，是因为 CRF
层可以从数据中学习出一些限制规则，并保证最后的预测结果是符合这些规则的，例如：

  * 一个标签的开头必须是“B-”或者“O”，而不应该出现“I-“。
  * 不应该出现“B-Person I-Organization”这样的标签，BI～ 必须指的是同一个实体。

**CRF 是如何学习到这些规则的呢？**

前面提到了转移矩阵，CRF 能学习到的限制规则就是从这个转移矩阵中体现出来的：

比如我们可以从上图中学习到下面这几个规则：

  * 一个标签的开头必须是“B-”或者“O”，而不应该出现“I-”：因为“START”到“I-Person 或者 I-Organization”的转移概率很低。
  * 不应该出现“B-Person I-Organization”这样的标签，BI～ 必须指的是同一个实体：因为“B-Organization”到“I-Person”的转移概率只有 0.0003，比其他的要小。

#### **模型的损失函数是什么？**

接下来我们进入具体原理部分， **首先定义一下数学符号。**

例如我们有一个句子包含 5 个单词，标签除了上面 5 个外，需要另外加两个标签，START 表示句子开始，END 表示句子结尾。先给每个标签设置序号：

    
    
    B-Person：0
    I-Person：1
    B-Organization：2
    I-Organization：3
    O：4
    START：5
    END    ：6
    

用 $x_{iy_j}$ 表示第 i 个单词在第 j 个标签上的**发射分数**，例如：$x_{i=1,y_j=2} =
x_{w_1,B-Organization} = 0.1$ 表示 $w_1$ 是 B-Organization 的分数为 0.1。

用 $t_{y_iy_j}$ 表示**转移分数**，例如 $t_{B-Person, I-Person} = 0.9$ 表示 标签 B-Person
转移到标签 I-Person 的分数是 0.9。

不考虑规则的话，这 5 个单词组成的句子所对应的标签序列会有 N=5^5 种组成情况：

    
    
    1) START B-Person B-Person B-Person B-Person B-Person END
    2) START B-Person I-Person B-Person B-Person B-Person END
    …
    10) START B-Person I-Person O B-Organization O END
    …
    N) O O O O O O O
    

我们将第 i 种序列组成情况的分数记为 $P_{i} = e^{S_i}$， 将所有组成情况的总分数记为

$$P_{total} = P_1 + P_2 + … + P_N = e^{S_1} + e^{S_2} + … + e^{S_N}$$

我们希望模型可以学习出真实的标签序列，也就是 **希望真实标签序列的分数是所有可能情况中分数最高的** 。

于是 CRF 层的损失函数为：

$$Loss Function = \frac{P_{RealPath}}{P_1 + P_2 + … + P_N}$$

  * 分子：真实标签序列的分数
  * 分母：所有可能的标签序列组成情况的总分数

对损失取 log 对数，将除式转化为减式，再取负号将求函数的最大值转化为求最小值。

于是损失函数变为：

$$\begin{align}Log Loss Function & = - \log \frac{P_{RealPath}}{P_1 + P_2 + …
+ P_N} \\\& = - \log \frac{e^{S_{RealPath}}}{e^{S_1} + e^{S_2} + … + e^{S_N}}
\\\& = - (\log(e^{S_{RealPath}}) - \log(e^{S_1} + e^{S_2} + … + e^{S_N})) \\\&
= - (S_{RealPath} - \log(e^{S_1} + e^{S_2} + … + e^{S_N})) \\\& = - (
\sum_{i=1}^{N} x_{iy_i} + \sum_{i=1}^{N-1} t_{y_iy_{i+1}} - \log(e^{S_1} +
e^{S_2} + … + e^{S_N})) \\\\\end{align}$$

#### **如何计算损失函数？**

先看如何计算 $P_{i} = e^{S_i} $。

$$S_i = EmissionScore + TransitionScore$$

假设这 5 个单词的正确标签序列是：

    
    
    START B-Person I-Person O B-Organization O END
    

那么这个序列的发射分数 EmissionScore 为：

$$EmissionScore=x_{0,START}+x_{1,B-Person}+x_{2,I-Person}+x_{3,O}+x_{4,B-Organization}+x_{5,O}+x_{6,END}$$

其中 $x_{0,START}$ 和 $x_{6,END}$ 设置为 0。

$$x_{1,B-Person}，x_{2,I-Person}，x_{3,O}，x_{4,B-Organization}，x_{5,O}$$

是 BiLSTM 层的输出结果。

**这个序列的转移分数为：**

$$TransitionScore=t_{START->B-Person} + t_{B-Person->I-Person} +
t_{I-Person->O} + t_{0->B-Organization} + t_{B-Organization->O} + t_{O->END}$$

这些转移分数来自 CRF 层，是 CRF 层的参数。

再看如何计算上式中的 $\log(e^{S_1} + e^{S_2} + … + e^{S_N})$。

这里用到了动态规划的思想来高效计算

$$\log(e^{S_1} + e^{S_2} + … + e^{S_N})$$

其中

$$S_i = EmissionScore + TransitionScore$$

为了简化说明，我们将例子缩减为 3 个单词 $\mathbf{x} = [w_0, w_1, w_2]$ 和 2 个标签 $LabelSet =
\\{l_1,l_2\\}$。

这样在每个单词位置处有两种标签选择，所以有 2^3=8 种路径组合，我们需要将这 8 种路径的分数都计算出来。

在计算过程中，我们需要两个变量：

  * obs 表示当前状态的发射分数
  * previous 表示到前一状态为止的所有路径的分数

下面一步步计算。

**1\. 当前状态为 $w_0$ 时**

$$obs = [x_{01}, x_{02}]$$

表示 $w_0$ 可以有两种状态，一个是标签为 l1，一个是标签为 l2。

$$previous = None$$

此时前面没有任何路径所以是 none。

这时总分数为：

$$TotalScore(w_0)=\log (e^{x_{01}} + e^{x_{02}})$$

**2\. 当前状态为 $w_1$ 时**

即从 $w_0$ 走到 $w_1$，

$$obs = [x_{11}, x_{12}]$$

表示 $w_1$ 可以有两种状态，一个是标签为 l1，一个是标签为 l2

$$previous = [x_{01}, x_{02}]$$

previous 有两个元素，previous[0] 表示到前一时刻为止，最终状态走到标签 l1 的所有路径总分数，previous[1]
表示到前一时刻为止，最终状态走到标签 l2 的所有路径总分数。

  * 当 $w_1$ 取标签 l1 时，它可以从前一时刻以 l1 结尾的路径走到这里，此时需要转移分数 $t_{11}$，也可以从前一时刻以 l2 结尾的路径走到这里，此时需要转移分数 $t_{21}$。
  * 当 $w_1$ 取标签 l2 时，它可以从前一时刻以 l1 结尾的路径走到这里，此时需要转移分数 $t_{12}$，也可以从前一时刻以 l2 结尾的路径走到这里，此时需要转移分数 $t_{22}$。

这样的话转移矩阵为：

$$\begin{pmatrix}t_{11} & t_{12}\\\t_{21} & t_{22}\end{pmatrix}$$

为了方便计算

$$S_i = EmissionScore + TransitionScore$$

我们将 obs 和 previous 也按求和对应位置扩展成相应的 2*2 矩阵形式：

$$obs =\begin{pmatrix}x_{11}&x_{12}\\\x_{11}&x_{12}\end{pmatrix}$$

$$previous =\begin{pmatrix}pre[0] & pre[0]\\\pre[1] & pre[1]\end{pmatrix}$$

于是分数等于三个矩阵的和：

$$scores
=\begin{pmatrix}x_{01}+x_{11}+t_{11}&x_{01}+x_{12}+t_{12}\\\x_{02}+x_{11}+t_{21}&x_{02}+x_{12}+t_{22}\end{pmatrix}$$

**然后我们需要更新 previous：**

  * previous[0] 为 scores 的第一列的总分数，表示目前为止以标签 l1 结尾的所有路径总分数；
  * previous[1] 为 scores 的第二列的总分数，表示目前为止以标签 l2 结尾的所有路径总分数。

这样，从 $w_0$ 到 $w_1$ 的总分数为：

$$\begin{align}TotalScore(w_0 → w_1)&= \log (e^{previous[0]} +
e^{previous[1]}) \\\&= \log (e^{\log(e^{x_{01}+x_{11}+t_{11}}
+e^{x_{02}+x_{11}+t_{21}})}+e^{\log(e^{x_{01}+x_{12}+t_{12}} +
e^{x_{02}+x_{12}+t_{22}})}
)\\\&=\log(e^{x_{01}+x_{11}+t_{11}}+e^{x_{02}+x_{11}+t_{21}}+e^{x_{01}+x_{12}+t_{12}}+e^{x_{02}+x_{12}+t_{22}})
\\\\\end{align}$$

从最后结果可以看出，正好得到了 $w_0$ 到 $w_1$ 的 4 种可能路径的总分数。

**3\. 当前状态为 $w_2$ 时**

即从 $w_0$ 走到 $w_1$，再走到 $w_2$，这一时刻的思考方式和上一时刻是一模一样的，即 $w_2$ 可以有两种状态，一个是标签为
l1，一个是标签为 l2。

$$obs = [x_{21}, x_{22}]$$

$$previous=[\log (e^{x_{01}+x_{11}+t_{11}} + e^{x_{02}+x_{11}+t_{21}}), \log
(e^{x_{01}+x_{12}+t_{12}} + e^{x_{02}+x_{12}+t_{22}})]$$

previous[0] 表示到前一时刻为止，最终状态走到标签 l1 的所有路径总分数，previous[1] 表示到前一时刻为止，最终状态走到标签 l2
的所有路径总分数。

  * 当 $w_2$ 取标签 l1 时，它可以从前一时刻以 l1 结尾的路径走到这里，此时需要转移分数 $t_{11}$，也可以从前一时刻以 l2 结尾的路径走到这里，此时需要转移分数 $t_{21}$；
  * 当 $w_2$ 取标签 l2 时，它可以从前一时刻以 l1 结尾的路径走到这里，此时需要转移分数 $t_{12}$，也可以从前一时刻以 l2 结尾的路径走到这里，此时需要转移分数 $t_{22}$。

同样，为了计算分数，将 obs 和 previous 扩展成 2*2 矩阵：

$$obs =\begin{pmatrix}x_{21}&x_{22}\\\x_{21}&x_{22}\end{pmatrix}$$

$$previous
=\begin{pmatrix}\log(e^{x_{01}+x_{11}+t_{11}}+e^{x_{02}+x_{11}+t_{21}})&\log(e^{x_{01}+x_{11}+t_{11}}
+
e^{x_{02}+x_{11}+t_{21}})\\\\\log(e^{x_{01}+x_{12}+t_{12}}+e^{x_{02}+x_{12}+t_{22}})&\log(e^{x_{01}+x_{12}+t_{12}}+e^{x_{02}+x_{12}+t_{22}})\end{pmatrix}$$

于是 scores 等于：

$scores =\begin{pmatrix}\log (e^{x_{01}+x_{11}+t_{11}}
+e^{x_{02}+x_{11}+t_{21}}) + x_{21} + t_{11}&\log (e^{x_{01}+x_{11}+t_{11}}
+e^{x_{02}+x_{11}+t_{21}}) + x_{22} + t_{12}\\\\\log (e^{x_{01}+x_{12}+t_{12}}
+ e^{x_{02}+x_{12}+t_{22}}) + x_{21} + t_{21}&\log (e^{x_{01}+x_{12}+t_{12}}
+e^{x_{02}+x_{12}+t_{22}}) + x_{22} + t_{22}\end{pmatrix}$

同样，需要更新 previous：

$previous = [\log( e^{\log (e^{x_{01}+x_{11}+t_{11}} +
e^{x_{02}+x_{11}+t_{21}}) + x_{21} + t_{11}}+e^{\log (e^{x_{01}+x_{12}+t_{12}}
+ e^{x_{02}+x_{12}+t_{22}}) + x_{21} + t_{21}}),\log(e^{\log
(e^{x_{01}+x_{11}+t_{11}} + e^{x_{02}+x_{11}+t_{21}}) + x_{22} +
t_{12}}+e^{\log (e^{x_{01}+x_{12}+t_{12}} + e^{x_{02}+x_{12}+t_{22}}) + x_{22}
+ t_{22}})]$

$=[\log( (e^{x_{01}+x_{11}+t_{11}} + e^{x_{02}+x_{11}+t_{21}})e^{x_{21} +
t_{11}} + (e^{x_{01}+x_{12}+t_{12}} + e^{x_{02}+x_{12}+t_{22}})e^{x_{21} +
t_{21}} ),\log( (e^{x_{01}+x_{11}+t_{11}} + e^{x_{02}+x_{11}+t_{21}})e^{x_{22}
+ t_{12}} + (e^{x_{01}+x_{12}+t_{12}} + e^{x_{02}+x_{12}+t_{22}})e^{x_{22} +
t_{22}})]$

这时总分数为：

$TotalScore(w_0 → w_1 → w_2 $ $=\log (e^{previous[0]} + e^{previous[1]})$
$=\log (e^{\log( (e^{x_{01}+x_{11}+t_{11}} +
e^{x_{02}+x_{11}+t_{21}})e^{x_{21} + t_{11}} + (e^{x_{01}+x_{12}+t_{12}} +
e^{x_{02}+x_{12}+t_{22}})e^{x_{21} + t_{21}} )}+e^{\log(
(e^{x_{01}+x_{11}+t_{11}} + e^{x_{02}+x_{11}+t_{21}})e^{x_{22} + t_{12}} +
(e^{x_{01}+x_{12}+t_{12}} + e^{x_{02}+x_{12}+t_{22}})e^{x_{22} + t_{22}})} ) $

$=\log(e^{x_{01}+x_{11}+t_{11}+x_{21}+t_{11}}+e^{x_{02}+x_{11}+t_{21}+x_{21}+t_{11}}+e^{x_{01}+x_{12}+t_{12}+x_{21}+t_{21}}+e^{x_{02}+x_{12}+t_{22}+x_{21}+t_{21}}+e^{x_{01}+x_{11}+t_{11}+x_{22}+t_{12}}+e^{x_{02}+x_{11}+t_{21}+x_{22}+t_{12}}+e^{x_{01}+x_{12}+t_{12}+x_{22}+t_{22}}+e^{x_{02}+x_{12}+t_{22}+x_{22}+t_{22}})$

可见结果正好是 8 条路径的总分数。

这样我们就可以计算 CRF 的损失函数了，然后就可以用 TensorFlow Keras 等框架自动训练优化模型。

#### **如何用模型预测新数据？**

当我们知道了 BiLSTM + CRF 模型中的发射分数和转移分数后，就可以预测一个新句子的实体标签序列。

计算过程和上面计算损失函数的分母时是类似的，也是动态规划的思想，区别在于因为要求出一个分数最大的路径，所以 **previous 所存储的是 scores
中每列的最大值。**

我们继续用上面的例子来演算。

**1\. 当前状态为 $w_0$ 时**

$$obs = [x_{01}, x_{02}]$$ $$previous = None$$

此时只有一个单词，很容易根据 $x_{01} 和 x_{02}$ 的大小，来找到 $w_0$ 的最佳标签。

**2\. 当前状态为 $w_1$ 时**

从 $w_0$ 走到 $w_1$：

$$obs = [x_{11}, x_{12}]$$ $$previous = [x_{01}, x_{02}]$$

和上面一样的，为了方便计算，我们将 previous 和 obs 扩展成矩阵形式：

$$obs =\begin{pmatrix}obs[0]&obs[1]\\\obs[0]&obs[1]\end{pmatrix}
=\begin{pmatrix}x_{11}&x_{12}\\\x_{11}&x_{12}\end{pmatrix}$$

$$previous
=\begin{pmatrix}previous[0]&previous[0]\\\previous[1]&previous[1]\end{pmatrix}=\begin{pmatrix}x_{01}&x_{01}\\\x_{02}&x_{02}\end{pmatrix}$$

于是得到分数为：

$$scores =
\begin{pmatrix}x_{01}+x_{11}+t_{11}&x_{01}+x_{12}+t_{12}\\\x_{02}+x_{11}+t_{21}&x_{02}+x_{12}+t_{22}\end{pmatrix}$$

**更新 previous：**

$$previous=[\max (scores[00], scores[10]),\max (scores[01],scores[11])]$$

这里和损失函数那里是有区别的，因为我们最后要的是分数最高的那条路径，所以 **previous 存储的就是到前一时刻为止，以标签 l1
结尾的所有路径中的最高分，和以标签 l2 结尾的所有路径中的最高分。**

例如：

$$scores =
\begin{pmatrix}x_{01}+x_{11}+t_{11}&x_{01}+x_{12}+t_{12}\\\\\underline{x_{02}+x_{11}+t_{21}}&\underline{x_{02}+x_{12}+t_{22}}\end{pmatrix}=\begin{pmatrix}0.2&0.3\\\\\underline{0.5}&\underline{0.4}\end{pmatrix}$$

那么：

$$\begin{align}previous&= [\max (scores[00], scores[10]),\max
(scores[01],scores[11])] \\\&= [0.5, 0.4] \\\\\end{align}$$

同时，我们用 $alpha_0$ 来存储这个最高分的历史记录：

$$\begin{align}alpha_0&= [(scores[10],scores[11])] \\\&= [(0.5,0.4)]
\\\\\end{align}$$

用 $alpha_1$ 来存储最高分所在的位置索引历史：

$$\begin{align}alpha_1&=[(ColumnIndex(scores[10]),ColumnIndex(scores[11]))]
\\\&= [(1,1)] \\\\\end{align}$$

(1,1) 表示标签为 $(l_2,l_2)$。

$alpha_0$ 和 $alpha_1$ 表达的信息为：

  * 在 $w_1$ 这一点，如果 $w_1$ 的标签为 $l_1$，那么从 $w_0$ 走到 $w_1$ 能得到的最高分为 0.5，这条路径来自 $w_0$ 以 l2 为结尾的路径；
  * 如果 $w_1$ 的标签为 $l_2$，那么从 $w_0$ 走到 $w_1$ 能得到的最高分为 0.4，这条路径来自 $w_0$ 以 l2 为结尾的路径。

**3\. 当前状态为 $w_2$ 时**

$$obs = [x_{21}, x_{22}]$$ $$previous = [0.5, 0.4]$$

$$obs
=\begin{pmatrix}obs[0]&obs[1]\\\obs[0]&obs[1]\end{pmatrix}=\begin{pmatrix}x_{21}&x_{22}\\\x_{21}&x_{22}\end{pmatrix}$$

$$previous
=\begin{pmatrix}previous[0]&previous[0]\\\previous[1]&previous[1]\end{pmatrix}=\begin{pmatrix}0.5&0.5\\\0.4&0.4\end{pmatrix}$$

$$scores
=\begin{pmatrix}0.5+x_{11}+t_{11}&0.5+x_{12}+t_{12}\\\0.4+x_{11}+t_{21}&0.4+x_{12}+t_{22}\end{pmatrix}$$

同样更新 previous 为：

$$previous=[\max (scores[00], scores[10]),\max (scores[01],scores[11])]$$

例如，我们得到的分数为：

$$scores = \begin{pmatrix} 0.6&\underline{0.9}\ \underline{0.8}&0.7
\end{pmatrix}$$

那么：

$$previous=[0.8,0.9]$$

$$\begin{align} alpha_0 &= [(0.5,0.4),\underline{(scores[10],scores[01])}] \
&= [(0.5,0.4),\underline{(0.8,0.9)}] \ \end{align}$$

$$alpha_1=[(1,1),\underline{(1,0)}]$$

接下来我们就可以根据 $alpha_0$ 和 $alpha_1$ 所记录的历史分数和位置索引来找到 $w_0$ 到 $w_1$ 再到 $w_2$
所有路径中分数最高的那一条：

$$alpha_0=[(0.5,0.4),\underline{(0.8,0.9)}]$$
$$alpha_1=[(1,1),\underline{(1,0)}]$$

我们从后向前看，在 $alpha_0$ 中 $\underline{(0.8,0.9)} $ 最大的是 0.9，所以 $w_2$ 取 l2，在
$alpha_1$ 中 0.9 对应的索引是 0，意思是这个分数来自 $w_1$ 取 l1 的路径，所以是 l1 → l2。

再向前一步看，如果 $w_1$ 取 l1，此时它对应的分数是 0.5，在 $alpha_1$ 中 0.5 对应的索引是 1，意思是这个分数来自 $w_0$
取 l2 的路径，所以是 l2 → l1。

这样我们就得到了分数最高的一条路径：$l_2 → l_1 → l_2$。

以上就是 BiLSTM + CRF 模型学习和预测的基本原理。

### 用 BiLSTM-CRF 模型实现 NER

我们所用的数据是 [Kaggle 上的一个 NER 数据集](https://www.kaggle.com/abhinavwalia95/entity-
annotated-corpus#ner_dataset.csv)，可以将 ner_dataset.csv 下载到本地。

加载所需的库：

    
    
    !pip install git+https://www.github.com/keras-team/keras-contrib.git
    !pip install seqeval
    
    
    
    import pandas as pd
    import numpy as np
    from keras.preprocessing.sequence import pad_sequences
    from keras.utils import to_categorical
    from sklearn.model_selection import train_test_split
    from keras.models import Model, Input
    from keras.layers import LSTM, Embedding, Dense, TimeDistributed, Dropout, Bidirectional
    from keras_contrib.layers import CRF
    from seqeval.metrics import precision_score, recall_score, f1_score, classification_report
    import matplotlib.pyplot as plt
    

#### **读取数据**

**首先看一下数据长什么样子** ，ner_dataset.csv 这个数据中有 4 列，分别是句子序号，句子中的单词，单词的 POS 标签和 NER
标签，一共有 47959 个句子，包括 35178 个不同的单词，和 17 种 NER 标签。

    
    
    data = pd.read_csv("ner_dataset.csv", encoding="latin1")
    data = data.fillna(method="ffill")
    data.head()
    

![](https://images.gitbook.cn/1c9a6e40-742b-11ea-8dd8-f78c5e1c22c8)

    
    
    words = list(set(data["Word"].values))
    words.append("ENDPAD")
    n_words = len(words)
    n_words 
    # 35179
    
    tags = list(set(data["Tag"].values))
    n_tags = len(tags) 
    n_tags
    # 17
    

**然后构造出模型所需要的数据格式** ，即属于同一句话的单词及其标签放在一个 list 中，SentenceGetter 就是用来构造这样的格式：`单词-
POS标签-NER标签`。

    
    
    class SentenceGetter(object):
    
        def __init__(self, data):
            self.n_sent = 1
            self.data = data
            self.empty = False
            agg_func = lambda s: [(w, p, t) for w, p, t in zip(s["Word"].values.tolist(),
                                                               s["POS"].values.tolist(),
                                                               s["Tag"].values.tolist())]
            self.grouped = self.data.groupby("Sentence #").apply(agg_func)
            self.sentences = [s for s in self.grouped]
    
        def get_next(self):
            try:
                s = self.grouped["Sentence: {}".format(self.n_sent)]
                self.n_sent += 1
                return s
            except:
                return None
    
    getter = SentenceGetter(data)
    sent = getter.get_next()
    sentences = getter.sentences
    print(sent)
    

例如原来的 data 就变成了下面这种形式：

    
    
    [('Thousands', 'NNS', 'O'), ('of', 'IN', 'O'), ('demonstrators', 'NNS', 'O'), ('have', 'VBP', 'O'), ('marched', 'VBN', 'O'), ('through', 'IN', 'O'), ('London', 'NNP', 'B-geo'), ('to', 'TO', 'O'), ('protest', 'VB', 'O'), ('the', 'DT', 'O'), ('war', 'NN', 'O'), ('in', 'IN', 'O'), ('Iraq', 'NNP', 'B-geo'), ('and', 'CC', 'O'), ('demand', 'VB', 'O'), ('the', 'DT', 'O'), ('withdrawal', 'NN', 'O'), ('of', 'IN', 'O'), ('British', 'JJ', 'B-gpe'), ('troops', 'NNS', 'O'), ('from', 'IN', 'O'), ('that', 'DT', 'O'), ('country', 'NN', 'O'), ('.', '.', 'O')]
    

#### **准备数据**

建立单词和 NER 标签的数字索引字典：

    
    
    max_len = 75
    word2idx = {w: i + 1 for i, w in enumerate(words)}
    tag2idx = {t: i for i, t in enumerate(tags)}
    
    
    
    word2idx：
    {'terrorists': 1,
     'Chaka': 2,
     'Sitnikov': 3,
     'bet': 4,
     ...}
    
     tag2idx:
     {'B-art': 3,
     'B-eve': 15,
     'B-geo': 14,
     'B-gpe': 1,
     'B-nat': 0,
     'B-org': 2,
     'B-per': 6,
     'B-tim': 12,
     'I-art': 4,
     'I-eve': 8,
     'I-geo': 9,
     'I-gpe': 16,
     'I-nat': 11,
     'I-org': 5,
     'I-per': 7,
     'I-tim': 10,
     'O': 13}
    

用这两个字典先将文本转化为数字序列，再指定句子的最大长度为 maxlen，如果输入序列超过了这个长度就将被截断，小于这个长度就在序列的尾部填充
n_words-1，注意这里没有用 0 是因为它有对应的单词，所以要填充其他没有意义的词。

    
    
    X = [[word2idx[w[0]] for w in s] for s in sentences]
    X = pad_sequences(maxlen=max_len, sequences=X, padding="post", value=n_words-1)
    

同样，先将每句话中所有单词对应的标签组成的序列 y 转化成数字序列，例如 y[1] 为 [13, 13, 13, 13, 13, 13, 14, 13,
13, 13, 13, 13, 14, 13, 13, 13, 13, 13, 1, 13, 13, 13, 13, 13]，

然后对标签也做 padding 处理，在序列尾部填充标签 "O" 所对应的索引：

    
    
    y = [[tag2idx[w[2]] for w in s] for s in sentences]
    y = pad_sequences(maxlen=max_len, sequences=y, padding="post", value=tag2idx["O"])
    

还需要做一步是将 y 中数字转化为 one-hot 类别数据：

    
    
    y = [to_categorical(i, num_classes=n_tags) for i in y]
    

接着划分训练集和测试集：

    
    
    X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.1)
    

#### **建立 LSTM-CRF 模型**

接下来我们要建立一个双向 LSTM + CRF，模型的原理在上文已经详细地进行了讲解，这里使用 Keras 可以很容易搭建出模型。

    
    
    input = Input(shape=(max_len,))
    model = Embedding(input_dim=n_words + 1, output_dim=20,
                      input_length=max_len, mask_zero=True)(input)  
    model = Bidirectional(LSTM(units=50, return_sequences=True,
                               recurrent_dropout=0.1))(model)  
    model = TimeDistributed(Dense(50, activation="relu"))(model)  
    crf = CRF(n_tags)  
    out = crf(model)  
    
    model = Model(input, out)
    
    model.compile(optimizer="rmsprop", loss=crf.loss_function, metrics=[crf.accuracy])
    
    model.summary()
    

![](https://images.gitbook.cn/84a4a5a0-742b-11ea-af98-5d60dd7fdb7d)

训练模型：

    
    
    history = model.fit(X_tr, np.array(y_tr), batch_size=32, epochs=5,
                        validation_split=0.1, verbose=1)
    

画出学习曲线：

    
    
    hist = pd.DataFrame(history.history)
    plt.style.use("ggplot")
    plt.figure(figsize=(12,12))
    plt.plot(hist["acc"])
    plt.plot(hist["val_acc"])
    plt.show()
    

![](https://images.gitbook.cn/2e690c30-7430-11ea-8dd8-f78c5e1c22c8)

在大概 epoch=2 时最优。

#### **模型评估**

接着我们要用 classification_report 计算 precision、recall、f1-score、support，它们的含义分别是：

  * precision 精度，分类正确的正样本个数占分类器判定为正样本的样本个数的比例，即分类器识别正样本的准确度是多少。
  * recall 召回率，分类正确的正样本个数占真正的正样本个数的比例，也叫做敏感度，即这个分类器识别正样本的敏感能力是多少。

![](https://images.gitbook.cn/aebd2ab0-742b-11ea-976b-69ddffd875eb)

[图片来源](https://en.wikipedia.org/wiki/Precision_and_recall)

  * f1-score 是二者的调和平均值，如果我们想创建一个具有最佳的精度—召回率平衡的模型，那么就要尝试将 F1 score 最大化。

![](https://images.gitbook.cn/cd0eb3d0-742b-11ea-bf4b-4f99c7eed580)

  * support 指每一类的实际样本个数。

最后一行分别是 precision、recall、f1-score 的加权平均，权重就是 support，例如 precision 的结果为：

    
    
    (0.73*1660 + 0.90*2004 + 0.96*1589 + 0.83*3706 + 0.71*1914 + 0.00*46 + 0.00*24 + 0.00*15) / 10958 = 0.83
    

此外还需要辅助函数 pred2label 将句子中每个单词的预测概率最大的值转化为相应的标签。

    
    
    test_pred = model.predict(X_te, verbose=1)
    
    idx2tag = {i: w for w, i in tag2idx.items()}
    
    def pred2label(pred):
        out = []
        for pred_i in pred:
            out_i = []
            for p in pred_i:
                p_i = np.argmax(p)
                out_i.append(idx2tag[p_i].replace("PAD", "O"))
            out.append(out_i)
        return out
    
    pred_labels = pred2label(test_pred)
    test_labels = pred2label(y_te)
    
    print(classification_report(test_labels, pred_labels))
    
    
    
               precision    recall  f1-score   support
    
          per       0.73      0.77      0.75      1660
          tim       0.90      0.84      0.87      2004
          gpe       0.96      0.91      0.94      1589
          geo       0.83      0.88      0.86      3706
          org       0.71      0.64      0.67      1914
          art       0.00      0.00      0.00        46
          eve       0.00      0.00      0.00        24
          nat       0.00      0.00      0.00        15
    
    micro avg       0.83      0.81      0.82     10958
    macro avg       0.82      0.81      0.82     10958
    

从上面结果可知模型对 gpe 这类的预测效果比较好，对 org 和 per 的预测要差一些。

我们还可以打印出某个句子来看预测结果和实际标签的比较：

    
    
    i = 1927
    p = model.predict(np.array([X_te[i]]))
    p = np.argmax(p, axis=-1)
    true = np.argmax(y_te[i], -1)
    print("{:15}||{:5}||{}".format("Word", "True", "Pred"))
    print(30 * "=")
    for w, t, pred in zip(X_te[i], true, p[0]):
        if w != 0:
            print("{:15}: {:5} {}".format(words[w-1], tags[t], tags[pred]))
    

![](https://images.gitbook.cn/0a31cb80-742c-11ea-9ada-bd17629b3862)

* * *

参考文献：

  * createmomo, [_CRF Layer on the Top of BiLSTM_](https://createmomo.github.io/2017/09/12/CRF_Layer_on_the_Top_of_BiLSTM_1/)
  * Guillaume Lample et al, [_Neural Architectures for Named Entity Recognition_](https://arxiv.org/abs/1603.01360)
  * Tobias Sterbak, [Introduction To Named Entity Recognition In Python](https://www.depends-on-the-definition.com/introduction-named-entity-recognition-python/)
  * Erdenebileg Batbaatar, Keun Ho Ryu, [Ontology-Based Healthcare Named Entity Recognition from Twitter Messages Using a Recurrent Neural Network Approach](https://www.researchgate.net/publication/336113245_Ontology-Based_Healthcare_Named_Entity_Recognition_from_Twitter_Messages_Using_a_Recurrent_Neural_Network_Approach)

