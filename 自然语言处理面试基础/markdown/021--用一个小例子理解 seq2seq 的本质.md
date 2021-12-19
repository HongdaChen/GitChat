我们知道序列预测有 one-to-one、many-to-one、many-to-many、one-to-many
几种类型。在前面的文章中，我们做过的任务大多是将一个序列映射为单个输出的问题，比如 NER 是预测单个单词的标签，语言模型是一个一个预测下一个单词的问题。

现在我们要看输出是一个序列的情况，也就是 to-many，这类问题有个挑战是它的输入和输出都是序列，而且序列的长度是不固定的，为了应对这种挑战，于是有了
seq2seq 模型。

### 什么是 seq2seq 模型

**seq2seq** 即 Sequence-to-sequence 模型，通常指 Encoder Decoder LSTM
模型，它由两个循环神经网络（可以是 RNN、LSTM、GRU 等）组成：

  * **一个 Encoder 编码器** ，输入为文本的序列，输出为一个固定维度的“上下文向量”。
  * **一个 Decoder 解码器** ，输入为这个“上下文向量”，输出为目标序列。

其中 Encoder 可以是双向的循环神经网络，这样的话，有一串 LSTM
是正向地读取输入序列，还有一串是反向地读取输入序列。因为是双向，我们会得到两个隐藏层状态，一个来自正向，一个来自反向，通过双向结构可以对文本的信息有更加全面的掌握。

Encoder 还可以增加层数，例如可以是多层双向 LSTM，每层的输出是下一层的输入。在一些复杂的任务中，为了得到更好的结果就可以用多层结构。

而且任务不同时，Encoder 的结构也可以选择不同的神经网络，比如做自然语言处理任务时用 LSTM，做图像的任务时可以用 CNN。

模型的 Decoder 部分是根据 Encoder 输出的信息来输出最终的目标序列，也可以用不同的结构。下面我们会介绍两种最常用的 seq2seq
模型结构，它们的 Decoder 部分就有所不同。

### seq2seq 模型原理

比较常用的 seq2seq 模型有两种，一种是来自 Cho et al.(2014) 的论文：

