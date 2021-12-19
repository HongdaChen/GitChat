前两篇我们讲了 BERT 的原理和应用，谷歌在发布 BERT 时就引起了不小的轰动，因为当时 BERT 在 11 项 NLP
任务测试中刷新了当时的最高成绩，这个震撼还未平息，CMU 与谷歌大脑提出的 XLNet 又掀起一阵高潮，它在 20 个 NLP 任务上超过了 BERT
的表现，尤其是在难度很大的大型 QA 任务 RACE 上也足足超越 BERT 模型 6~9 个百分点，在其中 18
个任务上都取得了当时最佳效果。今天我们就来看看 XLNet 的原理和应用。

* * *

前面的课程中我们有讲过语言模型，即根据上文内容预测下一个可能的单词，这种类型的语言模型也被称为 **自回归语言模型** 。例如 GPT
是典型的自回归语言模型，此外 ELMo 本质上也是自回归语言模型，虽然它使用了双向 LSTM，但其实在每个方向上都是一个单向的自回归语言模型，两个方向上的
LSTM 的训练过程其实是独立的，只是最后将两个方向的隐节点状态拼接到一起。

![](https://images.gitbook.cn/3a0ef4b0-a0a8-11ea-bf38-950ba54cfedc)

另外还有一种语言模型叫做 **自编码语言模型** ，BERT 就是典型的代表，它在预训练时随机将所有句子中 15% 的 token 用 <Mask>
来替代，然后再根据上下文来预测这些被替代掉的原单词，这种方法可以使模型充分用到上下文的信息。

不过这两类语言模型都有不足之处，自回归语言模型只是单向的，不能考虑到双向的信息，自编码语言模型虽然具有了双向的功能，但是在预训练时会出现特殊的 <Mask>
token，可是到了下游的 fine-tuning 中又不会出现这些 <Mask>，这就出现了数据不匹配，会带来预训练的网络差异。

XLNet 就是为了充分结合二者的优点，弥补对方的不足而产生的，它可以让自回归语言模型部分具有双向的功能，也能让自编码语言模型部分的预训练和 Fine-
tuning 保持一致性。下面来看看 XLNet 是如何进行改进的。

### 自回归

自回归（AutoRegressive，AR）：给定一段序列 ${x_1,x_2,…,x_t}$，先使用 ${x_1}$ 预测 $x_2$，然后使用
${x_1,x_2}$ 预测 $x_3$，最后用 ${x_1,x_2,…,x_{t-1}}$ 预测
$x_t$，这样就可以生成整个句子，这种语言模型的目标是找出一个参数 θ 最大化 ${x_1,x_2,…,x_t}$
的对数似然函数，可以看出这个过程中下一个单词只依赖了上文：

$$\underset{\theta}{max}\; log p_\theta(\mathbf{x})=\sum_{t=1}^T log
p_\theta(x_t \vert \mathbf{x}_{<t})=\sum_{t=1}^T log
\frac{exp(h_\theta(\mathbf{x}_{1:t-1})^T
e(x_t))}{\sum_{x'}exp(h_\theta(\mathbf{x}_{1:t-1})^T e(x'))} $$

其中 𝐱<𝑡 表示 t 时刻之前的所有 x，ℎ𝜃(𝐱1:𝑡−1) 是 RNN 在 t 时刻之前的隐状态，𝑒(𝑥) 是词 x 的 embedding。

#### **XLNet 如何捕捉上下文？**

XLNet 想要保持自回归的方法来预测下一个单词，又想要利用上下文的信息，于是提出了 **Permutation Language
Modeling（PLM）** 方法。

