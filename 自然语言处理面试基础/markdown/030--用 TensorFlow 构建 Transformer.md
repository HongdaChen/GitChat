上一篇我们介绍了 Transformer 的原理，今天我们来根据论文动手编写一个
Transformer，并将其应用在我们前面讲过的机器翻译的例子上，大家感兴趣也可以进行对比学习。

这里的数据部分和前面机器翻译那篇是一样的，使用 [ManyThings.org](http://www.manythings.org/anki/)
的英语—法语的数据集，同样为了简化只选取 20 句。因为例子比较小，我们也会对论文中设置的一些参数进行简化。

### 准备数据

    
    
    import tensorflow as tf
    import numpy as np
    import unicodedata
    import re
    import time
    import matplotlib.pyplot as plt
    
    raw_data = (
        ('What a ridiculous concept!', 'Quel concept ridicule !'),
        ('Your idea is not entirely crazy.', "Votre idée n'est pas complètement folle."),
        ("A man's worth lies in what he is.", "La valeur d'un homme réside dans ce qu'il est."),
        ('What he did is very wrong.', "Ce qu'il a fait est très mal."),
        ("All three of you need to do that.", "Vous avez besoin de faire cela, tous les trois."),
        ("Are you giving me another chance?", "Me donnez-vous une autre chance ?"),
        ("Both Tom and Mary work as models.", "Tom et Mary travaillent tous les deux comme mannequins."),
        ("Can I have a few minutes, please?", "Puis-je avoir quelques minutes, je vous prie ?"),
        ("Could you close the door, please?", "Pourriez-vous fermer la porte, s'il vous plaît ?"),
        ("Did you plant pumpkins this year?", "Cette année, avez-vous planté des citrouilles ?"),
        ("Do you ever study in the library?", "Est-ce que vous étudiez à la bibliothèque des fois ?"),
        ("Don't be deceived by appearances.", "Ne vous laissez pas abuser par les apparences."),
        ("Excuse me. Can you speak English?", "Je vous prie de m'excuser ! Savez-vous parler anglais ?"),
        ("Few people know the true meaning.", "Peu de gens savent ce que cela veut réellement dire."),
        ("Germany produced many scientists.", "L'Allemagne a produit beaucoup de scientifiques."),
        ("Guess whose birthday it is today.", "Devine de qui c'est l'anniversaire, aujourd'hui !"),
        ("He acted like he owned the place.", "Il s'est comporté comme s'il possédait l'endroit."),
        ("Honesty will pay in the long run.", "L'honnêteté paye à la longue."),
        ("How do we know this isn't a trap?", "Comment savez-vous qu'il ne s'agit pas d'un piège ?"),
        ("I can't believe you're giving up.", "Je n'arrive pas à croire que vous abandonniez."),
    )
    

**数据预处理：**

  * 将 Unicode 转换为 ASCII
  * 将字符串规范化，除了 `(a-z, A-Z, ".", "?", "!", ",")` 之外，将不需要的标记替换成空格，
  * 在紧跟单词后面的标点前面加上空格，例如：`"he is a boy." => "he is a boy ."`。

    
    
    def unicode_to_ascii(s):
        return ''.join(
            c for c in unicodedata.normalize('NFD', s)
            if unicodedata.category(c) != 'Mn')
    
    def normalize_string(s):
        s = unicode_to_ascii(s)
        s = re.sub(r'([!.?])', r' \1', s)
        s = re.sub(r'[^a-zA-Z.!?]+', r' ', s)    # 除了 (a-z, A-Z, ".", "?", "!", ",") 之外，将不需要的标记替换成空格
        s = re.sub(r'\s+', r' ', s)                # 在紧跟单词后面的标点前面加上空格，eg: "he is a boy." => "he is a boy ."
        return s
    

**给翻译句的前后加上这两个标记：**

  * 将数据中的英语和法语句子分别拿出来，形成原句和翻译句两个部分，
  * 并在翻译句的开始和结尾标记 `<start>` 与 `<end>`。

    
    
    raw_data_en, raw_data_fr = list(zip(*raw_data))
    raw_data_en, raw_data_fr = list(raw_data_en), list(raw_data_fr)
    
    raw_data_en = [normalize_string(data) for data in raw_data_en]
    raw_data_fr_in = ['<start> ' + normalize_string(data) for data in raw_data_fr]
    raw_data_fr_out = [normalize_string(data) + ' <end>' for data in raw_data_fr]
    

**接下里我们需要用 Tokenizer 将字符串转换为整数序列：**

  * Tokenizer 里面的 filters 指的是要去掉所有标点符号，因为前面已经进行了预处理，这里就设置为空。
  * fit_on_texts：是根据传入文本中每个单词出现的频数，给出对应单词的索引号。

例如 `"The cat sat on the mat."`，the 出现了两次排在第一位，所以 `word_index["the"] =
1`，索引越小的说明这个单词出现的频数越大。

    
    
    en_tokenizer = tf.keras.preprocessing.text.Tokenizer(filters='')
    en_tokenizer.fit_on_texts(raw_data_en)
    
    print(en_tokenizer.word_index)
    
    
    
    {
     '.': 1, 'you': 2, '?': 3, 'the': 4, 'a': 5, 'is': 6, 'he': 7,
     'what': 8, 'in': 9, 'do': 10, 'can': 11, 't': 12, 'did': 13, 'giving': 14
     ...
    }
    

  * 再用 texts_to_sequences 将文本数据转化为整数序列数据，也就是将单词替换成 word_index 中相应的索引号。
  * 此外还需要将句子长度补齐，使它们具有相同的长度，在后面创建 tf.data.Dataset 时候会用到，因为到时需要对输入序列使用词嵌入，对输出序列进行热编码等操作。

    
    
    data_en = en_tokenizer.texts_to_sequences(raw_data_en)
    data_en = tf.keras.preprocessing.sequence.pad_sequences(data_en, padding='post')
    
    print(data_en[:3])
    
    
    
    [[ 8  5 21 22 23  0  0  0  0  0]
     [24 25  6 26 27 28  1  0  0  0]
     [ 5 29 30 31 32  9  8  7  6  1]]
    

**我们再用同样的预处理步骤将目标语言的句子进行处理：**

    
    
    fr_tokenizer = tf.keras.preprocessing.text.Tokenizer(filters='')
    
    fr_tokenizer.fit_on_texts(raw_data_fr_in)
    fr_tokenizer.fit_on_texts(raw_data_fr_out)
    
    data_fr_in = fr_tokenizer.texts_to_sequences(raw_data_fr_in)
    data_fr_in = tf.keras.preprocessing.sequence.pad_sequences(data_fr_in,
                                                               padding='post')
    
    data_fr_out = fr_tokenizer.texts_to_sequences(raw_data_fr_out)
    data_fr_out = tf.keras.preprocessing.sequence.pad_sequences(data_fr_out,
                                                               padding='post')
    

**建立数据集 Dataset：**

用 from_tensor_slices 可以保持三个集合的独立性，shuffle 随机打乱数据集中的数据，Batch 将数据集划分成几份。

    
    
    dataset = tf.data.Dataset.from_tensor_slices(
        (data_en, data_fr_in, data_fr_out))
    dataset = dataset.shuffle(20).batch(5)
    

接下来我们就按照 Transformer 的结构图一块一块构建模型，首先建立基础组件 Positional Encoding 和 Multi-Head
Attention。

### Positional Encoding

我们知道在单词进入模型之前要先做 word embedding，会将单词表示成一个长度为 MODEL_SIZE
的向量，不过这个只是词语含义的特征，并没有位置的特征，于是需要做
positional_embedding，它是对单词的位置信息进行编码，会将单词的位置这个特征也编码成一个长度为 MODEL_SIZE
的向量，计算公式如下所示：

$$ PE(pos, 2i) = sin( \frac{pos}{ 10000^{2i/d_{model}} } ) $$

$$ PE(pos, 2i + 1) = cos( \frac{pos}{ 10000^{2i/d_{model}} } ) $$

下面的 positional_embedding 函数会接收两个参数：pos 代表句子中每个单词的序号，本例中句子的最大长度 = 14，所以序号从 0 到
13，以及编码向量的长度 `MODEL_SIZE = 128`。

这个函数要做的事情是给定单词的位置 pos 后，编码成 model_size 维的向量，它会在 128
个位置中的偶数位置上使用正弦编码，在奇数位置上使用余弦编码，这样向量中的每一个维度对应着正弦曲线，波长构成了从 $2\pi$ 到 $10000*2\pi$
的等比序列。

因为本例中最长的句子长度是 14，所以会建立 14 个位置的编码向量，最后生成的 pes 的维度是
14*128，之后对于每个句子在相应的位置上，会在词向量的基础上加上该单词所在的位置的编码向量。

    
    
    def positional_embedding(pos, model_size):
        PE = np.zeros((1, model_size))
    
        for i in range(model_size):
            if i % 2 == 0:
                PE[:, i] = np.sin(pos / 10000 ** (i / model_size))
            else:
                PE[:, i] = np.cos(pos / 10000 ** ((i - 1) / model_size))
    
        return PE
    
    max_length = max(len(data_en[0]), len(data_fr_in[0]))
    MODEL_SIZE = 128
    
    pes = []
    for i in range(max_length):
        pes.append(positional_embedding(i, MODEL_SIZE))
    
    pes = np.concatenate(pes, axis=0)
    pes = tf.constant(pes, dtype=tf.float32)
    

我们可以将生成的 14 个向量可视化看一下效果：

    
    
    plt.pcolormesh(pes, cmap='RdBu')
    plt.xlabel('Depth')
    plt.xlim((0, 128))
    plt.ylabel('Position')
    plt.colorbar()
    plt.show()
    

![](https://images.gitbook.cn/f80e57a0-8ed1-11ea-86f7-9b5ee2f8f4da)

### Multi-Head Attention

下面建立 Transformer 的核心部分为 Multi-Head Attention，它的作用时可以注意到来自多个角度不同表示空间的信息。

在这里我们要建立如下图的模型结构：

![](https://images.gitbook.cn/229296d0-8ed2-11ea-8e9a-9bc3d95576e7)

从图中可以看到这个结构大体有下面几个步骤。

首先 Q、K、V 的初始大小是 model_size，我们需要将它们分成 h 份。

然后对其中每一份进行 Scaled Dot-Product Attention 机制，具体结构为：

![](https://images.gitbook.cn/4e336990-8ed2-11ea-9144-a708da03c9c4)

用公式表示为：

$$ Attention(Q, K, V) = softmax( \frac{ QK^T }{ \sqrt{d_k} } ) V $$

如果不考虑多个 head 的话，Scaled Dot-Product Attention 可以用下面的代码实现，它其实和 Luong Attention
基本差不多：

    
    
        def call(self, query, value):
    
            score = tf.matmul(query, value, transpose_b=True) / tf.math.sqrt(tf.dtypes.cast(self.key_size, tf.float32))
    
            alignment = tf.nn.softmax(score, axis=2)
    
            context = tf.matmul(alignment, value)
    
            return context
    

如果考虑多个 head，用一个 for 循环来解决。

把 h 个注意力结果合并起来，这里 Multi-Head Attention 的公式表示为：

$$ MultiHead(Q, K, V) = Concat( head_1, ... , head_h) W^o $$

其中，

$$ head_i = Attention( QW_i^Q, KW_i^K, VW_i^V) $$

最后经过线性映射得到最终的输出。

论文里面 `d_{model}=512，h=8`，作为简化我们这里 `d_{model}=128，h=2`，下面按照公式构建模型：

    
    
    class MultiHeadAttention(tf.keras.Model):
        def __init__(self, model_size, h):
            super(MultiHeadAttention, self).__init__()
            self.query_size = model_size // h
            self.key_size = model_size // h
            self.value_size = model_size // h
            self.h = h    
    
            self.wq = [tf.keras.layers.Dense(self.query_size) for _ in range(h)]
            self.wk = [tf.keras.layers.Dense(self.key_size) for _ in range(h)]
            self.wv = [tf.keras.layers.Dense(self.value_size) for _ in range(h)]
            self.wo = tf.keras.layers.Dense(model_size)
    
        def call(self, query, value):
            # query 的维度为 (batch, query_len, model_size)
            # value 的维度为 (batch, value_len, model_size)
            heads = []
    
            # 一共 h 份，对每一份实现 scaled dot-product attention 机制的公式
            for i in range(self.h):
                score = tf.matmul(self.wq[i](query), self.wk[i](value), transpose_b=True)
    
                # score 的维度为 (batch, query_len, value_len)
                score /= tf.math.sqrt(tf.dtypes.cast(self.key_size, tf.float32))
    
                # alignment 的维度为 (batch, query_len, value_len)
                alignment = tf.nn.softmax(score, axis=2)           
    
                # head 的维度为 (batch, decoder_len, value_size)
                head = tf.matmul(alignment, self.wv[i](value))
    
                heads.append(head)
    
            # 将所有的 head 连接起来
            heads = tf.concat(heads, axis=2)
            heads = self.wo(heads)
            # heads 的维度为 (batch, query_len, model_size)
    
            return heads
    

### Encoder

完成了 Positional Encoding 和 Multi-head Attention 的搭建，接下来要构建 Encoder 和 Decoder 了。

在这里我们要建立如下图所示的模型结构：

![](https://images.gitbook.cn/bf335470-8ed2-11ea-861a-9398d62a6944)

可以看到 Encoder 的结构包括下面几个部分：

  * 一个 Embedding 层和 Positional Encoding 层
  * 一个或者多个 Encoder 组件，论文中是 6 个，我们这里用 2 个
  * 一个 Multi-Head Attention 层
  * 一个 Normalization 层
  * 一个 Feed-Forward Network
  * 一个 Normalization 层

在前向计算 call 中：

  1. 首先将 Positional Encoding 加到 word embedding 上，拼接到一起得到 sub_in，维度为 `(batch_size, length, model_size)`；
  2. 然后进入 `num_layers = 2` 个 (Attention + FFN) 中：
  3. 先做 Multi-Head Attention，得到结果 sub_out；
  4. 然后是 Residual 连接，就是将 sub_out 与 sub_in 相加得到新的 sub_out；紧接着给 sub_out 做 Normalization；
  5. 然后进入 FFN 层，它的输入 ffn_in 就是前面的 sub_out；
  6. 用同样的方式对 FFN 做 Residual 连接和 Normalization；
  7. 最后当前 Encoder 组块的 FFN 的输出将作为下一个 (Attention + FFN) 的输入：`sub_in = ffn_out`。

    
    
    class Encoder(tf.keras.Model):
    
        def __init__(self, vocab_size, model_size, num_layers, h):
            super(Encoder, self).__init__()
            self.model_size = model_size
            self.num_layers = num_layers
            self.h = h
    
            # Embedding 层
            self.embedding = tf.keras.layers.Embedding(vocab_size, model_size)
    
            # num_layers 个 Multi-Head Attention 层和 Normalization 层
            self.attention = [MultiHeadAttention(model_size, h) for _ in range(num_layers)]
            self.attention_norm = [tf.keras.layers.BatchNormalization() for _ in range(num_layers)]
    
            # num_layers 个 FFN 和 Normalization 层
            self.dense_1 = [tf.keras.layers.Dense(model_size * 4, activation='relu') for _ in range(num_layers)]
            self.dense_2 = [tf.keras.layers.Dense(model_size) for _ in range(num_layers)]
            self.ffn_norm = [tf.keras.layers.BatchNormalization() for _ in range(num_layers)]
    
        def call(self, sequence):
    
            sub_in = []
            for i in range(sequence.shape[1]):
                # 计算嵌入向量
                embed = self.embedding(tf.expand_dims(sequence[:, i], axis=1))
    
                # 将 Positional Encoding 加到 embedded 向量上
                sub_in.append(embed + pes[i, :])
    
            # 拼接结果，维度为 (batch_size, length, model_size)
            sub_in = tf.concat(sub_in, axis=1)
    
            # 一共有 num_layers 个 (Attention + FFN)
            for i in range(self.num_layers):
    
                sub_out = []
    
                # 遍历序列长度
                for j in range(sub_in.shape[1]):
                    # 计算上下文向量
                    attention = self.attention[i](
                        tf.expand_dims(sub_in[:, j, :], axis=1), sub_in)
    
                    sub_out.append(attention)
    
                # 合并结果维度为 (batch_size, length, model_size)
                sub_out = tf.concat(sub_out, axis=1)
    
                # Residual 连接
                sub_out = sub_in + sub_out
                # Normalization 层
                sub_out = self.attention_norm[i](sub_out)
    
                # FFN 层
                ffn_in = sub_out
                ffn_out = self.dense_2[i](self.dense_1[i](ffn_in))
    
                # 加上 residual 连接
                ffn_out = ffn_in + ffn_out
                # Normalization 层
                ffn_out = self.ffn_norm[i](ffn_out)
    
                # FFN 的输出作为下一个 Multi-Head Attention 的输入
                sub_in = ffn_out
    
            return ffn_out
    

### Decoder

我们要构建的 Decoder 模型结构为：

![](https://images.gitbook.cn/331ec680-8ed3-11ea-a141-a13a4f8833dd)

在每个 Decoder 组块中有两个注意力层，一个是位于底部的 Multi-Head Attention 记为
attention_bot，一个是位于中间的记为 attention_mid。

  1. 第一步和 Encoder 中一样的，做完 word embedding 后加上 Positional Encoding；
  2. 这个结果将作为第一个 Multi-Head Attention 的输入：`bot_sub_in = embed_out`；论文中指出这个注意力层需要做 mask，我们用 `values = bot_sub_in[:, :j, :]` 来实现，在当前指标 j 的右侧的数据就不要了；
  3. 然后进入第二个 Multi-Head Attention 层，它的输入有来自上面 Multi-Head Attention 层的输出，还有 Encoder 的输出：encoder_output；
  4. 接着进入 FFN 层，这里和 Encoder 中是一样的；
  5. 另外 Decoder 比 Encoder 多了 `logits = self.dense(ffn_out)`，用来计算最后的输出结果。

    
    
    class Decoder(tf.keras.Model):
        def __init__(self, vocab_size, model_size, num_layers, h):
            super(Decoder, self).__init__()
            self.model_size = model_size
            self.num_layers = num_layers
            self.h = h
    
            self.embedding = tf.keras.layers.Embedding(vocab_size, model_size)
    
            self.attention_bot = [MultiHeadAttention(model_size, h) for _ in range(num_layers)]
            self.attention_bot_norm = [tf.keras.layers.BatchNormalization() for _ in range(num_layers)]
    
            self.attention_mid = [MultiHeadAttention(model_size, h) for _ in range(num_layers)]
            self.attention_mid_norm = [tf.keras.layers.BatchNormalization() for _ in range(num_layers)]
    
            self.dense_1 = [tf.keras.layers.Dense(model_size * 4, activation='relu') for _ in range(num_layers)]
            self.dense_2 = [tf.keras.layers.Dense(model_size) for _ in range(num_layers)]
            self.ffn_norm = [tf.keras.layers.BatchNormalization() for _ in range(num_layers)]
    
            self.dense = tf.keras.layers.Dense(vocab_size)
    
    
        def call(self, sequence, encoder_output):
            # Embedding 和 Positional Embedding，和 Encoder 中一样的
            embed_out = []
            for i in range(sequence.shape[1]):
                embed = self.embedding(tf.expand_dims(sequence[:, i], axis=1))
                embed_out.append(embed + pes[i, :])
    
            embed_out = tf.concat(embed_out, axis=1)
    
    
            bot_sub_in = embed_out
    
            for i in range(self.num_layers):
                # 模型中最底部的 Multi-Head Attention 层，即 target 序列的 self-attention，这一层需要做 mask
                bot_sub_out = []
    
                for j in range(bot_sub_in.shape[1]):
                    # 这里做了 mask，在当前 j 的右侧的数据就不要了
                    values = bot_sub_in[:, :j, :]
                    attention = self.attention_bot[i](
                        tf.expand_dims(bot_sub_in[:, j, :], axis=1), values)
    
                    bot_sub_out.append(attention)
    
                bot_sub_out = tf.concat(bot_sub_out, axis=1)
                bot_sub_out = bot_sub_in + bot_sub_out
                bot_sub_out = self.attention_bot_norm[i](bot_sub_out)
    
                # 中间的 Multi-Head Attention 层，它的输入来自上面 Multi-Head Attention 层的输出和 Encoder 的输出 encoder_output
                mid_sub_in = bot_sub_out
    
                mid_sub_out = []
                for j in range(mid_sub_in.shape[1]):
                    attention = self.attention_mid[i](
                        tf.expand_dims(mid_sub_in[:, j, :], axis=1), encoder_output)
    
                    mid_sub_out.append(attention)
    
                mid_sub_out = tf.concat(mid_sub_out, axis=1)
                mid_sub_out = mid_sub_out + mid_sub_in
                mid_sub_out = self.attention_mid_norm[i](mid_sub_out)
    
                # FFN
                ffn_in = mid_sub_out
    
                ffn_out = self.dense_2[i](self.dense_1[i](ffn_in))
                ffn_out = ffn_out + ffn_in
                ffn_out = self.ffn_norm[i](ffn_out)
    
                bot_sub_in = ffn_out
    
            logits = self.dense(ffn_out)
    
            return logits
    

### Loss Function 和 Optimizer

损失函数用 SparseCategoryCrossentropy，并带有 mask 用来过滤掉之前数据预处理时填充 0 的地方。

    
    
    crossentropy = tf.keras.losses.SparseCategoricalCrossentropy(
        from_logits=True)
    
    def loss_func(targets, logits):
        mask = tf.math.logical_not(tf.math.equal(targets, 0))
        mask = tf.cast(mask, dtype=tf.int64)
        loss = crossentropy(targets, logits, sample_weight=mask)
    
        return loss
    
    optimizer = tf.keras.optimizers.Adam()
    

### 训练函数

前面的组件都搭建完毕，训练过程也很直观明了了，从 Encoder 那里得到输出结果，传递给
Decoder，最后计算预测结果和实际结果之间的损失，用优化算法对模型的参数进行学习更新。

    
    
    @tf.function
    def train_step(source_seq, target_seq_in, target_seq_out):
        with tf.GradientTape() as tape:
            encoder_output = encoder(source_seq)
    
            decoder_output = decoder(target_seq_in, encoder_output)
    
            loss = loss_func(target_seq_out, decoder_output)
    
        variables = encoder.trainable_variables + decoder.trainable_variables
        gradients = tape.gradient(loss, variables)
        optimizer.apply_gradients(zip(gradients, variables))
    
        return loss
    

### 预测函数

  1. 当不指定要预测的数据时，就从训练集中随机选择一个句子作为输入；
  2. 然后对这个句子要做同样的预处理过程；
  3. 将处理好的数据输入给 Encoder 后得到一个输出 en_output；
  4. 第一个 Decoder 的输入包括 `<start>` 所对应的标记，以及 en_output。然后根据 Decoder 的输出 de_output 可以得到一个预测结果 new_word；
  5. 接下来就和以往的 seq2seq 略有不同。

之前的模型预测的一般过程是：

先将 `<start>` 输入给模型，最后一个 output 是预测词，将预测词加入到结果中，将预测词和它相关联的状态合在一起给模型并重复步骤 2。

在 Transformer 中，因为缺少序列机制，步骤变为：

先将 `<start>` 输入给模型，最后一个 output 是预测词，将预测词加入到结果中，将整个结果给模型并重复步骤 2。

所以这时要将 new_word 直接添加到前面所有的 de_input 上作为新的 de_input，在进入下一个 Decoder 时，要将整个
de_input 以及 en_output 都加入到模型中。

    
    
    def predict(test_source_text=None):
        # 没有测试序列时，从训练数据中随机选择一个
        if test_source_text is None:
            test_source_text = raw_data_en[np.random.choice(len(raw_data_en))]
    
        print(test_source_text)
    
        # 预处理
        test_source_seq = en_tokenizer.texts_to_sequences([test_source_text])
        print(test_source_seq)
    
        en_output = encoder(tf.constant(test_source_seq))
    
        de_input = tf.constant([[fr_tokenizer.word_index['<start>']]], dtype=tf.int64)
    
        out_words = []
    
        while True:
            de_output = decoder(de_input, en_output)
    
            # 最后的 token 作为预测值
            new_word = tf.expand_dims(tf.argmax(de_output, -1)[:, -1], axis=1)
            out_words.append(fr_tokenizer.index_word[new_word.numpy()[0][0]])
    
            # 下一个输入是一个新的序列，包括输入序列和预测值
            de_input = tf.concat((de_input, new_word), axis=-1)
    
            # 当遇到 <end> 或者长度超过 14
            if out_words[-1] == '<end>' or len(out_words) >= 14:
                break
    
        print(' '.join(out_words))
    

### 训练模型

我们将论文中模型进行了简化，只采用 H = 2 两个多头注意力，以及在每个 Encoder 和 Decoder 中都采用 NUM_LAYERS = 2
个组块。

    
    
    H = 2
    NUM_LAYERS = 2
    
    en_vocab_size = len(en_tokenizer.word_index) + 1
    encoder = Encoder(en_vocab_size, MODEL_SIZE, NUM_LAYERS, H)
    
    fr_vocab_size = len(fr_tokenizer.word_index) + 1
    max_len_fr = data_fr_in.shape[1]
    decoder = Decoder(fr_vocab_size, MODEL_SIZE, NUM_LAYERS, H)
    
    NUM_EPOCHS = 100
    
    start_time = time.time()
    for e in range(NUM_EPOCHS):
        for batch, (source_seq, target_seq_in, target_seq_out) in enumerate(dataset.take(-1)):
            loss = train_step(source_seq, target_seq_in,
                              target_seq_out)
    
        print('Epoch {} Loss {:.4f}'.format(
              e + 1, loss.numpy()))
    
        try:
          predict()
        except Exception as e:
          print(e)
          continue
    
    
    
    

**训练 100 次后，观察结果可以发现：**

    
    
    Epoch 1 Loss 1.1374
    Your idea is not entirely crazy .
    [[24, 25, 6, 26, 27, 28, 1]]
    tom idee idee travaillent completement completement scientifiques . <end>
    

Epoch = 1 时，得到的翻译结果为：

    
    
    tom idee idee travaillent completement completement scientifiques .
    

这句法语如果用谷歌翻译成英语的结果是：

    
    
    tom idee idee work completely completely scientific. 
    

除了 completely 和 entirely 意思相近外，整个句子和原句很难看出直接的关系。

    
    
    Epoch 100 Loss 0.0002
    Excuse me . Can you speak English ?
    [[67, 15, 1, 11, 2, 68, 69, 3]]
    je vous prie de m excuser ! savez vous parler anglais ? <end>
    

而在 Epoch = 100 时，模型预测的结果为：

    
    
    je vous prie de m excuser ! savez vous parler anglais ?
    

这句法语用谷歌翻译成英语的结果是：

    
    
    please excuse me ! can you speak english?
    

可以看到和原句基本一致了。

* * *

参考文献：

  * [理解语言的 Transformer 模型](https://www.tensorflow.org/tutorials/text/transformer#top_of_page)

