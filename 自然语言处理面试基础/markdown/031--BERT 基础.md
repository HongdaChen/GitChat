BERT，Bidirectional Encoder Representations from Transformers，来自 Google AI
Language 的论文：

> [_BERT: Pre-training of Deep Bidirectional Transformers for Language
> Understanding_](https://arxiv.org/pdf/ 1810.04805.pdf)

是在和 BooksCorpus 集上预训练模型，然后应用于其他具体任务上时再进行微调。

BERT 模型可以用于问答系统，情感分析，垃圾邮件过滤，命名实体识别，文档聚类等多种任务中，当时 BERT 模型在 11 个 NLP
任务上的表现刷新了记录，在自然语言处理领域引起了不小的轰动，这些任务包括问答 Question Answering（SQuAD v1.1），推理
Natural Language Inference（MNLI）等：

    
    
    GLUE ：General Language Understanding Evaluation
    MNLI ：Multi-Genre Natural Language Inference
    SQuAD v1.1 ：The Standford Question Answering Dataset
    QQP ： Quora Question Pairs 
    QNLI ： Question Natural Language Inference
    SST-2 ：The Stanford Sentiment Treebank
    CoLA ：The Corpus of Linguistic Acceptability 
    STS-B ：The Semantic Textual Similarity Benchmark
    MRPC ：Microsoft Research Paraphrase Corpus
    RTE ：Recognizing Textual Entailment 
    WNLI ：Winograd NLI
    SWAG ：The Situations With Adversarial Generations
    

Google 也将 BERT 应用于搜索上，并称它为搜索历史上的最大飞跃之一，因为 BERT
可以更好地理解用户搜索的真实意图，它通过区分一些细微差别来找到更相关结果，例如“nine to five”和“a quarter to
five”中的“to”是不一样的含义，对人类来说这两个含义不同很明显，但是对于搜索引擎来说却很困难，以往的模型中“to”会得到一样的表示，但是 BERT
会区分出二者的差别，这就可以提升搜索结果的质量，帮助用户找到他们真正想要的东西。

### BERT 原理

概括地说 BERT 是一个多层双向 Transformer Encoder 的堆栈。

我们在前面的文章中知道 Transformer 是一种注意力机制，可以学习文本中单词之间的上下文关系。

在 Transformer 中包括两个机制，一个是 Encoder 负责接收文本作为输入，一个是 Decoder 负责预测结果。而 BERT
的目标是生成语言模型，所以只需要 Transformer 的 Encoder 机制。

在 BERT 这篇论文中提出了两种大小的模型：

  * **BERT BASE：** 这个是为了与 OpenAI Transformer 的表现进行比较而构建的，模型的大小也与它相当。包括 12 个 Transformer 编码器层，12 个注意力头，768 个隐藏单元，1.1 亿参数。
  * **BERT LARGE：** 一个非常巨大的模型，达到了当时最佳的表现。包括 24 个 Transformer 编码器层，16 个注意力头，1024 个隐藏单元，3.4 亿参数。

![](https://images.gitbook.cn/cd2d9000-9665-11ea-a9dc-ef5cf20ceba2)

我们可以回顾一下 Transformer 原始论文的模型配置是 6 个编码器层，8 个注意头，512 个隐藏单元。

Transformer 的 Encoder
是一次性读取整个文本序列，而不是从左到右或从右到左地按顺序读取，这个特征使得模型能够基于单词的两侧学习，相当于是一个双向的功能。

下图是单独提取出 Transformer 的 Encoder 部分，输入是一个 Token 序列，先对其进行 Embedding
成为词向量，然后输入给模型，输出是大小为 H 的向量序列：

![](https://images.gitbook.cn/76ef5b50-9666-11ea-bcac-1f458c1b219e)

（[图片来源](https://www.lyrn.ai/2018/11/07/explained-bert-state-of-the-art-
language-model-for-nlp/)）

**BERT 的创新点在于它将双向 Transformer 用于语言模型。**

我们知道语言模型是计算句子的概率：

![](https://images.gitbook.cn/8ef6f000-9666-11ea-96a2-752a88cf1dc1)

之前的模型是 Left-to-Right 输入一个文本序列，或者将 Left-to-Right 和 Right-to-Left 的训练结合起来，就是在前向
RNN 构成的语言模型中，当前词的概率只依赖前面出现词的概率，而后向 RNN
构成的语言模型中，当前词的概率只依赖后面出现的词的概率，每个方向都是单向的而不是同时双向。

BERT 采用 Transformer 的 Encoder，也就是说它在每个时刻计算注意力时都会考虑到所有时刻的输入，实验的结果也表明
**这种双向训练的语言模型对语境的理解会比单向的语言模型更深刻。**

不过当我们在训练语言模型时，有一个挑战就是要定义一个预测目标，一般模型是在一个序列中单向地预测下一个单词，而双向的方法在这样的任务中是有限制的，为了克服这个问题，
**BERT 提出了两个策略：**

  1. Masked LM（MLM）
  2. Next Sentence Prediction（NSP）

### Masked LM（MLM）

Masked LM
即有掩码的语言模型，如果直接将双向模型用于语言模型，那么每个词都会在多层上下文中间接地看到自己，于是就用掩码来解决，在这个技术出现之前是无法进行双向语言模型训练的，具体方法是：

![](https://images.gitbook.cn/bd84a980-9666-11ea-96a2-752a88cf1dc1)

  * BERT 会掩盖 Input 中 15% 的词：在将 Input 序列输入给 BERT 之前，每个序列中有 15％ 的单词被 [MASK] Token 替换，然后模型尝试基于序列中其他未被掩码的单词上下文来预测被掩盖的原单词。
  * 除了掩蔽 15% 的输入，BERT 还混用了一些其他方法来改善模型之后的调整，因为在微调的时候看不到 [MASK] Token，所以会采用下面几种策略：
    * 80% 的情况下将选中的词用 [MASK] 代替，例如：my dog is hairy → my dog is [MASK]
    * 10% 的情况下将选中的词用任意词来进行代替，例如：my dog is hairy → my dog is apple
    * 10% 的情况下选中的词不发生变化，例如：my dog is hairy → my dog is hairy

这样的话和传统语言模型相比，BERT 更像是个分类模型，因为它要根据隐藏状态来预测这个时刻的 Token 应该是什么，而不是预测词的概率分布了。

为了实现这种分类技术，在模型上也与 Transformer 有些不同：

  1. 在 Encoder 的输出上添加了一个分类层；
  2. 用嵌入矩阵乘以输出向量，将其转换为词汇表的维度；
  3. 用 softmax 计算词汇表中每个单词的概率；
  4. BERT 的损失函数只考虑了掩码的预测值，忽略了没有掩码的单词的预测，这样模型要比单向模型收敛得慢，不过结果的情境意识也增加了。

![](https://images.gitbook.cn/1ba3c980-966a-11ea-958b-6d75f69bc560)

### Next Sentence Prediction（NSP）

在 BERT 的训练过程中，模型会接收成对的句子作为输入，50％ 的输入对在原始文档中是前后关系，另外 50％
是从语料库中随机组成的，并且是与第一句断开的，模型需要学习预测其中第二个句子是否在原始文档中也是后续句子，这个就叫做 **Next Sentence
Prediction** 。

为了帮助模型区分开训练中的两个句子，Input 在进入模型之前要按以下方式进行处理：

  1. Token Embedding，即在第一个句子的开头插入 [CLS] 标记，在每个句子的末尾插入 [SEP] 标记；
  2. 将表示句子 A 和句子 B 的 Sentence Embedding 添加到每个 Token 上；
  3. 给每个 Token 添加一个 Positional Embedding，来表示它在序列中的位置。

![](https://images.gitbook.cn/3a0cab30-966a-11ea-bcac-1f458c1b219e)

为了预测第二个句子是否是第一个句子的后续句子，分为下面几个步骤：

  1. 将整个 Input 序列输入给 Transformer 模型
  2. 用一个简单的分类层将 [CLS] 标记的输出变换为 2×1 的向量
  3. 用 softmax 计算 IsNextSequence 的概率

在训练 BERT 模型时，Masked LM 和 Next Sentence Prediction
是一起训练的，最终目标就是要最小化这两种策略的组合损失函数。

这里我们再进一步介绍一下上面提到的三种嵌入技术。

### Token Embedding

这一层的作用是将单词转化为指定维度的向量，论文中是 768 维。

![](https://images.gitbook.cn/531ab360-966a-11ea-9fd5-332242a3cf46)

  * 例如“I like strawberries”有 3 个单词；
  * 分词后加上 [CLS] 和 [SEP] 两个标记后有 6 个 Token，其中 [CLS] 表示的是当前的数据是为了进行分类任务，[SEP] 用来分隔一对输入文本；
  * 而且这里使用了 WordPiece Token 的方法，我们看到“strawberries”被分成“straw”和“berries”两个词，这种方法可以实现词汇量和非词汇量之间的平衡；
  * 经过一个 30522×768 维度的矩阵作用，得到结果的维度是 6 × 768。其中 30522 是 BERT 词汇表中存储的数量。

### Sentence Embedding：

当模型所处理的任务需要考虑一对 Input 时，需要这种嵌入方法。

![](https://images.gitbook.cn/737c09b0-966a-11ea-9fd5-332242a3cf46)

例如想要看“I like cats”、“I like dogs”这两个句子在语义上是否相似：

  * 首先将两个句子连起来并转化为向量，那么会有 8 个 Token；
  * 然后从 [CLS] 到 [SEP] 之间标记为 0，表示第一句，[SEP] 之后标记为 1，表示第二句，这里看出 [SEP] 的作用很明显了；
  * 接着作用 2×768 的矩阵，得到的结果维度为 8×768。

### Positional Embedding

在 Transformer 中我们知道了 Positional Encoding 的意义，是为了将单词的位置信息考虑进来。

在 BERT 中同样也要实现这样的功能，例如“I think, therefore I
am”这句话，第二个“I”与第一个“I”的向量表示应该是不同的，因为进入它们的隐藏层状态是不同的，第一个“I”那里隐藏状态刚初始化，第二个“I”那里隐藏状态已经是经过“I
think, therefore”的了。

如果不考虑位置而只看 Transformer 的 Self-Attention
层的话，尽管两个“I”的位置不同，但是由于影响它们输出的输入数据是一样的，所以得到的结果也会一样。

![](https://images.gitbook.cn/93d60d00-966a-11ea-a9dc-ef5cf20ceba2)

不过 BERT 里没有使用和 Transformer 一样的正弦编码方法。它的 Position Embedding 层是一个维度为 512×768
的查找表，其中第一行是第一个位置的向量表示，第二行是第二个位置上的向量表示，以此类推，最长一共有 512
个位置，注意这个向量表示和各个位置上的单词是无关的。例如输入为“Hello world”和“Hi there”，那么“Hello”和“Hi”是具有相同的
位置嵌入的，因为它们都是第一个单词。

### 合并

经过上面三个步骤，每个长度为 n 的 Input 会有三种不同的向量表示：

  1. 单词的向量表示，形状为 (1, n, 768)
  2. 成对输入序列的向量表示，形状为 (1, n, 768)
  3. 输入的位置属性表示，形状为 (1, n, 768)

将这三种表示进行求和，形状仍为 (1, n, 768)，然后将其传递给 BERT 的编码器层。

下面我们整理一下 BERT 模型的原理：

![](https://images.gitbook.cn/ac992980-966a-11ea-9fd5-332242a3cf46)

1\. 首先，模型的输入既可以是单个句子，也可以是成对的句子，输入的长度是 512。

2\. 对输入序列进行预处理：

  * Token Embedding
  * Sentence Embedding
  * Positional Embedding

还要对输入进行 15% 的随机掩码处理。这些预处理使我们不需要对 BERT 模型进行太大的改动，就可以将其灵活运用各种 NLP 任务。

3\. 将处理好的输入向量表示传递给 BERT 模型：

  * 开始时和 Transformer 一样，经过 Self-Attention 层和前馈神经网络层；
  * 得到的结果传递给下一个编码器层；
  * 经过所有编码器层后，每个位置都会得到一个长度为 hidden size 的向量，在 BERT Base 中这个值为 768；
  * 接着在分类任务中，我们只关注第一个位置即 [CLS] 对应的输出；
  * 然后将其传递给一个分类器模型，在论文中只是使用了单层神经网络就已经取得了非常不错的结果了；
  * 如果是多分类任务，只需要将分类器网络调整到输出相应个数的神经元即可，然后通过 softmax 获得概率。

本文中我们介绍了 BERT 的基础，在后面文章中会讲 BERT 的应用，可以帮助大家更好地理解 BERT 模型。

参考文献：

  * Jay Alammar, [_The Illustrated BERT, ELMo, and co. (How NLP Cracked Transfer Learning)_](http://jalammar.github.io/illustrated-bert/)