![](https://images.gitbook.cn/a6b12ac0-a0a8-11ea-816e-999de338b69b)

PLM 这个方法的思想是：

假设有一个序列 ${x_1,x_2,x_3,x_4}$，先对这个序列做 permutation 排列组合，比如一句话 x = [This, is, a,
sentence] 有 4 个 token，就会有 4! 种排列可能，例如得到其中的一种排列情况
${x_2,x_4,x_3,x_1}$，然后随机选择一个单词作为预测目标，例如 $x_3$ 是当前的预测目标，在新的排列序列中可以看到，$x_3$
不仅可以用到上文的 $x_2$，也能用到下文的 $x_4$ 了，这样就实现了双向。

![](https://images.gitbook.cn/ba01ee20-a0a8-11ea-a7e9-93a4ac8821bf)

上图中✅表示 token 之间有 attetion，可以看到 $x_3$ 可以关联到 ${ x_3,x_2,x_4 }$，$x_4$ 可以关联到
${x_4,x_2}$，$x_2$ 可以关联到自己，空白是被 mask 了。

仍以上面的 `x = [This, is, a, sentence]` 为例，要根据前两个 token 预测第三个 token 时，排列 [1, 2, 3,
4]、[1, 2, 4, 3]、[4, 3, 2, 1] 对应的目标函数就是 $P (a, | This, is)$、$P(sentence | This,
is)$ 和 $P(is | sentence, a)$；要根据第一个 token 预测第二个 token 时，目标函数是 $P (is |
This)$、$P(is | This)$ 和 $P(a | sentence)$。

当然为了更具有操作性，不会真的将原始序列进行重排，因为这样需要记录很多数据，而且预测后还需要按照字典再排回去，所以作者在不需要变换原始序列的基础上，使用
mask 来实现排列组合。

![](https://images.gitbook.cn/ef1804a0-a0a8-11ea-bf38-950ba54cfedc)

例如原始序列为：${x_1,x_2,x_3,x_4}$，新的序列排列情况为 ${x_1,x_4,x_2,x3}$，如果 $x_3$
是当前的预测目标，如左图所示，$x_3$ 可以关联到 ${ x_1,x_4,x_2,x_3 }$，$x_2$ 可以关联到
${x_1,x_4,x_2}$，$x_4$ 可以关联到 ${x_1,x_4}$，$x_1$ 可以关联到自己，那么只是在有关联的地方加上
token，而矩阵本身的轴是不变的：

![](https://images.gitbook.cn/00000560-a0a9-11ea-97df-0d0e3bd6b465)

我们仍以 `x = [This, is, a, sentence]` 来具体说明，例如排列情况 [3, 2, 4, 1] 下：

  * 想要预测第一个 token=3 的话，因为它前面没有任何信息，所以位置 mask 为 [0, 0, 0, 0]
  * 想要预测第二个 token=2 时，它前面要用到的为“3”，所以位置 mask 为 [0, 0, 1, 0]
  * 同理，第三个 token=4 时的位置 mask 为 [0, 1, 1, 0]
  * 第四个 token=1 时的位置 mask 为 [0, 1, 1, 1]，这样最后得到的 mask 矩阵为：

$$ \begin{bmatrix} 0 & 1 & 1 & 1 \\\ 0 & 0 & 1 & 0 \\\ 0 & 0 & 0 & 0 \\\ 0 & 1
& 1 & 0 \\\ \end{bmatrix} $$

这样训练目标就变成了：

$$P(This | -, is+2, a+3, sentence+4)$$

$$P(is | -, -, a+3, -)$$

$$P(a | -, -, -, -)$$

$$P(sentence | -, is+2, a+3, -)$$

### 自编码

自编码（AutoEncoding，AE）：将输入数据构造成一个低维的特征是编码部分，然后再用解码器将特征恢复成原始的数据。

BERT 就是一种去噪自编码方法，它在原始数据上加了 15% 的 Mask Token 构造了带噪声的特征，然后试图通过上下文来恢复这些被掩盖的原始数据。

$$\underset{\theta}{max}\;log p_\theta(\bar{\mathbf{x}} | \hat{\mathbf{x}})
\approx \sum_{t=1}^Tm_t log p_\theta(x_t | \hat{\mathbf{x}})=\sum_{t=1}^T m_t
log \frac{exp(H_\theta(\mathbf{x})_{t}^T
e(x_t))}{\sum_{x'}exp(H_\theta(\mathbf{x})_{t}^T e(x'))} $$

其中 $𝑚_𝑡=1$ 时则表示 t 时刻是一个 Mask，需要恢复，$𝐻_𝜃$ 是一个 Transformer，用来将长度为𝑇 的序列 𝐱
映射为隐状态的序列 $𝐻_𝜃(𝐱)=[𝐻_𝜃(𝐱)_1,𝐻_𝜃(𝐱)_2,...,𝐻_𝜃(𝐱)_𝑇]$。

此外 BERT 还有一个问题是如果一个序列中被 <Mask> 的有两个以上时，在预测时，这样这些被 <Mask>
的位置就变成了相互独立的了，而事实上它们之间是可能存在依赖关系的，如下图所示，$x_3$ 和 $x_4$ 之间应该是有依赖关系的，可是在模型训练过程中是用
${x_1,x_2,x_5}$ 一起预测 ${x_3,x_4}$ 的，为了解决这个问题，就需要用到自回归的方法先预测 $x_3$ 再预测 $x_4$：

$$p(x_3|x_1, x_2, x_5) * p(x_4|x_1, x_2, x_5) $$

![](https://images.gitbook.cn/6c94cdf0-a0a9-11ea-9d24-cfb0df3065fc)

#### **XLNet 如何取代 <Mask>？**

BERT 通过 <Mask> 来传递预测目标的位置和上下文信息，XLNet 通过 Conten stream 和 Query stream 这种
**Two-Stream Self-Attention** 来实现这两个功能。Content stream 负责学习上下文信息，Query stream
负责代替 <Mask>，Query stream 只存在于预训练时，在 Finetune 时就不用了。

![](https://images.gitbook.cn/bf119860-a0a9-11ea-a321-115ce75343e0)

**1\. Content Stream**

Content Stream 就是一个标准的 self-attention，如图所示，h1 在进行 Self-Attention 时会用到 QKV，其中 Q
是 h1，KV 是 h1~h4，然后会得到 Attention weight，再和 V 相乘得到 h1 在下一层的表示：

![](https://images.gitbook.cn/ec92eaf0-a0a9-11ea-853e-a34978cba4d6)

$$h^m_{z_t} = Attention(Q = h^{m-1}_{z_t}, KV = h^{m-1}_{z_{\leq t}};
\theta)$$

**2\. Query Stream**

Query Stream 的作用是预测单词，在预测时模型不能知道实际的 token 是什么，此时又不能使用 mask，于是作者设置了一个 g
表示，如图所示，用 g1 去作用 h2~h4，根据公式计算 Attention，从公式中可以看到 Query stream 将当前 t 位置的
attention weight 掩盖掉：

![](https://images.gitbook.cn/fee63b80-a0a9-11ea-8705-c338ee6eeef7)

$$g^m_{z_t} = Attention(Q = g^{m-1}_{z_t}, KV = h^{m-1}_{z_{<t}}; \theta)$$

也就是说每个 token 的位置 i 在每个 self-attention 层 m 上有两个相关的向量：$h^m_i$、$g^m_i$，其中 h 属于
content stream，g 属于 query stream。

  * h 向量初始化为 token embeddings 加 positional embeddings
  * g 向量初始化为 generic embedding 加 positional embeddings

其中 generic embedding 向量 w 与 token 无关，即不管 token 是什么 w 都是一样的。

例如下图表示了如何计算第 m 层的第 4 个 token 的 g 向量，可以看到 $g^m_4$ 的计算用到了 is+2、a+3 和 w = 4
的信息，也就是在预测 sentence 这个单词时要考虑第二个位置的 is，第三个位置的 a，以及自身的位置 4：

![](https://images.gitbook.cn/3351ba20-a0aa-11ea-a7e9-93a4ac8821bf)

这样模型的训练目标变成了：

$$P(This | *, is+2, a+3, sentence+4)$$

$$P(is | -, *, a+3, -)$$

$$P(a | -, -, *, -)$$

$$P(sentence | -, is+2, a+3, *)$$

其中 `*` 表示当前正在计算概率的 token 位置。

### XLNet 具有大型文本学习能力

此外，作者还改进了预训练的架构设计，借鉴了 Transformer-XL 的 Segment recurrence mechanism 分段重复机制和
Relative positional encoding 相对位置编码两种方法，简单说就是让不同的 segment 之间可以互相做 Attention：

$$h^m_{z_t} = Attention(Q = h^{m-1}_{z_t}, KV = [\tilde h^{m-1},
h^{m-1}_{z_{\leq t}}]; \theta), [.,.]: Concatenate$$

这部分如果大家对细节感兴趣可以看看这篇文章：[XLNet 原理](http://fancyerii.github.io/2019/06/30/xlnet-
theory/)。

这样 XLNet 通过 PLM 既保持了 ELMo, GPT 等模型的自回归性质，又具有了 BERT 一样的捕捉双向信息的功能，还通过 Query
stream 代替了 BERT 的 <Mask> 的作用，最后还像 Transformer-XL 一样能处理大型文本，这些改进使 XLNet
在长文本处理任务中有突出的表现。

### XLNet 应用

下面我们来通过一个简单的例子看如何应用 XLNet 模型进行文本分类任务。

所用的数据为 SST2 电影评论情感分析数据集，第一列为评论，第二列为情感标签，一共有两类，1 为积极评论，0 为消极评论。

![](https://images.gitbook.cn/67e189a0-a0aa-11ea-816e-999de338b69b)

#### **1\. 加载库**

    
    
    !pip install transformers
    
    import numpy as np
    import pandas as pd
    from sklearn.model_selection import train_test_split
    from sklearn.linear_model import LogisticRegression
    from sklearn.model_selection import GridSearchCV
    from sklearn.model_selection import cross_val_score
    import torch
    import transformers as ppb
    import warnings
    warnings.filterwarnings('ignore')
    

#### **2\. 导入数据**

这里我们以前 2000 个样本为例。

    
    
    df = pd.read_csv('https://github.com/clairett/pytorch-sentiment-classification/raw/master/data/SST2/train.tsv', delimiter='\t', header=None)
    
    batch_1 = df[:2000]
    

#### **3\. 加载预训练 XLNet 模型**

    
    
    # xlnet
    model_class, tokenizer_class, pretrained_weights = (ppb.XLNetModel, ppb.XLNetTokenizer, 'xlnet-base-cased')
    
    # 加载预训练模型和 tokenizer
    tokenizer = tokenizer_class.from_pretrained(pretrained_weights)
    model = model_class.from_pretrained(pretrained_weights)
    

这里还可以换成其他模型，大家可以在《[transformers
官方文档](https://huggingface.co/transformers/model_doc/xlnet.html#xlnettokenizer)》里找到其他模型的使用方法。

例如：

    
    
    # Albert
    model_class, tokenizer_class, pretrained_weights = (ppb.AlbertModel, ppb.AlbertTokenizer, 'albert-base-v2')
    
    # DistilBERT:
    model_class, tokenizer_class, pretrained_weights = (ppb.DistilBertModel, ppb.DistilBertTokenizer, 'distilbert-base-uncased')
    
    # BERT
    model_class, tokenizer_class, pretrained_weights = (ppb.BertModel, ppb.BertTokenizer, 'bert-base-uncased')
    

![](https://images.gitbook.cn/84a45c70-a0aa-11ea-9d24-cfb0df3065fc)

[图片来源](https://www.kdnuggets.com/2019/09/bert-roberta-distilbert-xlnet-one-
use.html)

#### **4\. 准备数据**

Tokenization：将文本转化为数字向量。

    
    
    tokenized = batch_1[0].apply((lambda x: tokenizer.encode(x, add_special_tokens=True)))
    

Padding：将数据补齐成指定长度。

    
    
    max_len = 0
    for i in tokenized.values:
        if len(i) > max_len:
            max_len = len(i)
    
    padded = np.array([i + [0]*(max_len-len(i)) for i in tokenized.values])
    np.array(padded).shape
    

Masking：告诉模型哪些是补齐的位置。

    
    
    attention_mask = np.where(padded != 0, 1, 0)
    attention_mask.shape
    

#### **5\. 应用模型**

从前面得到的 token 矩阵中创建了一个输入张量，将其传递给模型，last_hidden_states 是模型的输出：

    
    
    input_ids = torch.tensor(padded)  
    attention_mask = torch.tensor(attention_mask)
    
    with torch.no_grad():
        last_hidden_states = model(input_ids, attention_mask=attention_mask)
    

因为模型对每个句子的第一个 token 的输出感兴趣，所以用 features 保存这些数据，并用 labels 保存句子的标签：

![](https://images.gitbook.cn/8140a290-a0ab-11ea-97df-0d0e3bd6b465)

    
    
    features = last_hidden_states[0][:,0,:].numpy()
    
    labels = batch_1[1]
    

划分训练集和测试集：

    
    
    train_features, test_features, train_labels, test_labels = train_test_split(features, labels)
    

训练逻辑回归模型用来进行分类预测：

    
    
    lr_clf = LogisticRegression()
    lr_clf.fit(train_features, train_labels)
    

分类准确率：

    
    
    # xlnet：
    lr_clf.score(test_features, test_labels)
    #0.732
    

大家感兴趣可以换成 BERT 等其他模型对比一下效果。

* * *

**面试题：**

  * XLNet 为什么会比 BERT 效果好？

**参考文献：**

  * [_XLNet: Generalized Autoregressive Pretraining for Language Understanding_](https://arxiv.org/pdf/1906.08237.pdf)
  * Jay Alammar, [_A Visual Guide to Using BERT for the First Time_](http://jalammar.github.io/a-visual-guide-to-using-bert-for-the-first-time/)

