### Skip-Gram

前一篇，我们学习了什么是 CBOW，今天来看 **Skip-Gram** ，它是 word2vec 的另一种训练思路。

Skip-Gram 和 CBOW 的思路是相反的，CBOW 是由上下文得到中心词，而 **Skip-Gram 是由中心词预测上下文** 。

![](https://images.gitbook.cn/375ba960-5329-11ea-87ff-fd6761ac434d)

所以 Skip-Gram 的模型输入是一个中心词的词向量，输出是中心词的上下文向量。不过它并不是对 CBOW
模型的简单的颠倒，而是用一个中心词来预测窗口内除它以外的每个词，虽然从上面两个的对比图看来，Skip-Gram
的输入是一个词，输出是多个词，但其实在代码中构造训练数据时，输出也是一个词：

![](https://images.gitbook.cn/4bdef950-5329-11ea-b315-dd4a0f63c225)

它的学习过程就像我们在讲 CBOW 的前向传播时输入是一个单词的那个流程一样，接下来我们看看 Skip-Gram
的前向计算和反向传播是怎样的，大家在看这部分推导时可以对比 CBOW 的内容。

#### **前向计算**

![](https://images.gitbook.cn/66bc4340-5329-11ea-b707-2f81b607a86b)

和 CBOW 中一样，模型两个核心矩阵 W 和 W‘，维度如下：

W 是 embedding 矩阵，维度是：

    
    
    vocab_size * embedding_size
    

记作：

$$W=\\{w_{ki}\\}$$

W' 是 context 矩阵，维度是：

    
    
    embedding_size * vocab_size
    

记作：

$$W'=\\{w'_{ij}\\}$$

输入只有一个单词，同样用 x 表示，也是 One-hot 向量，大小是 `vocab_size * 1`。

然后按同样的方式得到 h，即在 embedding 矩阵中找到自己的位置，就是 W 的其中一行，将 x 和嵌入矩阵相乘就可以得到 $h = W^T x =
W^T_{(k,.)} := V^T_{w_I}$（式 1），维度也是 embedding_size * 1。$V_{w_I}$ 仍然表示输入单词
$w_I$ 的词向量。h 的第 i 个元素 $h_i = \sum_{k=1}^V x_k w_{ki} $（式 2）。

下一步要生成 u，CBOW 中是生成一个结果，而在 Skip-Gram 中是最终要生成 C 个结果，所以用 $u_{c} $ 来表示。每个 $u_{c} $
的生成方式是一样的，$h$ 与 context 矩阵 W' 相乘一次，得到一个维度为 vocab_size * 1 的向量，因为每个向量所用第 W'
是一样的，所以可以统一表示为 $u_{c} = W'^T h$（式 3）。同样，$u_{c}$ 的第 j 个元素 $u_{c,j} =
W'^T_{(.,j)} h := V'^T_{w_j} h，for c = 1, 2, · · · , C$（式 4）。其中我们用 $V'_{w_j}$
来表示预测单词 $w_j$ 的词向量，其中 j 取遍整个词汇表 1~V，同时它也是 W' 的其中一列。

在 CBOW 中我们用 $y_j = P(w_j | w_I) $ 表示输出的概率，这里也同样用 Softmax 求概率：

$$y_{c,j} = P(w_{c,j} = w_{O,c} | w_I) = softmax(u_{c,j}) =
\frac{e^{u_{c,j}}}{\sum_{j'=1}^V e^{u_{j'}}}（式 5）$$

其中 $w_{c,j}$ 是指要预测的上下文窗口中第 c 个结果向量中第 j 个位置对应的单词， $w_{O,c}$ 指上下文窗口中实际的第 c 个单词。

和 CBOW 同样的道理，我们可以得到损失函数为（式 6）：

$$\begin{align}E & = - log p ( w_{O,1}, w_{O,2},..., w_{O,C} | w_I ) \\\ & =
-log \prod^C_{c=1} \frac{e^{u_{c,j^*_c}}}{\sum_{j'=1}^V e^{u_{j'}}} \\\ & = -
\sum^C_{c=1} u_{c,j^*_c} + C \cdot log \sum_{j'=1}^V e^{u_{j'}}
\\\\\end{align}$$

#### **反向传播**

Skip-Gram 的反向传播同样和 CBOW 的有很多相似之处。

首先求

