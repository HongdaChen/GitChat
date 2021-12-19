前面我们学习了 word2vec 的两种训练模型：CBOW 和 Skip-
Gram，这两种都是通过一个神经网络学习出单词的词向量，今天我们来学习另外一个学习词向量的模型 GloVe，全称叫 Global Vectors for
Word Representation，它是一个基于全局词频统计的词表征工具，属于非监督式学习方法。

word2vec 是深度学习中比较主流的词嵌入方法，后来在 2014 年由 Stanford NLP 团队的 Jeffrey
Pennington、Richard Socher、Chris Manning 在论文[ _GloVe: Global Vectors for Word
Representation_](https://nlp.stanford.edu/pubs/glove.pdf) 中提出了 GloVe。在论文中给出了
GloVe 和 word2vec 的对比，经过实验表明在单词类比任务中 GloVe 的效果要好于 CBOW 和 Skip-Gram，可以比 word2vec
更快地学习出更好的词向量。

![](https://images.gitbook.cn/c82ec5f0-596c-11ea-902a-115518f76783)

**GloVe 的基本思想是：**

  * 首先建立一个很大的单词-上下文的共现矩阵，矩阵的每个元素 $X_{ij}$ 代表每个单词 $x_i$ 在相应上下文 $x_j$ 的环境中共同出现的次数。
  * 然后用共现次数计算出共现比率，并建立词向量与共现比率之间的映射关系，进而得到 GloVe 的模型函数。
  * 接着建立加权平方差损失函数，再用 AdaGrad 优化算法学习出词向量 $w$、$\tilde{w}$。

接下来就详细学习一下 GloVe 的原理。

* * *

### GloVe 的模型函数是什么？

在前面的文章中，我们介绍了共现矩阵和 word2vec，先来回顾一下：

  * **共现矩阵** ，是在全局文本中进行计数得到的矩阵，它考虑到了单词之间的共现性，并且可以包含全局的信息，但是包含的语义信息有限，不能像 word2vec 那样能获得单词之间的相似性。
  * **word2vec** 学习到的词向量包含着丰富的语义，向量的不同维度可能包含着不同的信息，比如性别、时态、单复数等。但它的算法中需要定义一个窗口，要么用上下文预测中心词（CBOW），要么用中心词预测上下文（Skip-Gram），所以每次只是考虑窗口中的数据，这个只是局部的上下文，没有用到全局的统计信息，这样当我们频繁遇到“the robot”之类的文本时，就不能确定“the”到底是在全文中都会经常出现，还是只和“robot”有密切的关系。
  * **GloVe 就是想要结合这二者的长处** ，既能利用全局的统计信息，又能够捕捉到单词的含义。

#### **1\. 首先，要建立一个共现矩阵**

建立方法和前面提到共现矩阵时讲的一样，建立矩阵时也可以考虑局部上下文的信息，就是通过设置一个窗口长度，这个矩阵中的数值只是在这个窗口长度内同时出现的次数统计。

例如我们有三句话：

    
    
    NLP is not easy, NLP is interesting, NLP is amazing
    

首先定义一个窗口长度 c，表示在当前单词的左边取 c 个，右边取 c 个，这里取 c=2，则共现矩阵为：

![](https://images.gitbook.cn/6efca5a0-596d-11ea-902a-115518f76783)

而且不难发现这个矩阵是对称的。其中我们看 `(NLP,is)=4` 这一对是如何计算的，以 NLP 为中心的窗口一共有 3 个，在这 3 个窗口中 is
一共出现了 4 次，所以共现次数就为 4，同理可以计算其他的数值：

![](https://images.gitbook.cn/ab6742c0-596d-11ea-b7a0-3967fecd90d4)

有了单词之间的共现统计信息之后，就可以计算共现率，因为上下文中两个单词之间的共现率与它们的含义是紧密相关的，这样后面我们就可以用共现率找到带有语义信息的词向量。

#### **2\. 为什么上下文中两个单词之间的共现率与它们的含义有关？**

论文中举了这样一个例子：“冰”和“水蒸气”它们的状态虽然不同，但是都可以形成水，所以可以猜想在“冰”和“水蒸气”的上下文中，和“水”有关的例如“潮湿”等词出现的概率是差不多的，而“冰冷”“固体”这样的词会在“冰”的上下文中出现得比较多，在“水蒸气”的上下文中就不怎么出现。经过实际的统计，他们得到了下面这个表：

![https://nlp.stanford.edu/pubs/glove.pdf](https://images.gitbook.cn/00a61360-596e-11ea-8155-9d6d04886d5b)

取 k 为“固体”、“气”、“水”这几个词时，第一行为在“冰”的上下文中，k 的出现概率，第二行为在“水蒸气”的上下文中 k
的出现概率，第三行是两者的比率，这个表就可以用共现矩阵计算出来。

从表中可以看到“水”、“固体”确实在“冰”的上下文中出现的概率比其他不相关的词要大，“水”、“气”在“水蒸气”的上下文中出现的概率比其他的要大。

再看最后一行的比率，“水”与“冰”和“水蒸气”都相关，这个比值接近 1，“时尚”与“冰”和“水蒸汽”都不相关，这个比值也接近于 1。

即对于 $word_i$、$word_j$、$word_k$，如果 $word_i$、$word_k$ 很相关，$word_j$、$word_k$
不相关，那么 $\frac{P_{ik}}{P_{jk}}$ 比值会比较大，如果都相关，或者都不相关，那么这个比值应该趋于 1。由此可见
**上下文中两个单词之间的共现比率可以对含义相关的单词进行建模** ，因为我们可以通过这个比率看出单词 i 和 j 相对于 k 来讲哪个会更相关一些：

  * 如果都相关或者都不相关，则比率靠近 1；
  * 如果分子比分母更相关，则比率比 1 大很多；
  * 如果分母比分子更相关，则比率比 1 小很多。

#### **3\. 接下来如何用共现率计算词向量呢？**

在上面我们提到了可以用共现比率来对含义相关的单词进行建模，那么希望用一个函数 F 来将三个词向量 $word_i$、$word_j$、$word_k$
映射为这个比率，即：

$$\begin{equation}F(w_i, w_j, \tilde{w_k}) =
\frac{P_{ik}}{P_{jk}}\end{equation}$$

其中 $P_{ik}=P(k|i)=X_{ik}/X_i$ 为在 $word_i$ 的上下文中 $word_k$ 的出现概率，$X_i$ 为单词 i
的上下文中所有单词的个数，$X_{ik}$ 为 $word_i$ 的上下文中 $word_k$ 的出现次数。

这里 $w_i$、$w_j$ 为输入的词向量，$\widetilde{w_k}$ 为输出的词向量，相当于是 word2vec 中的上下文向量，因为在
GloVe 中上下文只有一个单词，所以 $\widetilde{w_k}$ 也是单个词向量。

**接下来我们就想要知道这个 F 的具体形式，** 希望它可以学习出带有含义的词向量，最好是运算越简单越好，例如简单的加减乘除。

于是想到 **用词向量的差来衡量单词之间的差异** ，将这个差距输入给 F，让 F 能捕捉到单词意义的不同，于是函数的输入就变成这个样子：

$$\begin{equation}F(w_i - w_j, \tilde{w_k}) =
\frac{P_{ik}}{P_{jk}}\end{equation}$$

**为了在 $w_i - w_j$ 和 $\tilde{w_k}$ 之间建立一个线性关系** ，于是将二者的点积作为函数的输入，函数关系变为：

$$\begin{equation}F((w_i - w_j)^T \tilde{w_k}) =
\frac{P_{ik}}{P_{jk}}\end{equation}$$

接下来 **想要 F 是同态的** ，即满足：

$$\begin{equation}F((w_i - w_j)^T \tilde{w_k}) = \frac{ F(w_i^T \tilde{w_k})
}{ F(w_j^T \tilde{w_k}) }\end{equation}$$

结合这两个式子可以得到：

$$\begin{equation}F(w_i^T \tilde{w_k}) = P_{ik} = \frac{X_{ik} }{ X_i
}\end{equation}$$

从共现矩阵那里也可以看出，$word_i$、$word_j$ 的地位是同等的，所以经过 F
的作用，要满足它们的的顺序不影响结果，最好可以将右边的除法变成加减法，那就想到可以两边取 log，这样除法变成了减法，则 **F 可以取为指数函数
exp** ，那么两边取对数，关系式变为：

$$\begin{equation}w_i^T \tilde{w_k} = log{P_{ik}} = log{X_{ik}} -
log{X_{i}}\end{equation}$$

然后将 $log X_i$ 移到左边，因为和 $word_k$ 没有关系，所以可以归入到偏置 $bias_i$ 中，同时为了保持公式的对称性，再加入
$word_k$ 的 $bias_k$，于是得到：

$$\begin{equation}w_i^T\tilde{w}_k + b_i + \tilde{b}_k =
logX_{ik}\end{equation}$$

**上式就是 GloVe 的核心等式** ，其中左边是词向量的运算，右边是共现矩阵的值。

### GloVe 的损失函数

有了上面的模型后，我们用平方差来衡量模型的损失：

$$\begin{equation}J = \sum_{i=1}^V \sum_{j=1}^V \; ( w_i^T\tilde{w}_j + b_i +
\tilde{b}_j - logX_{ij} ) ^2\end{equation}$$

其中 V
是词汇表的大小。单纯用平方差作为损失函数存在一个问题，就是所有的共现情况都具有一样的权重，这样一些罕见的或者携带无用信息的共现会和携带更多信息的共现组合的地位平等，这其实是一种噪音，所以作者
**对损失函数进行加权，希望更多地关注共现多的词。** 于是损失函数变成：

$$\begin{equation}J = \sum_{i=1}^V \sum_{j=1}^V \; f(X_{ij}) (
w_i^T\tilde{w}_j + b_i + \tilde{b}_j - logX_{ij} ) ^2\end{equation}$$

为了让权重函数达到降噪的目的，它需要满足几个条件：

  * f(0) = 0；
  * f(x) 应该是非递减函数，这样稀有的共现就不会具有大的权重；
  * 对于较大的 x，f(x) 应该相对小一些，这样经常出现的共现就不会被赋予很大的权重，因为像“'it is”这种组合是经常共现的，但是却没有携带重要的信息，所以也不希望这样的情况权重过大。

通过实验，GloVe 的作者发现 **下面这个加权函数表现相对较好：**

$$\begin{equation}f(X_{ij}) =
\Biggl\\{\begin{array}{lr}(X_{ij}/x_{max})^\alpha \quad if \; X_{ij} <
x_{max}, & \\\1 \quad otherwise\\\\\end{array}\end{equation}$$

它的图像是这样的：

![](https://images.gitbook.cn/50e68cf0-596f-11ea-991f-a5176cd1689d)

作者还给出了两个超参数的参考值：$x_{max} = 100, \alpha=0.75$。可以看出开始时权重随 $X_{ij}$ 增加而逐渐增加，到
$x_{max}$ 之后为 1，整个函数值一直都不会大于 1。有了权重函数后我们就得到了最终的损失函数。

从 GloVe 的模型中，我们可以看出它会生成两个词向量 $w$、$\tilde{w}$，当 X 对称时， $w$、$\tilde{w}$
也是对称的，因为随机初始化时的值不一样，最后得到的结果会不一样，不过这两个词向量其实是等价的，最后会选择 $w$ + $\tilde{w}$
作为最终的词向量。

经过上面的内容，我们知道了 GloVe
想要达成两个目的，一个是利用全局的统计信息，而不仅仅是用局部信息，另一方面是想捕捉到词的含义，而且它在损失函数上还加了权重，为不同频率的共现组合予以不同的关注度。

GloVe 的优点是训练比较快，即使语料库较小时表现也不错，并且也能扩展到更大的语料库上，训练时改善较小就可以进行 early
stopping。当然这个方法会占大量内存，有时还会对学习率比较敏感。

### 接下来我们具体应用一下 GloVe

所用到的数据和前面的 CBOW、Skip-Gram 是一样的，所以在 **数据导入、数据预处理** 部分的代码也是一样的。

接着就是要准备 GloVe 算法要用到的数据，要按照窗口大小，遍历预处理后的文本数据，构造出每一对 (batch,labels)，并计算出它们的
weights=1/d。

之后就是用这样的数据构造 **共现矩阵** ，累计求和每一对 (batch,labels) 的 weights 之和。

数据都准备好之后，按照公式写出 **损失函数** 的形式，再选择 **优化算法** ，就可以开始 **训练 GloVe 算法** 了。

#### **1\. 导入所需要的库**

    
    
    %matplotlib inline
    from __future__ import print_function
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
    from scipy.sparse import lil_matrix
    import nltk 
    import operator 
    from math import ceil
    

* * *

#### **2\. 下载 Wikipedia 数据**

> <http://www.evanjones.ca/software/wikipedia2text.html>
    
    
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

读取文本文件，转化为一个单词列表，经过预处理后，有 3360286 个单词，我们可以打印看一下 words 里面的数据。

    
    
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
    
    
    
    Reading data...
    Data size 3360286
    Example words (start):  ['propaganda', 'is', 'a', 'concerted', 'set', 'of', 'messages', 'aimed', 'at', 'influencing']
    Example words (end):  ['favorable', 'long-term', 'outcomes', 'for', 'around', 'half', 'of', 'those', 'diagnosed', 'with']
    

#### **4\. 建立字典**

假设我们输入的文本是 "I like to go to school"：

  * dictionary：将单词映射为 ID，例如 {I:0, like:1, to:2, go:3, school:4}
  * reverse_dictionary：将 ID 映射为单词，例如 {0:I, 1:like, 2:to, 3:go, 4:school}
  * count：计数每个单词出现了几次，例如 [(I,1),(like,1),(to,2),(go,1),(school,1)]
  * data：将读取的文本数据转化为 ID 后的数据列表，例如 [0, 1, 2, 3, 2, 4]

此外，我们用 UNK 代表比较少见的单词。

    
    
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

这一步和 word2vec 有点区别，GloVe 中需是要获得每一批的 batch、labels、weights 数据。

例如我们取出 data 的前 8 个词：

    
    
    ['propaganda', 'is', 'a', 'concerted', 'set', 'of', 'messages', 'aimed']
    

窗口大小为 2 时，batch 从 'a' 开始，取其前面 2 个和后面 2 个词，一共组合成 4 对，即：

  * batch：['a', 'a', 'a', 'a']
  * labels：['propaganda', 'is', 'concerted', 'set']

weights 就是 1/d，d 为 batch 和 label 的距离，即：

  * weights：[1/2, 1, 1, 1/2]

    
    
    data_index = 0
    
    def generate_batch(batch_size, window_size):
      # 每读进一个数据 data_index 加 1
      global data_index 
    
      batch = np.ndarray(shape=(batch_size), dtype=np.int32)
      labels = np.ndarray(shape=(batch_size, 1), dtype=np.int32)
      weights = np.ndarray(shape=(batch_size), dtype=np.float32)
    
      # span 是整个窗口 [ skip_window target skip_window ] 的长度 
      span = 2 * window_size + 1 
    
      # 用 buffer 装整个 span 里面的数据
      buffer = collections.deque(maxlen=span)
    
      # 填充 buffer 更新 data_index
      for _ in range(span):
        buffer.append(data[data_index])
        data_index = (data_index + 1) % len(data)
    
      # 每个 target 要取的上下文词的个数
      num_samples = 2*window_size 
    
      # 内循环用 span 里面的数据先来得到 num_samples 个数据点
      # 外循环重复 batch_size//num_samples 次来得到整个批次数据
      for i in range(batch_size // num_samples):
        k=0
        # 得到 batch ，label ，weights，注意 label 里面没有 target 单词
        for j in list(range(window_size))+list(range(window_size+1,2*window_size+1)):
          batch[i * num_samples + k] = buffer[window_size]
          labels[i * num_samples + k, 0] = buffer[j]
          weights[i * num_samples + k] = abs(1.0/(j - window_size))
          k += 1 
    
        # 每次读取 num_samples 个数据后，span 向右移动一位得到新的 span
        buffer.append(data[data_index])
        data_index = (data_index + 1) % len(data)
      return batch, labels, weights
    
    print('data:', [reverse_dictionary[di] for di in data[:8]])
    
    for window_size in [2, 4]:
        data_index = 0
        batch, labels, weights = generate_batch(batch_size=8, window_size=window_size)
        print('\nwith window_size = %d:' %window_size)
        print('    batch:', [reverse_dictionary[bi] for bi in batch])
        print('    labels:', [reverse_dictionary[li] for li in labels.reshape(8)])
        print('    weights:', [w for w in weights])
    
    
    
    data: ['propaganda', 'is', 'a', 'concerted', 'set', 'of', 'messages', 'aimed']
    
    with window_size = 2:
        batch: ['a', 'a', 'a', 'a', 'concerted', 'concerted', 'concerted', 'concerted']
        labels: ['propaganda', 'is', 'concerted', 'set', 'is', 'a', 'set', 'of']
        weights: [0.5, 1.0, 1.0, 0.5, 0.5, 1.0, 1.0, 0.5]
    
    with window_size = 4:
        batch: ['set', 'set', 'set', 'set', 'set', 'set', 'set', 'set']
        labels: ['propaganda', 'is', 'a', 'concerted', 'of', 'messages', 'aimed', 'at']
        weights: [0.25, 0.33333334, 0.5, 1.0, 1.0, 0.5, 0.33333334, 0.25]
    

#### **6\. 建立共现矩阵**

这一步是要获得共现矩阵 cooc_mat，也是 word2vec 中没有的，矩阵的大小是 `vocabulary_size *
vocabulary_size`，数据由 generate_batch 获得。

其中每一行对应着输入数据 batch data，每一列对应着 label， 坐标 (data, label) 上的值是相应 weights 的累加，这个
weights 也是由 generate_batch 得到。

如果不考虑权重的话，共现矩阵中的值应该是在指定窗口大小中，data 和 label 共现的累计次数，但这里加上了一个权重，即在每个共现的窗口中，累计的是
1/d，d 为 data 和 label 之间的间隔步数。

例如，输出 10 个 target 单词，以及各自的 10 个上下文单词，还包括每个词的 Id，共现矩阵中的 count 值。

    
    
    Target Word: "imagery"
    Context word:"and"(id:5,count:1.25), "UNK"(id:0,count:1.00), "generated"(id:3145,count:1.00), "demonstrates"(id:10422,count:1.00), "explored"(id:5276,count:1.00), "horrific"(id:16241,count:1.00), "("(id:13,count:1.00), ","(id:2,count:0.58), "computer"(id:936,count:0.50), "goya"(id:22688,count:0.50), 
    
    
    
    cooc_data_index = 0
    dataset_size = len(data)     # 遍历整个文本
    skip_window = 4             # 窗口大小
    
    # 共现矩阵 cooc_mat，它的大小是 vocabulary_size * vocabulary_size
    cooc_mat = lil_matrix((vocabulary_size, vocabulary_size), dtype=np.float32)
    
    print(cooc_mat.shape)
    def generate_cooc(batch_size,skip_window):
        data_index = 0
        print('Running %d iterations to compute the co-occurance matrix'%(dataset_size//batch_size))
        for i in range(dataset_size//batch_size):
            if i>0 and i%100000==0:
                print('\tFinished %d iterations'%i)
    
            # 生成一批数据
            batch, labels, weights = generate_batch(batch_size, skip_window)
            labels = labels.reshape(-1)
    
            # 共现矩阵的每一行对应着输入数据 batch data，每一列对应着 label，
            # 坐标（data，label）上的值是相应 weights 的累加
            for inp,lbl,w in zip(batch,labels,weights):            
                cooc_mat[inp,lbl] += (1.0*w)
    
    # 生成共现矩阵
    generate_cooc(8,skip_window)    
    
    # 打印几个样本
    print('Sample chunks of co-occurance matrix')
    
    # 选择几个词打印看一下结果，要计算它们的最多的几个共现词
    for i in range(10):
        # 输出 10 个 target 单词
        idx_target = i
    
        # 在共现矩阵中，取出 target 单词相应的行    
        ith_row = cooc_mat.getrow(idx_target)     
        ith_row_dense = ith_row.toarray('C').reshape(-1)        
    
        # 选择合适的 target 词
        while np.sum(ith_row_dense)<10 or np.sum(ith_row_dense)>50000:
            # 随机选择一个词
            idx_target = np.random.randint(0,vocabulary_size)
    
            ith_row = cooc_mat.getrow(idx_target) 
            ith_row_dense = ith_row.toarray('C').reshape(-1)    
    
        print('\nTarget Word: "%s"'%reverse_dictionary[idx_target])
    
        # target 的这一行，按照 count 从大到小的顺序，取出其相应的位置，组成 sort_indices
        sort_indices = np.argsort(ith_row_dense).reshape(-1) 
        sort_indices = np.flip(sort_indices,axis=0) 
    
        # 对每个 target 按照 count 从大到小取出前 10 个上下文单词，输出 Id，和共现矩阵中的 count 值
        print('Context word:',end='')
        for j in range(10):        
            idx_context = sort_indices[j]       
            print('"%s"(id:%d,count:%.2f), '%(reverse_dictionary[idx_context],idx_context,ith_row_dense[idx_context]),end='')
        print()
    

#### **7\. GloVe 算法部分**

**1\. 定义超参数**

超参数部分也和前面的 word2vec 几乎差不多。

    
    
    batch_size = 128                 # 每批数据的个数                   
    embedding_size = 128             # embedding 向量的维度
    window_size = 4                 # data 的左右两边各取 4 个 label
    
    valid_size = 16                 # 随机选择一个 validation 集来评估单词的相似性
    valid_window = 50                # 从一个大窗口随机采样 valid 数据
    
    # 选择 valid 样本时, 取一些频率比较大的单词，也选择一些适度罕见的单词
    valid_examples = np.array(random.sample(range(valid_window), valid_size))
    valid_examples = np.append(valid_examples,random.sample(range(1000, 1000+valid_window), valid_size),axis=0)
    
    num_sampled = 32                 # 负采样的样本个数
    
    epsilon = 1                     # 用来控制损失函数中 log 的稳定性
    

**2\. 定义输入输出**

    
    
    tf.reset_default_graph()
    
    # 训练数据 target 的 ID
    train_dataset = tf.placeholder(tf.int32, shape=[batch_size])
    # 训练数据 label 的 ID
    train_labels = tf.placeholder(tf.int32, shape=[batch_size])
    # Validation 不需要用 placeholder，因为前面已经定义了验证集数据的 ID 
    valid_dataset = tf.constant(valid_examples, dtype=tf.int32)
    

**3\. 定义模型的参数、变量**

    
    
    in_embeddings = tf.Variable(tf.random_uniform([vocabulary_size, embedding_size], -1.0, 1.0),name='embeddings')
    in_bias_embeddings = tf.Variable(tf.random_uniform([vocabulary_size],0.0,0.01,dtype=tf.float32),name='embeddings_bias')
    
    out_embeddings = tf.Variable(tf.random_uniform([vocabulary_size, embedding_size], -1.0, 1.0),name='embeddings')
    out_bias_embeddings = tf.Variable(tf.random_uniform([vocabulary_size],0.0,0.01,dtype=tf.float32),name='embeddings_bias')
    

**4\. 定义损失函数**

损失函数和上文中提到的一样：

$$\begin{equation}J = \sum_{i=1}^V \sum_{j=1}^V \; f(X_{ij}) (
w_i^T\tilde{w}_j + b_i + \tilde{b}_j - logX_{ij} ) ^2\end{equation}$$

embed_in 由 in_embeddings 得到，对应着数据 data，相当于公式中的 $w_i$，embed_bias_in 对应着偏置
$b_i$。

embed_out 由 out_embeddings 得到，对应着数据 label，相当于公式中的 $w_j$，embed_bias_out 对应着偏置
$b_j$。weights_x 就是损失函数中的权重。

$x_{ij}$ 是共现矩阵中坐标为 (data,label) 相应位置的值。

    
    
    embed_in = tf.nn.embedding_lookup(in_embeddings, train_dataset)
    embed_out = tf.nn.embedding_lookup(out_embeddings, train_labels)
    embed_bias_in = tf.nn.embedding_lookup(in_bias_embeddings,train_dataset)
    embed_bias_out = tf.nn.embedding_lookup(out_bias_embeddings,train_labels)
    
    # 损失函数中的权重
    weights_x = tf.placeholder(tf.float32,shape=[batch_size],name='weights_x') 
    # （data，label）相应位置的值
    x_ij = tf.placeholder(tf.float32,shape=[batch_size],name='x_ij')
    
    # 损失函数和上文中提到的一样
    loss = tf.reduce_mean(
        weights_x * (tf.reduce_sum(embed_in*embed_out,axis=1) + embed_bias_in + embed_bias_out - tf.log(epsilon+x_ij))**2)
    

**5\. 计算单词相似度**

得到 embedding 后，求其范数，用来进行标准化，再用余弦距离衡量单词向量的相似度。

    
    
    embeddings = (in_embeddings + out_embeddings)/2.0
    norm = tf.sqrt(tf.reduce_sum(tf.square(embeddings), 1, keepdims=True))
    normalized_embeddings = embeddings / norm
    valid_embeddings = tf.nn.embedding_lookup(normalized_embeddings, valid_dataset)
    similarity = tf.matmul(valid_embeddings, tf.transpose(normalized_embeddings))
    

**6\. 选择优化算法**

    
    
    optimizer = tf.train.AdagradOptimizer(1.0).minimize(loss)
    

#### **8\. 训练 GloVe 算法**

在每一次迭代中，首先用 generate_batch 得到 batch_data、batch_labels。

然后计算出 batch_xij 和 batch_weights。

batch_xij 是要获得当前批次的共现矩阵中坐标为 (data,label) 相应位置的值，batch_weights
为损失函数中用到的权重，包含了当前一批数据中每一个数据应该赋予的权重，它的计算公式就是前文提到的一样。

$$\begin{equation}f(X_{ij}) =
\Biggl\\{\begin{array}{lr}(X_{ij}/x_{max})^\alpha \quad if \; X_{ij} <
x_{max}, & \\\1 \quad otherwise\\\\\end{array}\end{equation}$$

其中，$x_{max} = 100, \alpha=0.75$。

    
    
    num_steps = 100001
    glove_loss = []
    
    average_loss = 0
    with tf.Session(config=tf.ConfigProto(allow_soft_placement=True)) as session:
    
        tf.global_variables_initializer().run()
        print('Initialized')
    
        for step in range(num_steps):
    
            # 在每一次迭代中，首先用 generate_batch 得到 batch_data, batch_labels
            batch_data, batch_labels, batch_weights = generate_batch(
                batch_size, skip_window) 
    
            batch_weights = []     
            batch_xij = [] 
    
            for inp,lbl in zip(batch_data,batch_labels.reshape(-1)):   
                # batch_weights 为损失函数中用到的权重，包含了当前一批数据中每一个数据应该赋予的权重，
                # 它的计算公式就是前文提到的，与每个xij有关
                point_weight = (cooc_mat[inp,lbl]/100.0)**0.75 if cooc_mat[inp,lbl]<100.0 else 1.0 
                batch_weights.append(point_weight)
                # batch_xij 是要获得当前批次的共现矩阵中坐标为（data，label）相应位置的值
                batch_xij.append(cooc_mat[inp,lbl])
            batch_weights = np.clip(batch_weights,-100,1)
            batch_xij = np.asarray(batch_xij)
    
            # 将训练数据喂给 optimizer 和 loss，进行优化
            feed_dict = {train_dataset : batch_data.reshape(-1), train_labels : batch_labels.reshape(-1),
                        weights_x:batch_weights,x_ij:batch_xij}
            _, l = session.run([optimizer, loss], feed_dict=feed_dict)
    
            # 更新平均损失变量
            average_loss += l
            if step % 2000 == 0:
              if step > 0:
                average_loss = average_loss / 2000
              # 计算平均损失
              print('Average loss at step %d: %f' % (step, average_loss))
              glove_loss.append(average_loss)
              average_loss = 0
    
            # 计算每个验证词的最近的 k 个词
            if step % 10000 == 0:
              sim = similarity.eval()
              for i in range(valid_size):
                valid_word = reverse_dictionary[valid_examples[i]]
                top_k = 8                     # 选最近的 k 的邻居
                nearest = (-sim[i, :]).argsort()[1:top_k+1]
                log = 'Nearest to %s:' % valid_word
                for k in range(top_k):
                  close_word = reverse_dictionary[nearest[k]]
                  log = '%s %s,' % (log, close_word)
                print(log)
    
        final_embeddings = normalized_embeddings.eval()
    

以上就是 GloVe 算法的应用，代码的前面一部分和 word2vec 的差不多，当然也可以将可视化的代码加进来。

参考文献：

  * Pennington et al. 2014, [_GloVe: Global Vectors for Word Representation_](https://nlp.stanford.edu/pubs/glove.pdf)
  * [斯坦福 GloVe 官网](https://nlp.stanford.edu/projects/glove/)
  * Thushan Ganegedara, _Natural Language Processing with TensorFlow_