> [Learning Phrase Representations using RNN Encoder–Decoder for Statistical
> Machine Translation](https://arxiv.org/pdf/1406.1078.pdf)

模型结构为：

![](https://images.gitbook.cn/1f263d90-75bd-11ea-b264-6326f7cc0e82)

其中，Encoder 的数学表示为：

$$\begin{aligned}h_t &= tanh(W[h_{t-1}, x_t] + b) \\\o_t &= softmax(Vh_t +
r)\end{aligned}$$

Encoder 输出的上下文向量为：

$$c = tanh(U h_T)$$

Decoder 的表示为：

$$\begin{aligned}h_t &= tanh(W[h_{t-1}, y_{t-1}, c] + b) \\\o_t &=
softmax(Vh_t + r)\end{aligned}$$

从结构图中可以看到，Decoder 接收来自 Encoder 的上下文向量 c，这个 **c 会传递到 Decoder 中的每一个时间步** 。

另一种更加常用的 seq2seq 模型来自 Sutskever et al.(2014) 的论文：

> [Sequence to Sequence Learning with Neural
> Networks](https://arxiv.org/pdf/1409.3215.pdf)

在这篇论文中，模型的结构是这样的：

![](https://images.gitbook.cn/f93528d0-75c1-11ea-aa63-49ba907907a9)

Encoder 的数学表示为：

$$\begin{aligned}h_t &= tanh(W[h_{t-1}, x_t] + b) \\\o_t &= softmax(Vh_t +
r)\end{aligned}$$

Encoder 输出的上下文向量为：

$$c = h_T$$

上下文向量直接用的隐藏状态，没有额外作用 tanh 函数。

Decoder 的表示为：

$$\begin{aligned}h_t &= tanh(W[h_{t-1}, y_{t-1}] + b) \\\o_t &= softmax(Vh_t +
r)\end{aligned}$$

其中

$$h_0 = c$$

从上面两个模型的结构图和数学表示可以看出，Sutskever 和 Cho 的 seq2seq 模型的区别是： **上下文向量 c 直接作为 Decoder
的隐藏状态输入，并且只在最开始时传递进去，而不是在每一步都传递一次** 。

更详细地来看一下 Sutskever 的模型原理：

![](https://images.gitbook.cn/24d0a460-75c2-11ea-9613-4f950122a537)

当我们输入一个序列，例如“how are you”，每个单词被表示成词向量，输入给 Encoder，Encoder 的每个时间步读取一个单词，取
Encoder 最后一层循环层的最后一个隐藏状态 c，这个 c 就捕捉了输入序列的信息。

![](https://images.gitbook.cn/3dcd2060-75c2-11ea-a3f1-b7b0a6636d71)

Decoder 的初始输入为隐藏层状态 $c$ 和开始符号 $w_{sos}$，再计算隐藏层状态 $h_0$，根据模型公式计算
$s_0$，它的维度和词汇表大小一样，接着作用 Softmax 得到概率
$p_0$，它的含义就是词汇表中每个单词是当前预测结果的概率，最后选出概率值最大的位置，它所对应的单词就作为当前预测结果 $i_0$：

$$\begin{aligned}h_0 &= \operatorname{LSTM}\left(c, w_{sos}\right)\\\s_0 &= V
h_0 + r\\\p_0 &= \operatorname{softmax}(s_0)\\\i_0 &=
\operatorname{argmax}(p_0)\\\\\end{aligned}$$

然后进入下一个 LSTM 单元，输入为 $h_0$ 和 $i_0$ 的词向量，经历和前一步同样的过程，最后得到 $i_1$，以此类推，Decoder
输出的每一步代表一个预测单词，当预测结果为 $<eos>$ 时 Decoder 停止预测。

$$\begin{aligned}h_1 &= \operatorname{LSTM}\left(h_0, w_{i_0}\right)\\\s_1 &=
V h_1 + r\\\p_1 &= \operatorname{softmax}(s_1)\\\i_1 &=
\operatorname{argmax}(p_1)\end{aligned}$$

### seq2seq 的优点

**1\. 输入序列和输出序列的长度可以不固定**

seq2seq 和普通 RNN 的区别是，它处理的是输出为一个序列的任务，而且序列的长度还可以是不固定的。如果用普通的
RNN，我们会设置一个句子的最大长度参数，超出规定长度的句子需要将其截断，没达到规定长度的句子需要做 padding 补充，但是用 seq2seq
就可以接收不定长的输入序列和输出不定长的序列。

**2\. Encoder 处理完整个输入句子后，Decoder 再进行预测**

如果用普通的 RNN
来做对话任务，模型是立刻就得到答案的，即输入第一个单词后，就会输出相应的预测单词，输出的第一个单词只依赖第一个输入单词，而且每个时刻的输出只依赖这一时刻之前的输入，而不能接收到这一时刻之后的信息，就相当于问题还没有问完，就急着做出回答了，这样的话答案肯定是不太准的。而
seq2seq 中就可以等 Encoder 处理完整个输入句子后，Decoder 再进行预测，这样 Decoder 可以考虑到整个输入的信息。

### seq2seq 有哪些应用领域

上面提到的 seq2seq 这些特性，使它具有很广泛的应用，例如：

**机器翻译**

[Sutskever et al., 2014](https://arxiv.org/abs/1409.3215) 的论文中就显示出了 seq2seq
模型在机器翻译任务中的优秀表现。在这篇论文中使用了八层 LSTM 模型达到了不错的效果，现在 seq2seq 结构已经成为了机器翻译的标准方法。

当然也存在一些问题，虽然模型在机器翻译任务中的表现不错，但是运算量太大了，训练起来比较困难。

**自动回复邮件**

[Kannan et al.(2016)](https://arxiv.org/pdf/1606.04870.pdf) 的论文中提到了 Google
邮箱的一个功能，就是接收到一些长邮件时可以自动生成“好的，我会做的”、“好的，下周三见”等回复。这个功能的核心技术就是用大量邮件及其回复的数据来训练
seq2seq 模型，一个 LSTM Encoder 读取邮件，一个 LSTM Decoder 产生一个合适的答案。

还可以用来做聊天机器人，文本摘要，图像字幕，都是输入序列后，得到一个序列。其实还有很多任务都是输入是一个序列，输出也是一个序列的，都可以用 Encoder
Decoder 的结构来做。比如 [Gillick et al.(2016)](https://aclweb.org/anthology/N16-1155)
用 seq2seq 来做 POS 和 NER 任务，[Vinyals et al.
(2014)](https://arxiv.org/abs/1412.7449v1) 用 seq2seq
来做句法分析。但这类模型对于这些任务来说也许并不是最好的选择，有一些特定任务会有更合适的模型。

### seq2seq 的改进

在应用处我们知道了 seq2seq 虽然很强大但是也存在一些不足，下面两种技术可以用来提高性能。

**Attention**

在 seq2seq 中，Decoder
的初始输入只有一个上下文向量，而这个向量需要包含输入文本的所有信息，当输入内容很多时，将所有信息浓缩成一个向量就会变得很困难，难免会损失很多信息，即
Decoder 在解码时就会忽略掉一些细节。为了应对这个问题，就有了注意力机制，这个机制使得 Decoder
可以有选择地关注输入的序列。例如在翻译一个文本序列时，很多时候没必要把所有内容全部都看完才能进行翻译，也许只要看其中一部分，就可以把这部分翻译出来了。关于注意力机制，我们会在后面的文章中更详细地讲解。

**Beam Search**

之前提到过 Beam Search，在这里也同样可以使用。在 Decoder 输出结果时，会选择概率最高的那个词，例如我们要输出长度为 10
的序列，在词汇表中一共有 10000 个词，那么输出结果会有 10000^10
种可能性，如果用穷举法几乎是不可能完成的。所以如果每次只取概率最高的那个词的话，计算开销会显著下降，不过仍然有问题，就是无法保证全局最优解。

这时就可以用波束搜索的方法，即每次选择概率最高的 k 个词，由它们再生成下一层的候选词，从中选出概率最高的 k
个词，不断重复这个步骤。然后将所有包含特殊符号 `<eos>` 的序列选择出来，并且将 `<eos>` 后面的输出都删掉，每个序列只保留 `<eos>`
之前的内容。再在这些序列中选择分数最高的那一个作为最后的输出结果。不过使用波束搜索时需要定义一个最长的输出序列长度，否则就可能停不下来。

### 用 Keras 实现 seq2seq

下面我们将通过一个最简单的例子来看如何应用
seq2seq，这个例子就是建立模型来计算两个数字的和。求和这个例子看起来简直太简单了，但正是从简单的例子中才能更深刻地体会到模型的本质和威力。

#### **1\. 加载所需的库**

    
    
    from random import seed
    from random import randint
    from numpy import array
    from math import ceil
    from math import log10
    from math import sqrt
    from numpy import argmax
    from keras.models import Sequential
    from keras.layers import Dense
    from keras.layers import LSTM
    from keras.layers import TimeDistributed
    from keras.layers import RepeatVector
    

#### **2\. 生成数据**

在训练模型前我们会设置这几个超参数的数值，如：

    
    
    n_samples = 1000
    n_numbers = 2
    largest = 10
    

random_sum_pairs 就是用来随机生成 1000 组数据，每一组的 X 是 2 个在 1~10 之间的数字，每一组的 y 是 X
中所有数字的求和。

例如 X[0]=[7, 7]，y[0]=14。

    
    
    def random_sum_pairs(n_examples, n_numbers, largest):
        X, y = list(), list()
        for i in range(n_examples):
            in_pattern = [randint(1,largest) for _ in range(n_numbers)]
            out_pattern = sum(in_pattern)
            X.append(in_pattern)
            y.append(out_pattern)
        return X, y
    

#### **3\. 数据预处理**

首先将 X 和 y 转化成字符串，X[0] 就变成了 `'7+7'`，y[0] 就变成了 `'14'` 的形式。

    
    
    def to_string(X, y, n_numbers, largest):
        max_length = n_numbers * ceil(log10(largest+1)) + n_numbers - 1
        Xstr = list()
        for pattern in X:
            strp = '+'.join([str(n) for n in pattern])
            strp = ''.join([' ' for _ in range(max_length-len(strp))]) + strp
            Xstr.append(strp)
        max_length = ceil(log10(n_numbers * (largest+1)))
        ystr = list()
        for pattern in y:
            strp = str(pattern)
            strp = ''.join([' ' for _ in range(max_length-len(strp))]) + strp
            ystr.append(strp)
        return Xstr, ystr
    

然后将 X 和 y 构造成带有加号的格式，首先需要定义

    
    
    alphabet = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9', '+', ' ']
    

这样 X[0] 为 [11, 11, 7, 10, 7]，其中 10 在 alphabet 中代表加号，11 代表空格，y[0] 就变成了 [1, 4]。

这样处理后，输入和输出都被转化为字符串，让模型自己去识别各个字符的含义，自己去识别各个整数和加号的含义。

问题也就变成了一个序列到序列的问题了，输入和输出都是序列，而且还是带着顺序的，字符之间的顺序是不能调换的了。否则如果按照整数的输入输出的话，就是能够调换整数的顺序，对模型结果没有影响，而这里却不可以。

    
    
    def integer_encode(X, y, alphabet):
        char_to_int = dict((c, i) for i, c in enumerate(alphabet))
        Xenc = list()
        for pattern in X:
            integer_encoded = [char_to_int[char] for char in pattern]
            Xenc.append(integer_encoded)
        yenc = list()
        for pattern in y:
            integer_encoded = [char_to_int[char] for char in pattern]
            yenc.append(integer_encoded)
        return Xenc, yenc
    

还需要对数据进行 one-hot 编码，此时，X[0] 变成：

    
    
    [[0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
     [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1],
     [0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0],
     [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0],
     [0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0]]
    

y[0] 变成：

    
    
    [[0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], 
     [0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0]]
    

于是这个求和问题变成一个分类问题，输出的类别有 12 种。

    
    
    def one_hot_encode(X, y, max_int):
        Xenc = list()
        for seq in X:
            pattern = list()
            for index in seq:
                vector = [0 for _ in range(max_int)]
                vector[index] = 1
                pattern.append(vector)
            Xenc.append(pattern)
        yenc = list()
        for seq in y:
            pattern = list()
            for index in seq:
                vector = [0 for _ in range(max_int)]
                vector[index] = 1
                pattern.append(vector)
            yenc.append(pattern)
        return Xenc, yenc
    

在构造数据时，直接按照上面的顺序处理 X 和 y：

    
    
    def generate_data(n_samples, n_numbers, largest, alphabet):
        X, y = random_sum_pairs(n_samples, n_numbers, largest)
        X, y = to_string(X, y, n_numbers, largest)
        X, y = integer_encode(X, y, alphabet)
        X, y = one_hot_encode(X, y, len(alphabet))
        X, y = array(X), array(y)
        return X, y
    

最后需要一个辅助函数，将预测结果从向量转化成数字。用 argmax() 将 one-hot 向量转化为整数编码值，再在 alphabet
找到整数对应的字符。

    
    
    def invert(seq, alphabet):
        int_to_char = dict((i, c) for i, c in enumerate(alphabet))
        strings = list()
        for pattern in seq:
            string = int_to_char[argmax(pattern)]
            strings.append(string)
        return ''.join(strings)
    

#### **4\. 训练模型**

首先定义超参数，模型处理这个求和的问题，是把数据当作字符串来处理的，所以它同样也要每个字符的向量表示的长度 n_chars，X 的最大长度
n_in_seq_length，y 的最大长度 n_out_seq_length。

    
    
    seed(1)
    n_samples = 1000        # 生成几组数据
    n_numbers = 2            # 每组是几个数字相加
    largest = 10            # 每个数字随机生成的上界
    alphabet = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9', '+', ' ']            # 用于将数字映射成数字时
    n_chars = len(alphabet)                                                            # 每个字符的向量表示的长度
    n_in_seq_length = n_numbers * ceil(log10(largest+1)) + n_numbers - 1            # X 的最大长度
    n_out_seq_length = ceil(log10(n_numbers * (largest+1)))                            # y 的最大长度
    n_batch = 10
    n_epoch = 30
    

**建立模型**

这里的 Encoder 是一层 LSTM，它的输出是一个固定大小的向量，在 Encoder 中最后一个 LSTM 层中定义的维度就是这个向量的大小。
每个时间步读取一个 one-hot 向量，它需要学习出整个输入序列中各个时间步之间的关系，并且将输入的信息浓缩成一个向量。

`input_shape=(5, 12)`，因为我们定了整数位于 100 以内，所以两个整数和一个加号一共是 5 个字符，输入序列中每个字符有 12
种可能情况，所以每个向量包括 12 个特征。

然后用 RepeatVector 将 Encoder 和 Decoder 连起来，因为 Encoder 直接输出的是一个 2D 的矩阵，但 Decoder
需要接收一个 3D 的输入 `[samples, time steps, features]`，所以需要将 2D 的输出重复多次变成 3D 的输入。

Decoder 部分也可以由一层或多层 LSTM 组成，这里也是用一层。

模型的最后用一个 Dense 层输出结果，将 Dense 层封装在 TimeDistributed 中，这样在输出序列的每个时间步都可以用相同的权重。

因为是一个分类问题，所以损失函数选择 categorical_crossentropy。

    
    
    model = Sequential()
    model.add(LSTM(100, input_shape=(n_in_seq_length, n_chars)))
    model.add(RepeatVector(n_out_seq_length))
    model.add(LSTM(50, return_sequences=True))
    model.add(TimeDistributed(Dense(n_chars, activation='softmax')))
    model.compile(loss='categorical_crossentropy', optimizer='adam', metrics=['accuracy'])
    print(model.summary())
    

![](https://images.gitbook.cn/0e117bf0-75c7-11ea-be15-27e75266f38f)

**训练模型**

    
    
    for i in range(n_epoch):
        X, y = generate_data(n_samples, n_numbers, largest, alphabet)
        print(i)
        model.fit(X, y, epochs=1, batch_size=n_batch)
    
    
    
    

#### **5\. 模型预测**

模型训练结束后，看一下预测结果如何。生成一批训练数据，进行预测：

    
    
    X, y = generate_data(n_samples, n_numbers, largest, alphabet)
    result = model.predict(X, batch_size=n_batch, verbose=0)
    

将预测结果转化成数字形式，比较实际值和预测值：

    
    
    expected = [invert(z, alphabet) for z in y]
    predicted = [invert(z, alphabet) for z in result]
    for i in range(20):
        print('Expected=%s, Predicted=%s' % (expected[i], predicted[i]))
    

![](https://images.gitbook.cn/2a237550-75c7-11ea-9613-4f950122a537)

通过上面的这个很简单的例子，我们知道了 seq2seq 是如何处理序列的，在后面的文章中我们还会有其他应用例子。

* * *

参考文献：

  * Nishant Shukla， _Machine Learning with TensorFlow_
  * Jason Brownlee， _Long Short-Term Memory Networks With Python_

