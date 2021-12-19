前一篇我们学习了 seq2seq 模型，从它的模型结构中我们可以看到存在 **两个瓶颈：**

例如，当我们用 seq2seq 翻译一句话时，它的 Encoder
需要将原始句子中的主语，谓语，宾语，以及主谓宾之间的关系等信息都压缩到一个固定长度的上下文向量中，这个向量的长度通常只是 128 或者 256，
**如果输入数据很长时，就会有很多信息无法被压缩进这么短的向量中。**

另一个瓶颈是，这个上下文向量在 Decoder 中只是在最开始的时候传递一次，之后都要靠 Decoder 自己的 LSTM 单元的记忆能力去传递信息，
**这样当遇到长句子时，记忆能力也是有限的。**

于是为了改善这两个问题，Bahdanau 在 2015 年首次提出注意力模型。

在这个注意力模型中，Decoder 的每一个时间步都可以访问到 Encoder 的所有状态信息，这样记忆问题得以改善，而且在 Decoder
的不同时间步可以对 Encoder 中不同的时间步予以不同程度的关注，这样重要信息不会被淹没。

我们来更直观地对比看一下有和没有注意力机制的 seq2seq 模型有什么区别：

![](https://images.gitbook.cn/cc19b2e0-7a75-11ea-9687-397b36d0cdab)

在没有注意力的 seq2seq 中，上下文向量是 Encoder 最后的隐向量，在 Attention 中， **上下文向量是这些隐向量的加权平均。**

在没有注意力的 seq2seq 中，上下文向量只是在 Decoder 开始时输入进去，在 Attention 中，上下文向量会传递给 Decoder
的每一个时间步。

上面提到注意力机制中上下文向量是 Encoder 中隐向量的加权平均，这种加权思想就是用数学模型来模拟人类注意力的核心所在。

当我们在看一个图片时，视线会首先集中在某个区域，或者当我们读一句话时，首先映入眼帘的也是某几个重要的单词，我们不会一开始就关注整个图片或者整段话，而只是某个区域会引起我们最初的注意力。

那么用数学模型来实现的话，就是在一个输入数据中，需要给每个数据赋予不同的权重，因为它们对结果的重要程度是不一样的，其中
**比较重要的部分会被赋予较大的权重** 。注意力机制可以让模型在遇到很长的数据时能够抓住关键信息。

### 注意力机制的应用场景

注意力机制对 seq2seq 进行了这两处改进后，在很多领域中都有不错的效果。

**1\. 用于机器翻译**

> [_Neural machine translation by jointly learning to align and
> translate_](https://arxiv.org/abs/1409.0473)

比如将一种语言翻译成另一种语言时，注意力机制可以使输出序列的每一个单词去更多地关注输入序列中与其相关的单词，模型在生成下一个预测词时只需要聚焦在与此相关的信息上，而不是将整个输入数据都考虑进去。

下图是论文中一个法语翻译成英语的结果，纵轴代表法语，横轴代表英语，比如图 a 中 Area 重点关注的就是 la zone 这两个词：

![](https://images.gitbook.cn/ffab9a60-7a75-11ea-a711-0f902cb8434c)

**2\. 用于图像描述**

> [_Show, Attend and Tell: Neural Image Caption Generation with Visual
> Attention_](https://arxiv.org/abs/1502.03044)

注意力也可以用于计算机视觉领域，比如看到一张图片要输出一个英文句子来描述这张图时，注意力可以让输出的每个单词聚焦在图片中相应的部分。

例如在第四个图中，surfboard 这个词就重点关注图片中船帆所在的区域：

![](https://images.gitbook.cn/24026dd0-7a76-11ea-9792-81939fbf7f0c)

**3\. 用于推理蕴涵关系**

> [_Reasoning about Entailment with Neural
> Attention_](https://arxiv.org/abs/1509.06664)

这个任务是为了判断两个句子之间蕴含的关系。例如其中一句话在描述一个场景，另一句话是关于这个场景的假设，模型要判断出这个假设和场景之间是什么关系，可以是矛盾的、相关的、包含等关系。注意力机制在这里就可以将假设中的每个单词和场景中的单词进行关联。

例如第一个假设就和描述中 A rides camel 有较强的关联：

![](https://images.gitbook.cn/3a24eed0-7a76-11ea-9164-d34ec3ae1078)

**4\. 用于语音识别**

> [_Attention-Based Models for Speech
> Recognition_](https://arxiv.org/abs/1506.07503)

输入是一个语音片段，输出是相应的音素序列，注意力可以将输出序列中的每个音素与输入序列中的特定帧进行关联。

![](https://images.gitbook.cn/50182270-7a76-11ea-b2e6-fdc2f968c34f)

**5\. 用于文本摘要**

> [_A Neural Attention Model for Abstractive Sentence
> Summarization_](https://arxiv.org/abs/1509.00685)

输入是一个文本，输出是一句总结这段文字主旨的话，注意力可以将总结中的每个单词与输入文本中的特定单词联系起来。

比如输出的 against 重点关注在 combating terrorism 这两个词上。

![](https://images.gitbook.cn/67e238f0-7a76-11ea-a523-c58f0b4e76ea)

注意力机制用在这些任务上，不仅更高效准确，而且还能将输入和输出之间的对齐关系进行可视化，这样我们可以更清楚地知道模型在各个任务中到底都学到了些什么。

### 注意力模型

前面介绍了注意力机制的作用和应用，下面我们具体看看注意力的模型原理到底是怎样的。

![](https://images.gitbook.cn/7c8741b0-7a76-11ea-9164-d34ec3ae1078)

注意力模型的输入是 n 个状态 $y_1～y_n$，和一个上下文向量 c，模型的输出是一个向量 z，这个 z 是 $y_i$ 的某种形式的求和，通常是
$y_i$ 的加权平均，其中每个权重，是在给定上下文向量 c 的情况下，由每个 $y_i$ 与目标的相关程度计算而得的。

对照上图，更具体地讲，一个注意力模型通常可以看成是由三部分函数组成：

![](https://images.gitbook.cn/87fc1410-7a78-11ea-b2e6-fdc2f968c34f)

  * 第一层，得分函数：$e_{ij}$ 是要能够反映出 Encoder 的第 j 个词对 Decoder 的第 i 个词的影响程度。
  * 第二层，对齐函数：$\alpha$ 是由 softmax 函数计算而得，为了保证所有权重之和为 1。最终 $\alpha_{ij}$ 就可以代表 Encoder 的第 j 个词对 Decoder 的第 i 个词的影响大小。
  * 第三层，生成向量函数：$y_i$ 的加权平均。

### 几种经典的注意力模型

关于注意力模型的论文比较多，其中有两个比较常用的模型，分别来自 Bahdanau 和 Luong 的论文，这两个模型也分别叫做 Additive
attention 和 Multiplicative attention。

Bahdanau 和 Luong 模型的主要区别在于得分函数 $e_{ij}$ 的选择。

根据对齐函数的不同，又有 Global attention 和 Local attention 两种。

根据生成向量函数的不同，有 Hard attention 和 Soft attention 之分。

![](https://images.gitbook.cn/58da4600-7a7a-11ea-a711-0f902cb8434c)

**1.** 注意力机制最早由 Bahdanau 在论文 [ _Neural machine translation by jointly learning
to align and translate_ ] (https://arxiv.org/pdf/1409.0473.pdf) 中提出。

![](https://images.gitbook.cn/72004300-7a7a-11ea-9164-d34ec3ae1078)

**2.** Luong 在 [_Effective Approaches to Attention-based Neural Machine
Translation_](https://arxiv.org/pdf/1508.04025.pdf)
中提出了几种更有效的方法，而且在这篇论文中，作者提出了 Global Attention 和 Local Attention 的概念。

![](https://images.gitbook.cn/85de5240-7a7a-11ea-8dae-453849991cc6)

![](https://images.gitbook.cn/937c5820-7a7a-11ea-a523-c58f0b4e76ea)

**Global Attention：**

在计算上下文向量的权重时，会用到所有的隐藏层状态，和 soft attention 类似。

这样的话，权重 $\alpha_t$ 的长度是可变的，因为它要和 Encoder 的输入句子的长度一样。

权重向量 $\alpha_t$ 由 Decoder 的隐状态 $h_t$ 和 Encoder 的隐状态 $h_s$ 计算而得，其中 score
函数有三种形式可以选择。

**Local Attention：**

可以提高 Global Attention 的效率，因为在 Global 中，每一次计算都要考虑 Encoder 的所有隐状态，这样会消耗不必要的资源。而且
Local 还能让 Hard 变得可微，它是 Soft Attention 和 Hard Attention 的结合形式，在预测每个 Decoder
端当前的词时，先为这个词预测出在 Encoder 端与它相关的位置 $p_t$ ，再根据这个 $p_t$ 选择一个窗口，上下文向量 $c_t$
的计算是根据这个窗口中的隐状态计算的。

**3.** Xu 在 [_Show, Attend and Tell: Neural Image Caption Generation with
Visual Attention_](http://proceedings.mlr.press/v37/xuc15.pdf)
中将注意力机制应用到图像领域，并且在这篇论文中提出了 Soft Attention 和 Hard Attention 的概念。

![](https://images.gitbook.cn/25f46080-7a7b-11ea-9792-81939fbf7f0c)

![](https://images.gitbook.cn/34378000-7a7b-11ea-8dae-453849991cc6)

**Soft Attention** 本质和 Bahdanau
的注意力机制是一样的，这种机制会关注输入数据的所有区域，模型函数是光滑可微的，可以直接用反向传播进行训练，所以应用比较广泛。

**Hard Attention** 每次只关注输入数据的某个部分，这样可以减少计算量，但是模型函数不可微，训练时需要用比较复杂的方法。Luong
属于这一种注意力模型。

### Bahdanau attention 和 Luong attention

在前面我们对几种注意力模型的分类有了整体的了解，下面我们来详细学习其中两个常用的注意力模型：Bahdanau attention 和 Luong
attention。

#### **Bahdanau attention**

> [_Neural machine translation by jointly learning to align and
> translate_](https://arxiv.org/pdf/1409.0473.pdf)

在文章开头我们有个图片是对比 Attention 和 seq2seq 的区别：

> 在 seq2seq 中，上下文向量 c 是 Encoder 的最后一个状态，并且只输入给 Decoder 的第一步。
>
> 在带有 Bahdanau attention 的模型中，Decoder 每一步的输入都增加了相应的 $c_i$，并且 $c_i$ 还是 Encoder
> 的所有时间步的加权平均。

下面详细看一下关键步骤的计算：

![](https://images.gitbook.cn/5f74fd10-7a7b-11ea-a711-0f902cb8434c)

符号：

  * Encoder 的隐藏状态记为 $h_i$
  * 上下文向量为 $c_i$
  * Decoder 的隐藏状态是 $s_i$
  * Decoder 每一步的输出为 $y_i$

![](https://images.gitbook.cn/7c5fca40-7a7b-11ea-b348-9df247d9e896)

上图右侧部分是模型中相应的公式：

  * 在 Decoder 的第 i 步所需要的 $c_i$，它是 Encoder 的所有时间步的隐藏状态 $h_j$ 的加权平均。
  * 权重 $\alpha_{ij}$ 表示的是在预测第 i 个词时，Encoder 的第 j 个隐藏状态有多么重要。$\alpha_{ij}$ 是一个 $e_{ij}$ 的 softmax 值。
  * $e_{ij}$ 的意思是，在计算 Decoder 的 $s_{i}$ 时，$s_{i-1}$ 和 Encoder 的 $h_j$ 贡献了多少能量。

从公式可以看出，$e_{ij}$ 其实是由一个多层感知器计算出来的，感知器的输入是 $s_{i-1}$ 和 $h_j$，权重为 $V、W、U$。

#### **Luong attention**

> [Effective Approaches to Attention-based Neural Machine
> Translation](https://arxiv.org/pdf/1508.04025.pdf)

整体来看，Luong 模型的大体结构和 Bahdanau 模型是类似的：

![](https://images.gitbook.cn/a2d75ad0-7a7b-11ea-b2e6-fdc2f968c34f)

  1. Bahdanau 的 Encoder 使用双向 RNN，Luong 的 Encoder 使用的是单向多层 RNN，Luong 的 Decoder 也是用的单向多层 RNN。
  2. Bahdanau 在计算输出时，需要将双向 RNN 前向和反向两个方向的隐状态连接起来。即在计算 $h_t$ 时，用的是 Decoder 的 $t-1$ 时刻的隐状态 $h_{t-1}$，随后计算注意力分数和上下文向量，这个上下文向量会和 Decoder 中 $t-1$ 的隐状态相连，所以在作用 softmax 函数之前，这个连接向量会进入一个 GRU。
  3. Luong 的输出用的是 Encoder 和 Decoder 最上面的一层的隐状态。即 Luong 在计算 $h_t$ 时，首先得到 Decoder 的 $h_t$，然后计算注意力分数，接着计算上下文向量，再用这个上下文向量与 Decoder 的隐状态连接，接着进行预测。
  4. 关于得分函数，Luong 在 Bahdanau 的基础上尝试了三种形式的得分函数，其中 concat 形式和 Bahdanau 是一样的。
  5. Bahdanau 的对齐函数是最基本的 softmax，Luong 也尝试了多种对齐函数，如 local-p、local-m。
  6. 关于上下文向量计算方式，Bahdanau 和 Luong 是一样的。

### 用 Keras 建立一个 Bahdanau 注意力模型

接下来我们动手用 Keras 构建一个 Bahdanau 注意力模型，加深对模型结构的理解。大家可以对照下面这个图来看代码是如何实现的：

![](https://images.gitbook.cn/c81c85e0-7a7b-11ea-8940-6df1558f5aa2)

我们的目的是想在 Keras 中编写一个 Attention 层，那么在 class AttentionDecoder 中需要实现下面几个方法：

  * `__init__`：初始化权重，激活函数，正则化算法等
  * `build(input_shape)`：在这里定义模型中用到的权重，
  * `call(x)`：在这里编写层的功能逻辑，它的第一个参数是输入张量，
  * `compute_output_shape(input_shape)`：在这里定义张量的形状变化逻辑，对于任意给定的输入，Keras 就可以自动推断各层输出的形状。

接下来详细介绍各方法。

在 build 中，我们定义了很多权重参数，用 self.add_weight 对权重进行初始化并且可训练，下标为 a 的是上下文向量的参数， 下标为
r、z、p 分别是 reset gate、update gate、proposal 的参数，下标为 o 的用于生成最终的预测结果。

此外，get_config 用来将模型加载到内存中，step 中会定义模型中的所有公式运算。

我们的目的是要实现上图中的注意力模型，模型的主体部分就是图片右侧这几个公式，公式的运算都在 step
方法中定义，在代码中有注释是第几个公式，在公式中出现的所有权重 W 都在方法 build 中进行定义。

更详细的解释请大家看代码注释：

    
    
    import tensorflow as tf
    from keras import backend as K
    from keras import regularizers, constraints, initializers, activations
    from keras.layers.recurrent import Recurrent          
    from keras.engine import InputSpec
    
    def time_distributed_dense(x, w, b=None, dropout=None,
                               input_dim=None, output_dim=None, timesteps=None):
        if not input_dim:
            input_dim = K.shape(x)[2]
        if not timesteps:
            timesteps = K.shape(x)[1]
        if not output_dim:
            output_dim = K.shape(w)[1]
    
        if dropout:
            # 在每个时间步采用同样的 dropout
            ones = K.ones_like(K.reshape(x[:, 0, :], (-1, input_dim)))
            dropout_matrix = K.dropout(ones, dropout)
            expanded_dropout_matrix = K.repeat(dropout_matrix, timesteps)
            x *= expanded_dropout_matrix
    
        x = K.reshape(x, (-1, input_dim))
    
        x = K.dot(x, w)
        if b:
            x = x + b
    
        x = K.reshape(x, (-1, timesteps, output_dim))
        return x
    
    tfPrint = lambda d, T: tf.Print(input_=T, data=[T, tf.shape(T)], message=d)
    
    class AttentionDecoder(Recurrent):
    
        # 在 __init__ 初始化权重，激活函数，正则化算法等
        def __init__(self, units, output_dim,
                     activation='tanh',
                     return_probabilities=False,
                     name='AttentionDecoder',
                     kernel_initializer='glorot_uniform',
                     recurrent_initializer='orthogonal',
                     bias_initializer='zeros',
                     kernel_regularizer=None,
                     bias_regularizer=None,
                     activity_regularizer=None,
                     kernel_constraint=None,
                     bias_constraint=None,
                     **kwargs):
    
            self.units = units                        # 隐状态的维度
            self.output_dim = output_dim            # 输出序列的标签个数
            self.return_probabilities = return_probabilities
            self.activation = activations.get(activation)
            self.kernel_initializer = initializers.get(kernel_initializer)
            self.recurrent_initializer = initializers.get(recurrent_initializer)
            self.bias_initializer = initializers.get(bias_initializer)
    
            self.kernel_regularizer = regularizers.get(kernel_regularizer)
            self.recurrent_regularizer = regularizers.get(kernel_regularizer)
            self.bias_regularizer = regularizers.get(bias_regularizer)
            self.activity_regularizer = regularizers.get(activity_regularizer)
    
            self.kernel_constraint = constraints.get(kernel_constraint)
            self.recurrent_constraint = constraints.get(kernel_constraint)
            self.bias_constraint = constraints.get(bias_constraint)
    
            super(AttentionDecoder, self).__init__(**kwargs)
            self.name = name
            self.return_sequences = True  # must return sequences
    
        # build(input_shape): 在这里定义模型中用到的权重
        def build(self, input_shape):
    
            self.batch_size, self.timesteps, self.input_dim = input_shape
    
            if self.stateful:
                super(AttentionDecoder, self).reset_states()
    
            self.states = [None, None]  # y, s
    
            """
                用来建立上下文向量
            """
    
            self.V_a = self.add_weight(shape=(self.units,),
                                       name='V_a',
                                       initializer=self.kernel_initializer,
                                       regularizer=self.kernel_regularizer,
                                       constraint=self.kernel_constraint)
            self.W_a = self.add_weight(shape=(self.units, self.units),
                                       name='W_a',
                                       initializer=self.kernel_initializer,
                                       regularizer=self.kernel_regularizer,
                                       constraint=self.kernel_constraint)
            self.U_a = self.add_weight(shape=(self.input_dim, self.units),
                                       name='U_a',
                                       initializer=self.kernel_initializer,
                                       regularizer=self.kernel_regularizer,
                                       constraint=self.kernel_constraint)
            self.b_a = self.add_weight(shape=(self.units,),
                                       name='b_a',
                                       initializer=self.bias_initializer,
                                       regularizer=self.bias_regularizer,
                                       constraint=self.bias_constraint)
            """
                用来建立 reset 门
            """
            self.C_r = self.add_weight(shape=(self.input_dim, self.units),
                                       name='C_r',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
            self.U_r = self.add_weight(shape=(self.units, self.units),
                                       name='U_r',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
            self.W_r = self.add_weight(shape=(self.output_dim, self.units),
                                       name='W_r',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
            self.b_r = self.add_weight(shape=(self.units, ),
                                       name='b_r',
                                       initializer=self.bias_initializer,
                                       regularizer=self.bias_regularizer,
                                       constraint=self.bias_constraint)
    
            """
                用来建立 update 门
            """
            self.C_z = self.add_weight(shape=(self.input_dim, self.units),
                                       name='C_z',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
            self.U_z = self.add_weight(shape=(self.units, self.units),
                                       name='U_z',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
            self.W_z = self.add_weight(shape=(self.output_dim, self.units),
                                       name='W_z',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
            self.b_z = self.add_weight(shape=(self.units, ),
                                       name='b_z',
                                       initializer=self.bias_initializer,
                                       regularizer=self.bias_regularizer,
                                       constraint=self.bias_constraint)
            """
                用来建立 proposal
            """
            self.C_p = self.add_weight(shape=(self.input_dim, self.units),
                                       name='C_p',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
            self.U_p = self.add_weight(shape=(self.units, self.units),
                                       name='U_p',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
            self.W_p = self.add_weight(shape=(self.output_dim, self.units),
                                       name='W_p',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
            self.b_p = self.add_weight(shape=(self.units, ),
                                       name='b_p',
                                       initializer=self.bias_initializer,
                                       regularizer=self.bias_regularizer,
                                       constraint=self.bias_constraint)
            """
                用来生成预测结果
            """
            self.C_o = self.add_weight(shape=(self.input_dim, self.output_dim),
                                       name='C_o',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
            self.U_o = self.add_weight(shape=(self.units, self.output_dim),
                                       name='U_o',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
            self.W_o = self.add_weight(shape=(self.output_dim, self.output_dim),
                                       name='W_o',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
            self.b_o = self.add_weight(shape=(self.output_dim, ),
                                       name='b_o',
                                       initializer=self.bias_initializer,
                                       regularizer=self.bias_regularizer,
                                       constraint=self.bias_constraint)
    
            # 建立初始状态
            self.W_s = self.add_weight(shape=(self.input_dim, self.units),
                                       name='W_s',
                                       initializer=self.recurrent_initializer,
                                       regularizer=self.recurrent_regularizer,
                                       constraint=self.recurrent_constraint)
    
            self.input_spec = [
                InputSpec(shape=(self.batch_size, self.timesteps, self.input_dim))]
            self.built = True
    
        # call(x): 在这里编写层的功能逻辑，它的第一个参数是输入张量
        def call(self, x):
            # 这一句是用来存储整个序列，以便后面在每一个时间步时使用
            self.x_seq = x
    
            # 这一步是在计算 e j t 中的 U a h j，
            # 用 time_distributed_dense 对编码序列的所有元素执行这个计算，
            # 放在这里因为它不依赖前面的步骤，可以节省计算时间。
            self._uxpb = time_distributed_dense(self.x_seq, self.U_a, b=self.b_a,
                                                 input_dim=self.input_dim,
                                                 timesteps=self.timesteps,
                                                 output_dim=self.units)
    
            return super(AttentionDecoder, self).call(x)
    
        def get_initial_state(self, inputs):
            print('inputs shape:', inputs.get_shape())
    
            # 在第一个时间步得到初始状态 s0.
            s0 = activations.tanh(K.dot(inputs[:, 0], self.W_s))
    
            # 用 keras.layers.recurrent 初始化向量 (batchsize, output_dim)
            y0 = K.zeros_like(inputs)          # (samples, timesteps, input_dims)
            y0 = K.sum(y0, axis=(1, 2))      # (samples, )
            y0 = K.expand_dims(y0)          # (samples, 1)
            y0 = K.tile(y0, [1, self.output_dim])
    
            return [y0, s0]
    
        # step 是模型建立的核心部分
        def step(self, x, states):
    
            # 首先取 t-1 时刻的输出和状态
            ytm, stm = states
    
            # equation 1
    
            # 因为计算是向量化的，需要将隐状态重复多次，使它的长度等于输入序列的长度
            _stm = K.repeat(stm, self.timesteps)
    
            # 权重矩阵与隐状态相乘
            _Wxstm = K.dot(_stm, self.W_a)
    
            # 计算注意力概率
            et = K.dot(activations.tanh(_Wxstm + self._uxpb),
                       K.expand_dims(self.V_a))
    
            # equation 2 
    
            at = K.exp(et)
            at_sum = K.sum(at, axis=1)
            at_sum_repeated = K.repeat(at_sum, self.timesteps)        # 用 repeat 使每个时间步除以相应的和
            at /= at_sum_repeated                                      # 此时向量大小 (batchsize, timesteps, 1)
    
            # equation 3
    
            # 计算上下文向量
            # self.x_seq 和 at 的维度中有 batch dimension，需要用 batch_dot 避开在这个维度上的乘法
            context = K.squeeze(K.batch_dot(at, self.x_seq, axes=1), axis=1)
    
            # equation 4  (reset gate)
    
            # 计算新的隐状态
            # 首先计算 reset 门
            rt = activations.sigmoid(
                K.dot(ytm, self.W_r)
                + K.dot(stm, self.U_r)
                + K.dot(context, self.C_r)
                + self.b_r)
    
            # equation 5 (update gate)
    
            # 计算 z 门
            zt = activations.sigmoid(
                K.dot(ytm, self.W_z)
                + K.dot(stm, self.U_z)
                + K.dot(context, self.C_z)
                + self.b_z)
    
            # equation 6 (proposal state)
    
            # 计算 proposal 隐状态
            s_tp = activations.tanh(
                K.dot(ytm, self.W_p)
                + K.dot((rt * stm), self.U_p)
                + K.dot(context, self.C_p)
                + self.b_p)
    
            # equation 7 (new hidden states)
    
            # 得到新的隐状态
            st = (1-zt)*stm + zt * s_tp
    
            # equation 8 
    
            yt = activations.softmax(
                K.dot(ytm, self.W_o)
                + K.dot(stm, self.U_o)
                + K.dot(context, self.C_o)
                + self.b_o)
    
            if self.return_probabilities:
                return at, [yt, st]
            else:
                return yt, [yt, st]
    
        # compute_output_shape(input_shape): 在这里定义张量的形状变化逻辑，对于任意给定的输入，Keras 可以自动推断各层输出的形状。
        def compute_output_shape(self, input_shape):
            if self.return_probabilities:
                return (None, self.timesteps, self.timesteps)
            else:
                return (None, self.timesteps, self.output_dim)
    
        def get_config(self):
            config = {
                'output_dim': self.output_dim,
                'units': self.units,
                'return_probabilities': self.return_probabilities
            }
            base_config = super(AttentionDecoder, self).get_config()
            return dict(list(base_config.items()) + list(config.items()))
    
    # 应用注意力机制，建立模型：
    if __name__ == '__main__':
        from keras.layers import Input, LSTM
        from keras.models import Model
        from keras.layers.wrappers import Bidirectional
    
        i = Input(shape=(100,104), dtype='float32')
        enc = Bidirectional(LSTM(64, return_sequences=True), merge_mode='concat')(i)
        dec = AttentionDecoder(32, 4)(enc)
        model = Model(inputs=i, outputs=dec)
        model.summary()
    

**面试题：**

  * 什么是注意力机制？
  * 为什么要应用注意力机制？
  * 注意力机制有哪几种类型？

* * *

参考文献：

  * Jason Brownlee, [_Attention in Long Short-Term Memory Recurrent Neural Networks_](https://machinelearningmastery.com/attention-long-short-term-memory-recurrent-neural-networks/)