$$ \frac{∂ E}{∂ V'_{w_j}} = \sum^C_{c=1} \frac{∂ E}{∂ u_{c,j}} \frac{∂
u_{c,j}}{∂ V'_{w_j}} $$

其中 $\frac{∂ E}{∂ u_{c,j}} = e_{c,j}$，详细推导请看下图：

![](https://images.gitbook.cn/977783c0-532c-11ea-8090-cb5bc330cfd0)

再由（式 4）得到 $\frac{∂ u_{c,j}}{∂ V'_{w_j}} = h$。

我们用 $EI_j = \sum^C_{c=1} e_{c,j}$ 表示窗口中 C 个 $e_{c,j}$ 的和，最后

$$ \frac{∂ E}{∂ V'_{w_j}} = \sum^C_{c=1} \frac{∂ E}{∂ u_{c,j}} \frac{∂
u_{c,j}}{∂ V'_{w_j}} = \sum^C_{c=1} e_{c,j} \cdot h = EI_j \cdot h,
j=1,...,V$$

这样就可以进行梯度更新：

$$V'^{(new)}_{w_j} = V'^{(old)}_{w_j} -\eta \cdot EI_j \cdot h, (j=1,...,V) （式
7）$$

接下来求 $ \frac{∂ E}{∂ w_{ki}}$，这部分和 CBOW 的几乎一样，就是在 $e_j$ 那里变成了 $EI_j$。

首先

$$ \frac{∂ E}{∂ w_{ki}} = \frac{∂ E}{∂ h_i} \frac{∂ h_i}{∂ w_{ki}}$$

其中

$$\begin{align}\frac{∂ E}{∂ h_i} & = \sum^V_{j=1} \sum^C_{c=1} \frac{∂ E}{∂
u_{c,j}} \frac{∂ u_{c,j}}{∂ h_i} \\\ & =\sum^V_{j=1} \sum^C_{c=1} e_{c,j}
w'_{i,j} \\\ & = \sum^V_{j=1} EI_j w'_{i,j} \\\ & := EH_i \\\\\end{align}$$

所以有

$$ \frac{∂ E}{∂ w_{ki}} = \frac{∂ E}{∂ h_i} \frac{∂ h_i}{∂ w_{ki}} = EH_i
\cdot x_k $$

进而

$$ \frac{∂ E}{∂ W} = x \cdot EH^T $$

又因为 x 是 One-Hot 向量，只有其中一个位置是 1，其余是 0， 所以只有输入单词对应的位置上有数值，于是：

$$ \frac{∂ E}{∂ V_{w_I}} = \frac{∂ E}{∂ W_{(k,.)}} = EH^T $$

然后进行梯度更新：

$$V^{(new)}_{w_I} = V^{(old)}_{w_I} -\eta \cdot EH^T （式 8）$$

这部分和 CBOW 中的主要区别就在于

$$EH_i = \sum^V_{j=1} EI_j \cdot w'_{ij}$$

在前一篇文章中也比较详细地介绍了 Huffman 树和负采样的思想，以及如何将它们用于 CBOW，对于 Skip-Gram 也同样介绍这两个方法。

### 基于层级 Softmax 的 Skip-Gram

我们在看这一部分时可以和上一篇的 CBOW 对比来看，此时模型的输入是训练样本的每个词，模型的输出是在每个目标词前后各取 c 个词。

![](https://images.gitbook.cn/89215c00-532d-11ea-87ff-fd6761ac434d)

**1\. 首先，因为模型的输入只是单个词向量** ，输入目标中心词就是一个 $x_w$，所以不需要取平均了。

**2\. 然后，将词汇表建立成一棵 Huffman 树。**

让我们先来回顾一下各符合的含义：

  * $x_w$：输入的中心词向量
  * $x_i$：中心词的上下文词向量
  * $l_{x_i}$：从根结点到词 $x_i$ 的路径上包含的结点个数
  * $\theta_{j}^{x_i}$：从根结点到词 $x_i$ 的路径上，每个非叶子结点逻辑回归的参数
  * $d_{j}^{x_i}$：是每个结点的编码，即 右边为 0，左边为 1

在这里我们要求的是给定中心词 $x_w$ 后，它的上下文的概率，即 $P(x_i|x_w), i=1,2...2c$，它等于 $x_w$ 下每个上下文单词
$x_i$ 的概率的乘积，而其中的每一项 $P(x_i|x_w)$ 和 CBOW 中的推导形式一样的，只需要将 CBOW 中 $d$ 和 $\theta$
处的 $x_w$ 换成 $x_i$：

$$P(x_i|x_w) = \prod_{j=2}^{l_{x_i}}P(d_j^{x_i}|x_w, \theta_{j-1}^{x_i}) =
\prod_{j=2}^{l_{x_i}} [\sigma(x_w^T\theta_{j-1}^{x_i})]
^{1-d_j^{x_i}}[1-\sigma(x_w^T\theta_{j-1}^{x_i})]^{d_j^{x_i}}$$

**3\. 然后转换成对数似然函数形式：**

$$L = \sum\limits_{x_w} log \prod_{x_i \in
Context(x_w)}\prod_{j=2}^{l_{x_i}}P(d_j^{x_i}|x_w, \theta_{j-1}^{x_i}) =
\sum\limits_{x_w} \sum\limits_{x_i \in Context(x_w)}
\sum\limits_{j=2}^{l_{x_i}} ((1-d_j^{x_i}) log
[\sigma(x_w^T\theta_{j-1}^{x_i})] + d_j^{x_i}
log[1-\sigma(x_w^T\theta_{j-1}^{x_i})])$$

**4\. 接着要用梯度上升法对其进行优化，就需要对目标函数里面的参数进行求梯度。**

先将最右边括号里的一串式子记为：

$$\zeta = ((1-d_j^{x_i}) log [\sigma(x_w^T\theta_{j-1}^{x_i})] + d_j^{x_i}
log[1-\sigma(x_w^T\theta_{j-1}^{x_i})])$$

然后对 $\theta$ 进行求导：

$$\begin{align} \frac{\partial \zeta}{\partial \theta_{j-1}^{x_i}} & =
(1-d_j^{x_i})\frac{(\sigma(x_w^T\theta_{j-1}^{x_i})(1-\sigma(x_w^T\theta_{j-1}^{x_i})}{\sigma(x_w^T\theta_{j-1}^{x_i})}x_w
- d_j^{x_i}
\frac{(\sigma(x_w^T\theta_{j-1}^{x_i})(1-\sigma(x_w^T\theta_{j-1}^{x_i})}{1-
\sigma(x_w^T\theta_{j-1}^{x_i})}x_w \\\ & =
(1-d_j^{x_i})(1-\sigma(x_w^T\theta_{j-1}^{x_i}))x_w -
d_j^{x_i}\sigma(x_w^T\theta_{j-1}^{x_i})x_w \\\& =
(1-d_j^{x_i}-\sigma(x_w^T\theta_{j-1}^{x_i}))x_w \end{align}$$

接着对 $x_w$ 求偏导，这里用到 $x_w$ 和 $theta_{j-1}^{x_i}$ 的对称性，可以得到：

$$\frac{\partial \zeta}{\partial x_w} =
\sum\limits_{j=2}^{l_{x_i}}(1-d_j^{x_i}-\sigma(x_w^T\theta_{j-1}^{x_i}))\theta_{j-1}^{x_i}$$

**5\. 求出梯度形式后，我们就可以用梯度上升法来进行梯度更新：**

$$\theta_{j-1}^{x_i} = \theta_{j-1}^{x_i} + \eta
(1-d_j^{x_i}-\sigma(x_w^T\theta_{j-1}^{x_i}))x_w$$

$$x_w= x_w +\eta
\sum\limits_{j=2}^{l_{x_i}}(1-d_j^{x_i}-\sigma(x_w^T\theta_{j-1}^{x_i}))\theta_{j-1}^{x_i}
\;(i =1,2..,2c)$$

其中二者相同的部分，令：

$$f = \sigma(x_w^T\theta_{j-1}^{x_i})$$

$$g = \eta (1-d_j^{x_i}-f)$$

于是梯度更新的公式变为：

$$x_w = x_w + g \theta_{j-1}^{x_i}$$

$$\theta_{j-1}^{x_i}= \theta_{j-1}^{x_i} + g x_w$$

**所以它的算法步骤为：**

  1. 输入语料样本集，每个样本为 $(x_w, context(x_w))$，基于训练样本建立 Huffman 树
  2. 初始化所有参数 $\theta$，所有的词向量 $x_w$
  3. 进行梯度更新：

$$\begin{align} & for \quad x_i \quad in \quad
(w_{t-c}...,w_{t-1},w_{t+1}...,w_{t+c}) \\\& \quad\quad x_w = 0 \\\&
\quad\quad for \quad j =2 \quad to \quad l_{x_i} \\\& \quad\quad\quad\quad f =
\sigma(x_w^T\theta_{j-1}^{x_i}) \\\& \quad\quad\quad\quad g = \eta
(1-d_j^{x_i}-f) \\\& \quad\quad\quad\quad x_w = x_w + g \theta_{j-1}^{x_i}
\\\& \quad\quad\quad\quad \theta_{j-1}^{x_i}= \theta_{j-1}^{x_i} + g x_w \\\&
\quad\quad x_i = x_i + x_w\end{align}$$

这里需要注意更新的是 $x_i$，即上下文的词，而不是目标中心词 $x_w$，即实际上用的是 $P(x_w|x_i), i=1,2...2c$。

一方面因为上下文是相互的，$x_i$ 是 $x_w$ 的上下文，当窗口移动到 $x_i$ 时， $x_w$ 也成为 $x_i$
的上下文。另一方面因为源码使用了随机梯度下降算法。

如果是完全梯度，更新 $p(w|context(w)) = p(context(w)|w)$
是一样的效果，因为它们的更新状态是一样的，要么都没更新，要么都更新了。

但在随机梯度下降中，如果按照 $p(context(w)|w)$ 更新，那么 中心词 $x_w$ 后面的词是还没有更新的，例如 p(3|2) 时，3
就没更新，这样显然不对，所以就按照 $p(w|context(w))$ 来更新上下文的词，这样可以保证每个窗口词都能更新一遍，可以让迭代更加均衡。

### 基于负采样的 Skip-Gram

下面我们继续看基于负采样的 Skip-Gram，这里同样可以和 CBOW 对比来看，在 CBOW 中我们是想要最大化这个式子：

$$\prod_{i=0}^{neg}P(context(w_0), w_i) =
\sigma(x_{w_0}^T\theta^{w_0})\prod_{i=1}^{neg}(1-
\sigma(x_{w_0}^T\theta^{w_i}))$$

在 Skip-Gram 中，它的目标函数是语料库 C 中每个单词 w 的 g(w) 累积：

$$G = \prod_{w \in C} g(w)$$

因为 Skip-Gram 是由一个中心词预测上下文，所以 g(w) 是 上下文每个单词 $x_i$ 的 $g(x_i)$ 累积：

$$G = \prod_{w \in C} g(w) = \prod_{w \in C} \prod_{x_i \in Context(w)} g(x_i)
$$

其中 $g(x_i)$ 和上面 CBOW 中的 g(w) 是一样的形式：

$$g(x_i) = \prod_{z \in {x_i} \bigcup negative(x_i)} p(z | w) =
\sigma(x_{w}^T\theta^{x_i}) \prod_{z}^{}(1- \sigma(x_{w}^T\theta^{z}))$$

当 z 为正例时，记 $y_z = 1$，z 为负例时，$y_z = 0$，则 $g(x_i)$ 又可以写成：

$$ g(x_i) = \prod_{z}^{} \sigma(x_{w}^T\theta^{x_i})^{y_z} (1-
\sigma(x_{w}^T\theta^{x_i}))^{1-y_z} $$

再转换为对数似然函数形式：

$$\begin{align} L = log G &= log \prod_{w \in C} \prod_{x_i \in Context(w)}
g(x_i) \\\&= \sum_{w \in C} \sum_{x_i \in Context(w)} \log{g(x_i)} \\\&=
\sum_{w \in C} \sum_{x_i \in Context(w)} \log \prod_{z \in {x_i} \bigcup
negative(x_i)} p(z | w) \\\&= \sum_{w \in C} \sum_{x_i \in Context(w)} \sum_{z
\in {x_i} \bigcup negative(x_i)} \log p(z | w) \\\&= \sum_{w \in C} \sum_{x_i
\in Context(w)} \sum_{z \in {x_i} \bigcup negative(x_i)} \log {
\sigma(x_{w}^T\theta^{z})^{y_z} (1- \sigma(x_{w}^T\theta^{z}))^{1-y_z} } \\\&=
\sum_{w \in C} \sum_{x_i \in Context(w)} \sum_{z \in {x_i} \bigcup
negative(x_i)} \\{ {y_z \log \sigma(x_{w}^T\theta^{z})} + {(1 -y_z) \log (1-
\sigma(x_{w}^T\theta^{z})} ) \\}\end{align}$$

先将右边的一串式子记为:

$$\zeta = \\{ {y_z \log \sigma(x_{w}^T\theta^{z})} + {(1 -y_z) \log (1-
\sigma(x_{w}^T\theta^{z})} ) \\}$$

然后对其求 $\theta$ 和 $x_w$ 的偏导：

$$\begin{align} \frac{\partial \zeta }{\partial \theta^{z} } &= y_z (1-
\sigma(x_{w}^T\theta^{z}))x_{w}-(1-y_z)\sigma(x_{w}^T\theta^{z})x_{w} \\\ & =
(y_z -\sigma(x_{w}^T\theta^{z})) x_{w} \end{align}$$

利用 $\theta$ 和 $x_w$ 的对称性，得到：

$$\begin{align} \frac{\partial \zeta }{\partial x_{w} } = (y_z
-\sigma(x_{w}^T\theta^{z}))\theta^{z}\end{align}$$

再用梯度上升法进行梯度更新，迭代到最优：

$$\theta^{z} = \theta^{z} + \eta (y_z -\sigma(x_{w}^T\theta^{z})) x_{w} $$

$$x_w = x_w + \eta \sum_{z \in {x_i} \bigcup negative(x_i)} (y_z
-\sigma(x_{w}^T\theta^{z}))\theta^{z}$$

令：

$$f = \sigma(x_w^T\theta^{z})$$

$$g = \eta (y_z-f)$$

于是梯度更新的公式变为：

$$x_w = x_w + g \theta^{z}$$

$$\theta^{z}= \theta^{z} + g x_w$$

所以它的算法步骤为：

  1. 输入语料样本集，每个样本为 $(x_w, context(x_w))$，负采样的个数 neg
  2. 初始化所有参数 $\theta$ ，所有的词向量 $x_w$
  3. 进行梯度更新：

$$\begin{align} & for \quad x_i \quad in \quad
(w_{t-c}...,w_{t-1},w_{t+1}...,w_{t+c}) \\\& \quad\quad x_w = 0 \\\&
\quad\quad for \quad z = w_0 \quad to \quad w_{neg} \\\& \quad\quad\quad\quad
f = \sigma(x_w^T\theta^{z}) \\\& \quad\quad\quad\quad g = \eta (y_z-f) \\\&
\quad\quad\quad\quad x_w = x_w + g \theta^{z} \\\& \quad\quad\quad\quad
\theta^{z}= \theta^{z} + g x_w \\\& \quad\quad x_i = x_i + x_w\end{align}$$

算法流程和基于层级 Softmax 的一样，也是要更新上下文的词而不是中心词。

### 应用 Skip-Gram

接下来我们看如何实现 Skip-Gram 算法，整体流程和 CBOW 是相同的。

前面四步和 CBOW 是一样的，导入依赖的库，下载数据，对数据进行预处理，建立字典。

**1\. 导入所需要的库**

    
    
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
    

**2\. 下载 Wikipedia 数据**

    
    
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

**3\. 读取数据，并用 NLTK 进行预处理**

读取第一个文本文件，转化为一个单词列表，经过预处理后，有 3360286 个单词，我们可以打印看一下 words 里面的数据。

    
    
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
    
    words = read_data(filename)
    print('Data size %d' % len(words))
    print('Example words (start): ',words[:10])
    print('Example words (end): ',words[-10:])
    
    
    
    Reading data...
    Data size 3360286
    Example words (start):  ['propaganda', 'is', 'a', 'concerted', 'set', 'of', 'messages', 'aimed', 'at', 'influencing']
    Example words (end):  ['favorable', 'long-term', 'outcomes', 'for', 'around', 'half', 'of', 'those', 'diagnosed', 'with']
    

**4\. 建立字典**

假设我们输入的文本是：

    
    
    "I like to go to school"
    

  * dictionary：将单词映射为 ID，例如 {I:0, like:1, to:2, go:3, school:4}
  * reverse_dictionary：将 ID 映射为单词，例如 {0:I, 1:like, 2:to, 3:go, 4:school}
  * count：计数每个单词出现了几次，例如 [(I,1),(like,1),(to,2),(go,1),(school,1)]
  * data：将读取的文本数据转化为 ID 后的数据列表，例如 [0, 1, 2, 3, 2, 4]

此外，我们用 UNK 代表比较少见的单词：

    
    
    # 将词汇表大小限制在 50000
    vocabulary_size = 50000                                                             
    
    def build_dataset(words):
      count = [['UNK', -1]]
      # 选择 50000 个最常见的单词，其他的用 UNK 代替
      count.extend(collections.Counter(words).most_common(vocabulary_size - 1))            
      dictionary = dict()
    
      # 为每个 word 建立一个 ID，保存到一个字典里
      for word, _ in count:                                                                
        dictionary[word] = len(dictionary)
    
      data = list()
      unk_count = 0
    
      # 将文本数据中所有单词都换成相应的 ID，得到 data
      # 不在字典里的单词，用 UNK 代替
      for word in words:                                                                
        if word in dictionary:                                                            
          index = dictionary[word]
        else:
          index = 0  # dictionary['UNK']
          unk_count = unk_count + 1
        data.append(index)
    
      count[0][1] = unk_count
    
      reverse_dictionary = dict(zip(dictionary.values(), dictionary.keys())) 
    
      # 确保字典和词汇表的大小一样
      assert len(dictionary) == vocabulary_size                                            
    
      return data, count, dictionary, reverse_dictionary
    
    data, count, dictionary, reverse_dictionary = build_dataset(words)
    print('Most common words (+UNK)', count[:5])
    print('Sample data', data[:10])
    del words  
    

**5\. 生成 Skip-Gram 的批次数据**

在这一步生成批次数据时，Skip-Gram 要生成的是和 CBOW 相反的 data-label 对。例如我们有一句：

    
    
    ['propaganda', 'is', 'a', 'concerted', 'set', 'of', 'messages', 'aimed']
    

当 `window_size = 2` 时，CBOW 生成的是：

    
    
    batch: [['propaganda', 'is', 'concerted', 'set'], ['is', 'a', 'set', 'of'], ['a', 'concerted', 'of', 'messages'], ['concerted', 'set', 'messages', 'aimed'], ['set', 'of', 'aimed', 'at'], ['of', 'messages', 'at', 'influencing'], ['messages', 'aimed', 'influencing', 'the'], ['aimed', 'at', 'the', 'opinions']]
    labels: ['a', 'concerted', 'set', 'of', 'messages', 'aimed', 'at', 'influencing']
    

而 Skip-Gram 生成的是：

    
    
    batch: ['a', 'a', 'a', 'a', 'concerted', 'concerted', 'concerted', 'concerted']
    labels: ['propaganda', 'is', 'concerted', 'set', 'is', 'a', 'set', 'of']
    
    
    
    data_index = 0
    
    def generate_batch_skip_gram(batch_size, window_size):
    
      # 用全局变量 data_index 记录当前取到哪里，每次读取一个数据集后增加 1，如果超出结尾则又从头开始
      global data_index                                                         
    
      # 在 Skip-Gram 中 batch 为中心词数据集
      # labels 为上下文数据集
      batch = np.ndarray(shape=(batch_size), dtype=np.int32)                    
      labels = np.ndarray(shape=(batch_size, 1), dtype=np.int32)                
    
      # span 是指中心词及其两侧窗口 [ skip_window target skip_window ] 的总长度
      span = 2 * window_size + 1                                                 
    
      # buffer 用来存储 span 内部的数据
      buffer = collections.deque(maxlen=span)                                    
    
      # 填充 buffer 更新 data_index
      for _ in range(span):                                                        
        buffer.append(data[data_index])
        data_index = (data_index + 1) % len(data)
    
      # 对每个中心词要取的上下文词的个数
      num_samples = 2*window_size                                                 
    
      # 内循环用 span 里面的数据先来得到 num_samples 个数据点
      # 外循环重复 batch_size//num_samples 次来得到整个批次数据
      for i in range(batch_size // num_samples):
        k=0
        # 得到 batch ，label，注意 label 里面没有 target 单词
        for j in list(range(window_size))+list(range(window_size+1,2*window_size+1)):
          batch[i * num_samples + k] = buffer[window_size]
          labels[i * num_samples + k, 0] = buffer[j]
          k += 1 
    
        # 每次读取 num_samples 个数据后，span 向右移动一位得到新的 span
        buffer.append(data[data_index])
        data_index = (data_index + 1) % len(data)
      return batch, labels
    
    print('data:', [reverse_dictionary[di] for di in data[:8]])
    
    for window_size in [1, 2]:
        data_index = 0
        batch, labels = generate_batch_skip_gram(batch_size=8, window_size=window_size)
        print('\nwith window_size = %d:' %window_size)
        print('    batch:', [reverse_dictionary[bi] for bi in batch])
        print('    labels:', [reverse_dictionary[li] for li in labels.reshape(8)])
    

**6\. 设置超参数**

这一步的设置和 CBOW 是一样的：

    
    
    batch_size = 128                             
    embedding_size = 128             # embedding 向量的维度
    
    window_size = 2                 # 中心词的左右两边各取 2 个
    
    valid_size = 16                 # 随机选择一个 validation 集来评估单词的相似性
    
    valid_window = 50                # 从一个大窗口随机采样 valid 数据
    
    # 选择 valid 样本时, 取一些频率比较大的单词，也选择一些适度罕见的单词
    valid_examples = np.array(random.sample(range(valid_window), valid_size))            
    valid_examples = np.append(valid_examples,random.sample(range(1000, 1000+valid_window), valid_size),axis=0)
    
    num_sampled = 32                 # 负采样的样本个数   
    

**7\. 定义输入输出**

这一步中 data 和 label 和 CBOW 是相反的：

    
    
    tf.reset_default_graph()
    
    # 用于训练输入的中心词 ID
    train_dataset = tf.placeholder(tf.int32, shape=[batch_size])
    # 用于训练的标签数据，即上下文词的 ID
    train_labels = tf.placeholder(tf.int32, shape=[batch_size, 1])
    # Validation 是用来验证的小数据集，是 constant 不需要用 placeholder，直接赋值前面已经定义了 valid_examples
    valid_dataset = tf.constant(valid_examples, dtype=tf.int32)
    

**8\. 定义模型的参数**

Embedding 变量用来存储词嵌入向量，初始化为 -1 到 1 之间的随机数，随着优化过程不断调整，在对 loss
函数进行优化时，会将其所有源头变量进行调整优化：

    
    
    embeddings = tf.Variable(tf.random_uniform([vocabulary_size, embedding_size], -1.0, 1.0))
    
    # softmax_weights 和 softmax_biases 是用来做分类的参数
    softmax_weights = tf.Variable(
        tf.truncated_normal([vocabulary_size, embedding_size],
                            stddev=0.5 / math.sqrt(embedding_size))
    )
    softmax_biases = tf.Variable(tf.random_uniform([vocabulary_size],0.0,0.01))
    

**9\. 定义损失函数**

这里仍然是用负采样方法，在 TensorFlow 中是用 tf.nn.sampled_softmax_loss 来实现，可以比较高效快速地优化参数。
当不用负采样时，假设词库有 10000 个单词，每个词向量的维度是 300，那么网络中每一隐藏层的参数是 3000000 个，输出层有 10000
个类，计算量很大。

使用负采样后，假设选择 5 个负样本，那么输出层的向量就只有 6 维，参数也从 300 万减少到 1800 个。

区别是，在 CBOW 中的输入 Embedding 是上下文的平均嵌入，在 Skip-Gram 中是直接用输入中心词的嵌入。

下面代码中 embedding_lookup 用来很方便地得到 Embeddings 中 id 为 train_dataset
的这些行，它和一般的查找最大的区别是，它可以接收一个 tensor 列表作为查找对象，并且输入的 ids 还需要服从一种分段策略。

Embeddings 的每一行代表了每个单词的词嵌入向量，在定义了损失函数和优化算法后，就可以进行训练学习。

    
    
    # 输入是训练数据的 embeddings
    embed = tf.nn.embedding_lookup(embeddings, train_dataset)
    
    # 每次用一些负采样样本计算 softmax loss, 
    # 用这个 loss 来优化 weights, biases, embeddings
    loss = tf.reduce_mean(
        tf.nn.sampled_softmax_loss(
            weights=softmax_weights, biases=softmax_biases, inputs=embed,
            labels=train_labels, num_sampled=num_sampled, num_classes=vocabulary_size)
    )
    

**10\. 定义优化算法**

优化算法用 Adagrad：

    
    
    optimizer = tf.train.AdagradOptimizer(1.0).minimize(loss)
    

**11\. 定义计算单词相似性的函数**

这一步也是没有变化的，用余弦距离来计算验证集中单词的相似度， norm 定义为二范数，再将词向量进行正则化得到
normalized_embeddings，这样变成了只考虑方向的单位向量，再从单位词向量矩阵中找出验证集词语所对应的向量，单词的相似性用余弦距离来表示，值越大代表越相似。

    
    
    norm = tf.sqrt(tf.reduce_sum(tf.square(embeddings), 1, keepdims=True))
    normalized_embeddings = embeddings / norm
    valid_embeddings = tf.nn.embedding_lookup(normalized_embeddings, valid_dataset)
    similarity = tf.matmul(valid_embeddings, tf.transpose(normalized_embeddings))
    

**12\. 训练 Skip-Gram**

这一步的过程也是和 CBOW 一样的，在每一次迭代中，用前面定义的 generate_batch_skip_gram 生成当前 batch
训练用的数据和标签，用 Adagrad 优化算法使损失达到最小，每 2000 步输出一下平均损失， 每 10000
次时从验证集选几个中心词，输出离每个词最近的 8 个单词。

    
    
    num_steps = 100001
    skip_losses = []
    
    with tf.Session(config=tf.ConfigProto(allow_soft_placement=True)) as session:
      # 初始化图中的变量
      tf.global_variables_initializer().run()
      print('Initialized')
      average_loss = 0
    
      # 训练 Word2vec 模型 num_step 次
      for step in range(num_steps):
    
        # 生成一批数据
        batch_data, batch_labels = generate_batch_skip_gram(
          batch_size, window_size)
    
        # 运行优化算法，计算损失
        feed_dict = {train_dataset : batch_data, train_labels : batch_labels}
        _, l = session.run([optimizer, loss], feed_dict=feed_dict)
    
        # 更新平均损失变量
        average_loss += l
    
        if (step+1) % 2000 == 0:
          if step > 0:
              # 计算一下平均损失，估计一下最近 2000 批数据的损失大小
            average_loss = average_loss / 2000
    
          skip_losses.append(average_loss)
          print('Average loss at step %d: %f' % (step+1, average_loss))
          average_loss = 0
    
        # 评估 validation 集的单词相似性
        if (step+1) % 10000 == 0:
          sim = similarity.eval()
          # 对验证集的每个用于验证的中心词，计算其 top_k 个最近单词的余弦距离，
          for i in range(valid_size):
            valid_word = reverse_dictionary[valid_examples[i]]
            top_k = 8 # number of nearest neighbors
            nearest = (-sim[i, :]).argsort()[1:top_k+1]
            log = 'Nearest to %s:' % valid_word
            for k in range(top_k):
              close_word = reverse_dictionary[nearest[k]]
              log = '%s %s,' % (log, close_word)
            print(log)
    
      skip_gram_final_embeddings = normalized_embeddings.eval()
    
    np.save('skip_embeddings',skip_gram_final_embeddings)
    
    with open('skip_losses.csv', 'wt') as f:
        writer = csv.writer(f, delimiter=',')
        writer.writerow(skip_losses)
    

训练结束后也可以用 CBOW 中的可视化代码进行可视化观察，只需要将 CBOW 代码中的这一行的 cbow_final_embeddings 换成
skip_gram_final_embeddings。

    
    
    selected_embeddings = skip_gram_final_embeddings[:num_points, :]
    

t-SNE 在可视化中会频繁用到，所以下面先来简单看一下它的原理。

### 什么是 t-SNE

t-SNE（t-distributed stochastic neighbor embedding）属于一种非线性降维算法，由 Laurens van
der Maaten 和 Geoffrey Hinton 在 2008 年提出。

它可以将高维数据压缩到 2 维或 3 维，但
**常用于可视化而不是直接用来降维，它的目的是保证高维空间中相似的数据，在被压缩到低维空间后也能够尽量挨得近** 。

用 t-SNE 将数据结果可视化后可以 **直观验证算法的有效性** ，看出结果一共聚成了几簇，哪些数据聚在一起，簇和簇之间的距离是怎么的。

**基本原理是** ，t-SNE 先将高维空间中数据之间的距离转换为高斯分布的概率，两点距离越近则概率值越大，再用一个长尾分布（t
分布）来表示低维空间的相似性.

**具体步骤如下：**

例如有高维空间的数据 $X = {x_1, ... , x_n}$，我们的目标是要得到低维空间的数据表示 $Y^T = {y_1, ... , y_n}$。

**1\. 首先计算条件概率：**

$$p_{j|i} = \frac{\exp{(-d(\boldsymbol{x}_i, \boldsymbol{x}_j) / (2
\sigma_i^2)})}{\sum_{i \neq k} \exp{(-d(\boldsymbol{x}_i, \boldsymbol{x}_k) /
(2 \sigma_i^2)})}, \quad p_{i|i} = 0$$

**2\. 再生成高维空间的联合分布：**

$$p_{ij} = \frac{p_{j|i} + p_{i|j}}{2N}$$

**3\. 然后要对 Y 进行随机初始化后开始迭代训练** ，

其中每次迭代时，先计算低维度下的 $q_{ij}$，

$$q_{ij} = \frac{(1 + ||\boldsymbol{y}_i -
\boldsymbol{y}_j)||^2)^{-1}}{\sum_{k \neq l} (1 + ||\boldsymbol{y}_k -
\boldsymbol{y}_l)||^2)^{-1}},$$

**4\. 然后计算两个分布之间的损失函数** ，称为 Kullback-Leibler divergence（KL 散度）:

$$ L = KL(P|Q) = \sum_{i \neq j} p_{ij} \log \frac{p_{ij}}{q_{ij}}$$

**5\. 接着计算梯度：**

$$ \frac{\delta L}{\delta y_i} = 4 \sum_j (p_{ij} - q_{ij})(y_i - y_j) $$

**6\. 得到梯度后，进行更新：**

$$Y^{t} = Y^{t-1} + \eta \frac{dC}{dY} + \alpha(t)(Y^{t-1} - Y^{t-2})$$

训练后，在得到低维空间的数据后，就可以方便地进行可视化了。

### 比较 CBOW 和 Skip-Gram

**首先，在模型输入上** ，Skip-Gram 每次只能看到一个单词的输入输出对，而 CBOW
可以看到上下文每个单词对，要比前者看到的东西多，因为它可以包含更多信息，这可以在前面代码中看出：

CBOW 生成的是：

    
    
    batch: [['propaganda', 'is', 'concerted', 'set'], ['is', 'a', 'set', 'of'], ['a', 'concerted', 'of', 'messages'], ['concerted', 'set', 'messages', 'aimed'], ['set', 'of', 'aimed', 'at'], ['of', 'messages', 'at', 'influencing'], ['messages', 'aimed', 'influencing', 'the'], ['aimed', 'at', 'the', 'opinions']]
    labels: ['a', 'concerted', 'set', 'of', 'messages', 'aimed', 'at', 'influencing']
    

而 Skip-Gram 生成的是：

    
    
    batch: ['a', 'a', 'a', 'a', 'concerted', 'concerted', 'concerted', 'concerted']
    labels: ['propaganda', 'is', 'concerted', 'set', 'is', 'a', 'set', 'of']
    

**其次，在模型表现上** ，在论文 _Distributed Representations of Words and Phrases and their
Compositionality, Mikolov and others, 2013_ 中表明 Skip-Gram 在语义任务中效果更好，而 CBOW
在语法任务中效果更好，但其实在大多数任务实践中，Skip-Gram 比 CBOW 表现要好一些。

一方面，对于同样的数据，Skip-Gram 的输入只是单个单词，而 CBOW 模型的输入大约是 Skip-Gram 模型输入的 2w 倍，w
是上下文窗口大小，这也让 **Skip-Gram 在效率上更有优势一些** ，尤其是在数十亿的大型数据集上。不过当数据量级小一些时，CBOW
的表现可能更好。

另一方面， **Skip-Gram 更能捕捉到词语的细小差距** ：

例如有下面两句话：

    
    
    It is a nice chatbot
    It is a wonderful chatbot
    

那么 CBOW 的输入数据为：

    
    
    [[It, is, nice, chatbot], a] [[It, is, wonderful, chatbot],a]
    

Skip-Gram 的输入数据为：

    
    
    [It, a], [is, a], [nice, a], [chatbot, a] [It, a], [is, a], [wonderful, a], [chatbot, a]
    

我们希望 word2vec 模型能够区分出 nice 和 wonderful 有一点不同，即希望模型能够识别出意义上有细微差别的词。

对于 CBOW 来说，因为在模型中会将这两个词的上下文词向量做一下平均，那么它很有可能会将这两个词看成一个。而对于 Skip-Gram
来说，这两个单词会与它们的上下文分开，模型就可以更多地关注到单词之间的微小差异。

在比较二者的表现时，我们可以通过观察损失的变化，也可以通过 t-SNE 的可视化来看词向量的聚集情况。

### 面试题

  1. 介绍 CBOW 和 Skip-Gram 的原理？
  2. CBOW 和 Skip-Gram 有什么区别？
  3. word2vec 是如何计算损失函数的？
  4. 如何应用负采样？

* * *

参考文献：

  * Xin Rong, [_word2vec Parameter Learning Explained_](https://arxiv.org/pdf/1411.2738.pdf)
  * Mikolov et al. [_Distributed Representations of Words and Phrases and their Compositionality_](https://papers.nips.cc/paper/5021-distributed-representations-of-words-and-phrases-and-their-compositionality.pdf)

