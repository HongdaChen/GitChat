**Transformer﻿** 是由 Google 团队的 Ashish Vaswani 等人在 2017 年 6 月发表的论文 [_Attention
Is All You Need_](https://arxiv.org/abs/1706.03762) 中提出的 NLP 经典之作，这个模型可以算是近几年来
NLP 领域的一个重大的里程碑，在它之前 seq2seq + Attention 就表现很强了，结果这篇论文一出来就引起了不小的轰动，它竟然不需要任何
RNN 等结构，只通过注意力机制就可以在机器翻译任务上超过 RNN，CNN 等模型的表现。

![](https://images.gitbook.cn/e6039c80-8965-11ea-91b0-1940e1c2e5e5)

[图片来源](https://ai.googleblog.com/2017/08/transformer-novel-neural-
network.html)

### Transformer﻿ 和 RNN 比较

在机器翻译任务中，虽然说在 Transformer 之前 Encoder-Decoder + Attention 结构已经有很好的表现了，但是其中的 RNN
结构却存在着一些不足。

  * 首先，RNN 模型不擅长并行计算。因为 RNN 具有序列的性质，就是当模型处理一个状态时需要依赖于之前的状态，这个性质不利于使用 GPU 进行计算，即使用了 CuDNN，RNN 在 GPU 上也还是很低效的。
  * 而 Transformer 最大的优点就是可以高效地并行化﻿，因为它的模型内部的核心其实就是大量的矩阵乘法运算，能够很好地用于并行计算，这也是 Transformer 很快的原因之一。
  * 另一个不足就是 RNN 在学习长期依赖上存在困难，可能存在梯度消失等问题。尽管 LSTM 和 Attention 在理论上都可以处理长期记忆，但是记忆这些长期信息也是一个很大的挑战。
  * 而 Transformer 可以用注意力机制来对这些长期依赖进行建模，尤其是论文还提出了 Multi-Head Attention，让模型的性能更强大。

### Transformer﻿ 模型

Transformer﻿ 的基本框架本质上就是一种 Encoder-Decoder 结构，如下图所示是它的基本单元，左侧是 Encoder 部分，右侧是
Decoder：

![](https://images.gitbook.cn/3b30b9e0-8966-11ea-91b0-1940e1c2e5e5)

[图片来源](https://arxiv.org/pdf/1706.03762.pdf)

在 Encoder 单元中，可以看出由两块子层组成，Multi-Head Attention 和 FFN：

  1. Input 经过 Word Embedding 后变成向量表示，接着要做 Positional Encoding 获取位置信息；
  2. 然后进入 Encoder 单元，第一个部分也叫 Encoder 的 Self-Attention 层，用的是 Multi-Head Attention 机制；
  3. 第二个部分是 Position-wise Feed Forward 层；
  4. 每个子层之间都有残差连接和 Layer Normalization。

在 Decoder 中，主要有三块子层，Masked Multi-Head Attention、Multi-Head Attention 和 FFN：

  1. Output 经过 Embedding，也要做 Positional Encoding；
  2. 在 Decoder 单元中，第一部分是 Decoder 的 Self-Attention 层，用的是 Masked Multi-Head Attention；
  3. 第二部分是 Encoder-Decoder Attention 层，即在做 Multi-Head Attention 时要考虑 Encoder 的结果；
  4. 第三部分是 Position-wise Feed Forward 层；
  5. 每个子层之间也要做残差连接和 Layer Normalization；
  6. 最后一个 Decoder 的输出要经过 Linear 和 Softmax 函数来输出概率结果。

上图所示的结构是 Encoder 和 Decoder 的基本单元：

  1. 可以看到两侧都有个 N，意思是可以叠加多个单元，在论文中 Transformer 的 Encoder 由 6 个单元叠加组成，Decoder 也由 6 个单元组成；
  2. 每个 Encoder/Decoder 的输出是它正上方 Encoder/Decoder 的输入；
  3. Encoder 最后的输出会和每一层 Decoder 进行结合：

![](https://images.gitbook.cn/66a6d320-8966-11ea-b21c-fd2ea4a0d841)

[图片来源](http://jalammar.github.io/illustrated-transformer/)

下面我们从 Encoder 到 Decoder 更具体地介绍每个组件。

#### **Word Embedding**

这个大家应该很熟悉了，我们在前面的课程中也讲了很多，在这里就是构建一个嵌入矩阵，里面的参数可以随着模型进行训练。

#### **Positional Encoding**

我们知道 RNN 结构是可以记忆位置信息的，而 Transformer
中不具备这样的结构，只靠注意力是无法捕捉位置信息的，位置信息对理解文本内容是很重要的，因为很多时候位置会决定句子的含义，所以需要给输入的数据做一下
Positional Encoding，而且它是 Transformer 中唯一的位置信息的来源，是个很重要的部分。

Positional Encoding
就是对序列中词语出现的位置进行编码，它将每个单词的位置信息，或者序列中不同单词之间的距离等信息编码成向量，再将这些向量加入到前面得到的 Embedding
上面，组成最终的 Embedding 作为下一层的输入数据，这样 Transformer 就能区分不同位置的单词了：

![from：http://jalammar.github.io/illustrated-
transformer/](https://images.gitbook.cn/80cd0210-8966-11ea-8852-7d0c628374dd)

具体 Positional Encoding 的公式是：

$$ PE(pos, 2i) = sin( \frac{pos}{ 10000^{2i/d_{model}} } ) $$

$$ PE(pos, 2i + 1) = cos( \frac{pos}{ 10000^{2i/d_{model}} } ) $$

其中 pos 是位置的索引，代表词语在序列中的位置，i 是向量中的索引，d model 是模型的维度，论文中是 512，每个单词的位置编码长为 512。

这个公式可以将位置信息 pos
编码成正余弦序列向量，在向量的偶数位置，用正弦函数编码，在奇数位置，用余弦函数编码，这样位置编码向量的每个维度是不同频率的波形，每个值介于 -1 和 1
之间。

之所以要用正余弦函数编码，是因为想要捕捉单词的相对位置信息。我们知道正余弦函数具有下面的性质：

$$sin(\alpha + \beta) = sin\alpha cos\beta + cos \alpha sin \beta$$

$$cos(\alpha + \beta) = cos\alpha cos\beta + sin \alpha sin \beta$$

而这恰好可以表达相对位置的关系，将 $\alpha$ 看成位置 pos，$\beta$ 看成词汇之间的位置偏移 k，那么根据上面的公式，Positional
Encoding(pos + k) 就可以表示成 Positional Encoding(pos) 和 Positional Encoding(k)
的线形的组合形式，这样利用已知数据可以比较容易的推断出相对位置。

接着进入 Multi-Head Attention，正如它的名称一样，指的是将若干个不同的 Self-Attention
集成在一起，所以先来看看它的基础组件 Self-Attention。

#### **Self-Attention**

在前面的文章中我们有讲过经典的注意力机制，其中在计算分数时会用到两个状态，输入序列第 i 个位置的状态 $h_i$ 和输出序列第 t 个位置
$s_t$，Self-Attention 是指自己和自己做注意力，也就是 $s_t$ 也是从输入序列中获取。

Self-Attention
的作用是让模型在处理每个单词时，可以先在该单词所在的序列中找到比较相关的信息，并融入到当前正在处理的单词中，来达到更好的编码效果。

例如我们要翻译这句话﻿：”The animal didn't cross the street because it was too tired”，其中的
it 是指什么呢，它指的是 street 还是 animal 呢，这对人类来说是一个简单的问题，但对算法来说却并不简单，﻿而 Self-Attention
可以让算法知道这里的 it 指的是 animal﻿：

![](https://images.gitbook.cn/8e828df0-896e-11ea-91b0-1940e1c2e5e5)

这个 Self Attention 机制也叫做 Scaled dot-product attention，我们可以看看下面这张注意力机制的对比图：

![](https://images.gitbook.cn/9bb5c410-896e-11ea-9db6-d98fa851b4fc)

[图片来源](https://lilianweng.github.io/lil-log/2018/06/24/attention-
attention.html#summary)

可以看到 Scaled dot-product attention 其实和 Luong Attention 中的 Dot-Product
分数形式只差了一个缩放因子 $ \frac{1}{\sqrt
d_k}$，之所以用了缩放因子，是因为当向量维度很大时，做点积的话维度会变得更大，做梯度时结果就会处于很小的范围内，这样反向传播时就可能出现消失，所以加入缩放因子来减轻这种影响。

**具体原理：**

![](https://images.gitbook.cn/af37d0a0-896e-11ea-b610-d9d63cf53c0b)

**第一步，为编码器的每个输入单词创建三个向量﻿。**

即 Query vector、Key vector、Value vector﻿，这些向量是通过 Embedding 和三个矩阵相乘得到的，将 x1 乘以
WQ 得到 Query 向量 q1，同理得到 Key 向量和 Value 向量。

**第二步，是计算一个得分。**

假设我们要计算下面例子中第一个单词“Thinking”的 Self-
Attention，就需要根据这个单词，对输入句子的每个单词进行评分，这个分数决定了要给其他单词放置多少关注度。﻿分数的计算方法是，例如我们正在考虑
Thinking 这个词，就用它的 q1 去乘其他每个单词的 k 向量。

**第三步和第四步，是将得分加以处理再传递给 softmax。**

Self Attention 的核心公式也正是应用在这里：

$$ Attention(Q, K, V) = softmax( \frac{ QK^T }{ \sqrt{d_k} } ) $$

Transformer 对 score 进行归一化，即除以 $\sqrt{d_k}$，论文中使用的 key 向量的长度是 64，所以归一时将得分除以
8，这样可以有更稳定的梯度。

然后传递给 softmax，softmax 可以将分数标准化，这样加起来保证为 1。这个 softmax 分数决定了每个单词在该位置受关注的程度。

**第五步，用这个得分乘以每个 value 向量。**

softmax 点乘 Value 值，得到加权的每个输入向量的评分 v，目的是让我们想要关注的单词的值保持不变，并通过乘以 0.001
这样小的数字来淹没其他不相关的单词﻿。

**第六步，求和。**

相加之后得到最终的输出结果 z 是 v 的加权求和，得到的向量接下来要输入到前馈神经网络。

这个注意力机制其实就是在根据一些 keys 和 queries 计算一组信息 values 之间的相关性。

#### **Multi-Headed Attention 机制﻿**

Multi-Head Attention 就是将整个 Q、K、V 分成 h 份，每一份都进行上述的 Self-Attention，然后将 h
个结果合并起来，经过线性映射，得到最终的输出，是 Transformer
的关键结构。它的作用就是让模型能够关注到不同的位置，来捕捉到数据的多个角度的信息，进而提升注意力层的性能﻿。

前面虽然经过 Self-Attention 的计算，向量 z 已经包含了一点其他位置单词的信息，但当前单词还是占主要地位，而 Multi-Headed
Attention 可以为注意力层提供多个“表示子空间”。这样一个注意力可以关注到一方面的信息，多个注意力就可以关注到不同的信息，例如有句话是说 "I
like singing more than dancing"，其中一个注意力可能捕捉到这句话是在比较两个东西，另一个注意力会捕捉到这两个东西是什么。

具体做法如下。

**1\. 根据 Multi-Headed 定义的 Heads 数目，定义一样多的 Query/Key/Value 权重矩阵组。**

论文中用了 8 个，那么每个 Encoder/Decoder 我们都会得到 8 个矩阵。﻿这些矩阵都是随机初始化的，经过训练之后，每个矩阵会将输入
Embeddings 投影到不同的表示子空间中。﻿

**2\. 结合定义的 h 组权重矩阵，每个单词会进行 h 次 Self-Attention 计算﻿。**

这样每个单词会得到 h 个不同的加权求和向量 z﻿。

![](https://images.gitbook.cn/f6e126e0-896e-11ea-b610-d9d63cf53c0b)

**3\. 在 Feed Forward 处只接收一个矩阵，所以需要将这 h 个向量压缩成一个。**

方法就是先将 h 个 z 连接起来，然后乘以另一个的权重矩阵 WO﻿。

![](https://images.gitbook.cn/068eabd0-896f-11ea-a501-156f424bd17b)

下图可以看到 Multi-Headed 的效果，在这个例句中，it 的不同的注意力 Heads
所关注的位置，一个注意力的焦点主要集中在“animal”上，而另一个注意力的焦点集中在“tired”。

![](https://images.gitbook.cn/18a170f0-896f-11ea-91b0-1940e1c2e5e5)

#### **Residuals**

在每个 Encoder 和 Decoder 里面的 Self-Attention、FFN、Encoder-Decoder Attention
层都有残差连接。

残差就是在 F(x) 后面增加了一项 x，变成了 F(x) + x，这样在反向传播过程中，在对 x 求梯度的时候就多了个
1，梯度连乘时可以避免梯度消失，主要用来解决深度学习中的退化问题。

#### **Layer-Normalization**

Normalization 是指将数据转化成均值为 0 方差为 1 的数据，这样标准化的数据进入激活函数可以避免进入饱和区。

Layer-Normalization 是深度学习中一种常见的标准化方法，就是在每一个样本上计算均值和方差：

$$ LN(x_i) = \alpha * \frac{x_i - u_L}{\sqrt{ \sigma_L^2 + \epsilon }} + \beta
$$

#### **Position-wise Feed-forward Networks**

在每个 Encoder 和 Decoder 的最后都有一个 FFN 子层，就是一个全连接网络，包括两个线性变换和一个 ReLU
非线性函数，它会对不同位置的向量进行相同的操作，在不同的层之间使用不同的参数。

### Decoder

在每个 Decoder 中有两个注意力子层，一个是 Masked Multi-Head Attention，一个是 Encoder-Decoder
Multi-Head Attention。

#### **Masked Multi-Head Attention**

Decoder 中的 Self Attention 层与 Encoder 中的略有不同，就是要做 Mask。Mask
掩码，就是将一些数据掩盖起来，使其不产生效果。

常用的 Mask 有两种，Padding Mask 和 Sequence Mask。

Padding Mask 是将预处理时做了 padding 补齐操作的位置用一个非常大的负数替换，这样在经过 softmax
作用后，这些补齐位置的值就接近于 0，
这么做的目的是，因为这些填充的位置原本是没有信息的，所以在后续要应用注意力时也可以将其忽略，不应该被补齐的无意义的信心占据资源，这种 Mask
也最常用，因为我们很多时候都是要 padding 的。

在这里尤其要强调的是 Sequence Mask，Decoder
是一个生成的过程，我们需要训练它能够根据过去的信息来预测未来的内容，所以不希望它能直接获得将来的信息，这时就需要做掩码。Decoder 应用
Sequence Mask 这种掩码后就可以屏蔽掉当前时刻之后的信息，这样做是为了让 Self Attention
层只能关注输出序列中靠前的一些位置。具体做法就是建立一个上三角矩阵，上三角的值为 1，对角线和下三角的值为 0，将这个矩阵作用在输入上就可以了。

#### **Encoder-Decoder Multi-Head Attention**

在这个注意力层，Decoder 会同时考虑来自自身和 Encoder 的信息。此时要将最上面的 Encoder 的输出结果变成一组注意力向量 K 和
V，这些向量会用于 Decoder 中的每个 Encoder-Decoder Attention 层，而公式中的 Q 则来自 Decoder
的上一层输出，其他地方就和前面将的 Self Attention 基本一样了。这样 Decoder 中的注意力层就可以从输入序列中获得相关信息了。

![](https://images.gitbook.cn/8a4d36c0-8970-11ea-9db6-d98fa851b4fc)

#### **线性层**

Decoder 最后输出的是一个向量，如何把它变成一个单词，这就要靠它后面的线性层和 softmax 层。

线性层就是一个很简单的全连接神经网络，将 Decoder 输出的向量映射成一个更长的向量。﻿例如我们有 10000
个无重复的单词，那么最后输出的向量就有一万维，向量中每个位置上的值代表了相应单词的分数。

#### **最后 softmax 层将这个分数转换为概率**

选择概率最大的单词作为当前时间步的输出结果。﻿

今天介绍了 Transformer 的模型基础，下一篇我们将动手实现一下，来进一步理解其原理。

**面试题：**

  * 什么是 Transformer？
  * Transformer 的优势有哪些？

* * *

参考文献：

  * Jay Alammar, [The Illustrated Transformer](http://jalammar.github.io/illustrated-transformer/)
  * Ashish Vaswani et al, [_Attention Is All You Need_](https://arxiv.org/pdf/1706.03762.pdf)

