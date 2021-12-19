前面我们讲过了 LSTM 的两个变体 Peephole LSTM 和 GRU
的模型原理，在这一节中我们会将这些模型应用起来，分别用它们来做一个文本生成的任务，并对比不同方案的效果。

我们先通过 TF1 构建 LSTM、Peephole、GRU 模型的代码来看看手动搭建过程中有什么区别，然后再用 Keras 中的 API 使用模型。

### LSTM、Peephole 源码对比

首先比较 LSTM 和 Peephole LSTM，先来回顾一下模型的公式：

![](https://images.gitbook.cn/060ee2f0-69fe-11ea-ae8e-993043b82faa)

（下面代码都是左边为 LSTM，右边为 Peephole LSTM 的）

首先定义输入门的参数：

  * ix 代表 $U_i$，是 input gate 关于 $x_t$ 的参数；
  * im 代表 $W_i$，是 input gate 关于 $h_{t-1}$ 的参数；
  * ib 为公式中输入门的参数 $b_i$，是 input gate 中的偏置；
  * ic 代表 $P_i$，是 input gate 关于 $c_{t-1}$ 的参数。

![](https://images.gitbook.cn/1a175d90-69fe-11ea-8181-9bde95dea606)

在遗忘门的参数中，也是 Peephole 比 LSTM 多了一个连接 c 的参数：

  * fx 代表 $U_f$，是 forget gate 关于 $x_t$ 的参数；
  * fm 代表 $W_f$，是 forget gate 关于 $h_{t-1}$ 的参数；
  * fb 为公式中输入门的参数 $b_f$，是 forget gate 中的偏置；
  * fc 代表 $P_f$，是 forget gate 关于 $c_{t-1}$ 的参数。

![](https://images.gitbook.cn/ab3be3e0-69fe-11ea-af72-7dcc88a7b3ca)

两边的记忆单元的参数是一样的，因为公式也是一样的：

  * cx 代表 $U_c$，是 $\tilde{C}_t$ 关于 $x_t$ 的参数；
  * cm 代表 $W_c$，是 $\tilde{C}_t$ 关于 $h_{t-1}$ 的参数；
  * cb 为公式中输入门的参数 $b_c$，是 $\tilde{C}_t$ 中的偏置；

![](https://images.gitbook.cn/bf99d180-69fe-11ea-8181-9bde95dea606)

和输入门，遗忘门一样，在输出门中 Peephole 也是多了一个连接 $c_{t-1}$ 的参数：

  * ox 代表 $U_o$，是 output gate 关于 $x_t$ 的参数；
  * om 代表 $W_o$，是 output gate 关于 $h_{t-1}$ 的参数；
  * ob 为公式中输入门的参数 $b_o$，是 output gate 中的偏置；
  * oc 代表 $P_o$，是 output gate 关于 $c_{t-1}$ 的参数。

![](https://images.gitbook.cn/d0e5eb40-69fe-11ea-8181-9bde95dea606)

最后在建立模型时，在 Peephole 的输入门，遗忘门，输出门那里多了 P c t-1 这一项，其他都是是一样的，其中模型的输入为：

  * i 代表 $x_t$ ，
  * o 代表 $h_{t-1}$，
  * state 代表 $c_{t-1}$， 

![](https://images.gitbook.cn/003b1370-69ff-11ea-b373-7feb1688649d)

经过这样的比较，相信大家对两个模型的原理有了更深入的理解，下面是这部分的完整代码。

**LSTM：**

    
    
    # 输入门（i_t）：用来向单元状态写入多少信息
    # ix 将当前输入连接到输入门
    ix = tf.Variable(tf.truncated_normal([vocabulary_size, num_nodes], stddev=0.02))
    # im 将先前隐藏层状态连接到输入门
    im = tf.Variable(tf.truncated_normal([num_nodes, num_nodes], stddev=0.02))
    # 输入门的偏置
    ib = tf.Variable(tf.random_uniform([1, num_nodes],-0.02, 0.02))
    
    # 遗忘门 (f_t) ：从单元状态中丢掉多少信息
    # fx 将当前输入连接到遗忘门
    fx = tf.Variable(tf.truncated_normal([vocabulary_size, num_nodes], stddev=0.02))
    # fm 将先前隐藏层状态连接到遗忘门
    fm = tf.Variable(tf.truncated_normal([num_nodes, num_nodes], stddev=0.02))
    # 遗忘门的偏置
    fb = tf.Variable(tf.random_uniform([1, num_nodes],-0.02, 0.02))
    
    # 候选值 (c~_t) ：用于计算当前单元状态
    # cx 将当前输入连接到候选值
    cx = tf.Variable(tf.truncated_normal([vocabulary_size, num_nodes], stddev=0.02))
    # cm 将先前隐藏层状态连接到候选值
    cm = tf.Variable(tf.truncated_normal([num_nodes, num_nodes], stddev=0.02))
    # 候选值的偏置
    cb = tf.Variable(tf.random_uniform([1, num_nodes],-0.02,0.02))
    
    # 输出门 ： 从单元状态中输出多少信息
    # ox 将当前输入连接到输出门
    ox = tf.Variable(tf.truncated_normal([vocabulary_size, num_nodes], stddev=0.02))
    # om 将先前隐藏层状态连接到输出门
    om = tf.Variable(tf.truncated_normal([num_nodes, num_nodes], stddev=0.02))
    # 输出门的偏置
    ob = tf.Variable(tf.random_uniform([1, num_nodes],-0.02,0.02))
    
    # Softmax 分类器的权重和偏置
    w = tf.Variable(tf.truncated_normal([num_nodes, vocabulary_size], stddev=0.02))
    b = tf.Variable(tf.random_uniform([vocabulary_size],-0.02,0.02))
    
    # 展开时保存的变量
    # 隐藏层状态
    saved_output = tf.Variable(tf.zeros([batch_size, num_nodes]), trainable=False)
    # 单元状态
    saved_state = tf.Variable(tf.zeros([batch_size, num_nodes]), trainable=False)
    
    saved_valid_output = tf.Variable(tf.zeros([1, num_nodes]), trainable=False)
    saved_valid_state = tf.Variable(tf.zeros([1, num_nodes]), trainable=False)
    
    saved_test_output = tf.Variable(tf.zeros([1, num_nodes]),trainable=False)
    saved_test_state = tf.Variable(tf.zeros([1, num_nodes]),trainable=False)
    
    algorithm = 'lstm'
    filename_to_save = algorithm + filename_extension +'.csv'
    
    # 单元计算的定义
    def lstm_cell(i, o, state):
    
        # 建立一个 LSTM 单元
        input_gate = tf.sigmoid(tf.matmul(i, ix) + tf.matmul(o, im) + ib)
        forget_gate = tf.sigmoid(tf.matmul(i, fx) + tf.matmul(o, fm) + fb)
        update = tf.matmul(i, cx) + tf.matmul(o, cm) + cb
        state = forget_gate * state + input_gate * tf.tanh(update)
        output_gate = tf.sigmoid(tf.matmul(i, ox) + tf.matmul(o, om) + ob)
    
        return output_gate * tf.tanh(state), state
    

**Peephole LSTM：**

    
    
    # 输入门：
    ix = tf.Variable(tf.truncated_normal([vocabulary_size, num_nodes], stddev=0.01))
    im = tf.Variable(tf.truncated_normal([num_nodes, num_nodes], stddev=0.01))
    ic = tf.Variable(tf.truncated_normal([1,num_nodes], stddev=0.01))
    ib = tf.Variable(tf.random_uniform([1, num_nodes],0.0, 0.01))
    
    # 遗忘门：
    fx = tf.Variable(tf.truncated_normal([vocabulary_size, num_nodes], stddev=0.01))
    fm = tf.Variable(tf.truncated_normal([num_nodes, num_nodes], stddev=0.01))
    fc = tf.Variable(tf.truncated_normal([1,num_nodes], stddev=0.01))
    fb = tf.Variable(tf.random_uniform([1, num_nodes],0.0, 0.01))
    
    # 记忆单元：
    cx = tf.Variable(tf.truncated_normal([vocabulary_size, num_nodes], stddev=0.01))
    cm = tf.Variable(tf.truncated_normal([num_nodes, num_nodes], stddev=0.01))
    cb = tf.Variable(tf.random_uniform([1, num_nodes],0.0,0.01))
    
    # 输出门：
    ox = tf.Variable(tf.truncated_normal([vocabulary_size, num_nodes], stddev=0.01))
    om = tf.Variable(tf.truncated_normal([num_nodes, num_nodes], stddev=0.01))
    oc = tf.Variable(tf.truncated_normal([1,num_nodes], stddev=0.01))
    ob = tf.Variable(tf.random_uniform([1, num_nodes],0.0,0.01))
    
    # Softmax 分类器的权重和偏置
    w = tf.Variable(tf.truncated_normal([num_nodes, vocabulary_size], stddev=0.01))
    b = tf.Variable(tf.random_uniform([vocabulary_size],0.0,0.01))
    
    # 展开时保存的变量
    saved_output = tf.Variable(tf.zeros([batch_size, num_nodes]), trainable=False)
    saved_state = tf.Variable(tf.zeros([batch_size, num_nodes]), trainable=False)
    
    saved_valid_output = tf.Variable(tf.zeros([1, num_nodes]), trainable=False)
    saved_valid_state = tf.Variable(tf.zeros([1, num_nodes]), trainable=False)
    
    saved_test_output = tf.Variable(tf.zeros([1, num_nodes]), trainable=False)
    saved_test_state = tf.Variable(tf.zeros([1, num_nodes]), trainable=False)
    
    algorithm = 'lstm_peephole'
    filename_to_save = algorithm + filename_extension +'.csv'
    
    # 单元计算的定义
    def lstm_with_peephole_cell(i, o, state):
    
        # 建立一个 peephole LSTM 单元
        input_gate = tf.sigmoid(tf.matmul(i, ix) + state*ic + tf.matmul(o, im) + ib)
        forget_gate = tf.sigmoid(tf.matmul(i, fx) + state*fc + tf.matmul(o, fm) + fb)
        update = tf.matmul(i, cx) + tf.matmul(o, cm) + cb
        state = forget_gate * state + input_gate * tf.tanh(update)
        output_gate = tf.sigmoid(tf.matmul(i, ox) + state*oc + tf.matmul(o, om) + ob)
    
        return output_gate * tf.tanh(state), state
    

### LSTM、GRU 源码对比

同样来回顾一下二者的模型原理：

![](https://images.gitbook.cn/c29800e0-69ff-11ea-ae8e-993043b82faa)

可以看出 GRU 和 LSTM 在数学模型上还是有较大变化的，所以就不逐步比较了，上图中的双箭头表示功能类似，在 LSTM 中有 3 个门需要 6
步完成的，在 GRU 中只有 2 个门需要 4 步完成，明显参数少了很多。

在搭建模型时，就按照数学模型前向计算一步一步构建即可：

![](https://images.gitbook.cn/d40b51b0-69ff-11ea-b31d-2b61fbcda176)

LSTM 的代码请参见上一部分，GRU 的完整代码如下，其中：

  * rx 代表 $U_r$，是 reset gate 关于 $x_t$ 的参数；
  * rh 代表 $W_r$，是 reset gate 关于 $h_{t-1}$ 的参数；
  * hx 代表 U，是 $\tilde{h}_t$ 关于 $x_t$ 的参数；
  * hh 代表 W，是 $\tilde{h}_t$ 关于 $r_t o h_{t-1}$ 的参数；
  * zx 代表 $U_z$，是更新门关于 $x_t$ 的参数；
  * zh 代表 $W_z$，是更新门关于 $h_{t-1}$ 的参数。

    
    
    # 重置门：
    rx = tf.Variable(tf.truncated_normal([vocabulary_size, num_nodes], stddev=0.01))
    rh = tf.Variable(tf.truncated_normal([num_nodes, num_nodes], stddev=0.01))
    rb = tf.Variable(tf.random_uniform([1, num_nodes],0.0, 0.01))
    
    # 隐藏层状态：
    hx = tf.Variable(tf.truncated_normal([vocabulary_size, num_nodes], stddev=0.01))
    hh = tf.Variable(tf.truncated_normal([num_nodes, num_nodes], stddev=0.01))
    hb = tf.Variable(tf.random_uniform([1, num_nodes],0.0, 0.01))
    
    # 更新门：
    zx = tf.Variable(tf.truncated_normal([vocabulary_size, num_nodes], stddev=0.01))
    zh = tf.Variable(tf.truncated_normal([num_nodes, num_nodes], stddev=0.01))
    zb = tf.Variable(tf.random_uniform([1, num_nodes],0.0, 0.01))
    
    # Softmax 分类器的权重和偏置
    w = tf.Variable(tf.truncated_normal([num_nodes, vocabulary_size], stddev=0.01))
    b = tf.Variable(tf.random_uniform([vocabulary_size],0.0,0.01))
    
    # 展开时保存的变量
    saved_output = tf.Variable(tf.zeros([batch_size, num_nodes]), trainable=False)
    saved_valid_output = tf.Variable(tf.zeros([1, num_nodes]),trainable=False)
    saved_test_output = tf.Variable(tf.zeros([1, num_nodes]),trainable=False)
    
    algorithm = 'gru'
    filename_to_save = algorithm + filename_extension +'.csv'
    
    # 单元计算的定义：
    def gru_cell(i, o):
    
        # 建立一个 GRU 单元
        reset_gate = tf.sigmoid(tf.matmul(i, rx) + tf.matmul(o, rh) + rb)
        h_tilde = tf.tanh(tf.matmul(i,hx) + tf.matmul(reset_gate * o, hh) + hb)
        z = tf.sigmoid(tf.matmul(i,zx) + tf.matmul(o, zh) + zb)
        h = (1-z)*o + z*h_tilde
    
        return h
    

### LSTM、Peephole、GRU 的应用

接下来我们分别将这三个模型应用于文本生成的任务上，大家应该还记得我们在《用多层 RNN
建立语言模型》这篇文章中讲到了如何生成莎士比亚风格的文章，在这里我们将使用同样的数据，数据预处理部分的代码也和原来一样，区别在于模型部分，所以代码含义就不赘述，主要比较模型。

#### **导入 TensorFlow 和其他库**

    
    
    from __future__ import absolute_import, division, print_function, unicode_literals
    
    try:
      # %tensorflow_version 仅存在于 Colab
      %tensorflow_version 2.x
    except Exception:
      pass
    import tensorflow as tf
    
    import numpy as np
    import os
    import time
    

#### **导入数据**

数据也是同样的文本，导入后取 16246 之后的文本，因为前面是版权声明等文字。

    
    
    with open('pg2265.txt', 'r', encoding='utf-8') as f: 
        text=f.read()
    
    text = text[16246:]
    

#### **建立字典**

这一步要做的也和 TF1 中的一样，先创建两个索引字典，然后将文本转化为数字。

    
    
    # 取文本中的非重复字符
    vocab = sorted(set(text))
    print ('{} unique characters'.format(len(vocab)))    # 65 unique characters, eg: ['\n', ' ', '!', '&', "'",...]
    
    # 创建从非重复字符到索引的映射
    char2idx = {u:i for i, u in enumerate(vocab)}        # length is 65, eg: {'\n': 0, ' ': 1, '!': 2, '&': 3, "'": 4}
    # 直接用array的序号与char对应
    idx2char = np.array(vocab)                            # eg: array(['\n', ' ', '!', '&', "'"], dtype='<U1')
    
    # 将文本 text 转化为数字序列
    text_as_int = np.array([char2idx[c] for c in text])    # length is  162849, eg: array([32, 46, 43,  1, 32, 56, 39, 45, 43, 42])
    
    # 可以看一下文本前 10 个字符被映射成了什么样的整数序列：
    print ('{} ---- characters mapped to int ---- > {}'.format(repr(text[:10]), text_as_int[:10]))
    
    # 'The Traged' ---- characters mapped to int ---- > [32 46 43  1 32 56 39 45 43 42]
    

#### **创建训练数据**

    
    
    # 设定每个输入句子长度的最大值
    seq_length = 100   
    examples_per_epoch = len(text)//seq_length  
    

这里我们用 tf.data.Dataset 对数据进行各种处理。

    
    
    # 创建训练样本 / 目标
    char_dataset = tf.data.Dataset.from_tensor_slices(text_as_int)
    

可以打印出 dataset 的前几个数据看一下：

    
    
    for i in char_dataset.take(5):
      print(i.numpy())
    

> 32 46 43 1 32

然后用 batch 方法将单个字符转换为所需长度的序列：

    
    
    sequences = char_dataset.batch(seq_length+1, drop_remainder=True)    # 最后剩余的忽略
    

就是原本在 char_dataset 中是单个数字的形式，用 batch 将 char_dataset 数据分成了每句话长度为
`seq_length+1=101` 的 sequences：

    
    
    for item in sequences.take(2):
      print(item.numpy())
    

> [32 46 43 1 32 56 39 45 43 42 47 43 1 53 44 1 21 39 51 50 43 58 0 0 14 41 58
> 59 57 1 28 56 47 51 59 57 9 1 31 41 53 43 52 39 1 28 56 47 51 39 9 0 0 18 52
> 58 43 56 1 15 39 56 52 39 56 42 53 1 39 52 42 1 19 56 39 52 41 47 57 41 53 1
> 58 61 53 1 16 43 52 58 47 52 43 50 57 9 0 0 1 1 15] [39 56 52 39 56 42 53 9
> 1 34 46 53 4 57 1 58 46 43 56 43 13 0 1 1 19 56 39 52 9 1 26 39 63 1 39 52
> 57 61 43 56 1 51 43 11 1 31 58 39 52 42 1 3 1 60 52 44 53 50 42 0 63 53 59
> 56 1 57 43 50 44 43 0 0 1 1 1 15 39 56 9 1 24 53 52 45 1 50 47 59 43 1 58 46
> 43 1 23 47 52 45 0 0 1]

接下来构造 x 和 y，同样的，y 和 x 之间需要错位一个字符，例如输入序列为“Hell”，则目标序列为“ello”。

这里可以直接用 map 对 sequences 的每个序列执行 split_input_target 函数，split_input_target
就是生成错位的 x 和 y：

    
    
    def split_input_target(chunk):
        input_text = chunk[:-1]
        target_text = chunk[1:]
        return input_text, target_text
    
    dataset = sequences.map(split_input_target)
    

现在 dataset 的每条数据包括一个输入序列和输出序列，二者之间错位一个数字：

    
    
    for input_example, target_example in  dataset.take(1):
      print ('Input data: ', input_example.numpy())
      print ('Target data:', target_example.numpy())
    
    
    
    Input data:  [32 46 43  1 32 56 39 45 43 42 47 43  1 53 44  1 21 39 51 50 43 58  0  0
     14 41 58 59 57  1 28 56 47 51 59 57  9  1 31 41 53 43 52 39  1 28 56 47
     51 39  9  0  0 18 52 58 43 56  1 15 39 56 52 39 56 42 53  1 39 52 42  1
     19 56 39 52 41 47 57 41 53  1 58 61 53  1 16 43 52 58 47 52 43 50 57  9
      0  0  1  1]
    Target data: [46 43  1 32 56 39 45 43 42 47 43  1 53 44  1 21 39 51 50 43 58  0  0 14
     41 58 59 57  1 28 56 47 51 59 57  9  1 31 41 53 43 52 39  1 28 56 47 51
     39  9  0  0 18 52 58 43 56  1 15 39 56 52 39 56 42 53  1 39 52 42  1 19
     56 39 52 41 47 57 41 53  1 58 61 53  1 16 43 52 58 47 52 43 50 57  9  0
      0  1  1 15]
    

#### **构造批次数据**

先用 shuffle 将数据打乱，然后再用 batch 构造批次数据。

    
    
    # 批大小
    BATCH_SIZE = 64
    
    # 设定缓冲区大小，以重新排列数据集 
    BUFFER_SIZE = 10000
    
    dataset = dataset.shuffle(BUFFER_SIZE).batch(BATCH_SIZE, drop_remainder=True)
    
    dataset        # <BatchDataset shapes: ((64, 100), (64, 100)), types: (tf.int64, tf.int64)>
    

我们可以看看此时 dataset 的第一条数据，它的维度是 `(64, 100), (64, 100)`，即 Input 和 Output 的每一条维度都是
(64, 100)，可见再用一次 batch 后，就将 64 个 sentense 组成了一个 batch，非常方便。

    
    
    for input_example, target_example in  dataset.take(1):
      print ('Input data: ', input_example.numpy())
      print ('Target data:', target_example.numpy())
    
    
    
    Input data:  [[56 41 46 ... 47 52 45]
     [ 1 29 59 ... 16 59 41]
     [41 39 58 ... 57 53 54]
     ...
     [39 52 42 ...  7  1 39]
     [ 0 58 47 ...  1 21 39]
     [39  1 60 ... 46 39 50]]
    Target data: [[41 46 39 ... 52 45  1]
     [29 59  9 ... 59 41 49]
     [39 58 43 ... 53 54 46]
     ...
     [52 42  1 ...  1 39 52]
     [58 47 50 ... 21 39 51]
     [ 1 60 47 ... 39 50 50]]
    

#### **文本生成辅助函数**

文本生成时，首先设置起始字符串，初始化 RNN 状态并设置要生成的字符个数，根据起始字符串和 RNN
的初始状态，模型可以得到下一个字符的预测分布，然后用这个类别分布计算预测出来的字符的索引，并将预测字符和前面的隐藏状态一起传递给模型作为下一个输入，就这样不断从前面预测的字符获得更多上下文，进行学习。

    
    
    def generate_text(model, start_string):
      # 评估步骤（用学习过的模型生成文本）
    
      # 要生成的字符个数
      num_generate = 1000
    
      # 将起始字符串转换为数字（向量化）
      input_eval = [char2idx[s] for s in start_string]
      input_eval = tf.expand_dims(input_eval, 0)
    
      # 空字符串用于存储结果
      text_generated = []
    
      # 低温度会生成更可预测的文本
      # 较高温度会生成更令人惊讶的文本
      # 可以通过试验以找到最好的设定
      temperature = 1.0
    
      # 这里批大小为 1
      model.reset_states()
      for i in range(num_generate):
          predictions = model(input_eval)
          # 删除批次的维度
          predictions = tf.squeeze(predictions, 0)
    
          # 用分类分布预测模型返回的字符
          predictions = predictions / temperature
          predicted_id = tf.random.categorical(predictions, num_samples=1)[-1,0].numpy()
    
          # 把预测字符和前面的隐藏状态一起传递给模型作为下一个输入
          input_eval = tf.expand_dims([predicted_id], 0)
    
          text_generated.append(idx2char[predicted_id])
    
      return (start_string + ''.join(text_generated))
    

#### **设置模型参数**

    
    
    # 词集的长度
    vocab_size = len(vocab)
    
    # 嵌入的维度
    embedding_dim = 256
    
    # RNN 的单元数量
    rnn_units = 128
    

前面都是准备工作，接下来进入三个模型的构造和训练。

#### **建立模型 LSTM**

这里用 tf.keras.layers.LSTM 建立 LSTM 单元：

    
    
    def build_model(vocab_size, embedding_dim, rnn_units, batch_size):
      model = tf.keras.Sequential([
        tf.keras.layers.Embedding(vocab_size, embedding_dim,
                                  batch_input_shape=[batch_size, None]),
    
        tf.keras.layers.LSTM(rnn_units,
                            return_sequences=True,
                            stateful=True,
                            recurrent_initializer='glorot_uniform'),
        tf.keras.layers.Dropout(0.5),
        tf.keras.layers.Dense(vocab_size)
      ])
      return model
    
    
    model = build_model(
      vocab_size = len(vocab),
      embedding_dim=embedding_dim,
      rnn_units=rnn_units,
      batch_size=BATCH_SIZE)
    
    model.summary()
    

![](https://images.gitbook.cn/671d7730-6a00-11ea-af72-7dcc88a7b3ca)

定义损失函数，配置模型，用 model.fit 训练模型：

    
    
    def loss(labels, logits):
      return tf.keras.losses.sparse_categorical_crossentropy(labels, logits, from_logits=True)
    
    model.compile(optimizer=tf.keras.optimizers.Adam(lr=0.001, clipvalue=5),
                  loss=loss)
    
    # 检查点保存至的目录
    checkpoint_dir = './training_checkpoints'
    
    # 检查点的文件名
    checkpoint_prefix = os.path.join(checkpoint_dir, "ckpt_{epoch}")
    
    checkpoint_callback=tf.keras.callbacks.ModelCheckpoint(
        filepath=checkpoint_prefix,
        save_weights_only=True)
    
    EPOCHS=10
    history = model.fit(dataset, epochs=EPOCHS, callbacks=[checkpoint_callback])
    

![](https://images.gitbook.cn/7fb11f40-6a00-11ea-b329-5bafbb4d9ea6)

以 `Ham:` 为开头生成文本：

    
    
    # 恢复最新的检查点
    tf.train.latest_checkpoint(checkpoint_dir)
    
    # 为了让预测步骤简单，将批次大小设定为 1。
    model = build_model(vocab_size, embedding_dim, rnn_units, batch_size=1)
    
    model.load_weights(tf.train.latest_checkpoint(checkpoint_dir))
    
    model.build(tf.TensorShape([1, None]))
    
    model.summary()
    
    print(generate_text(model, start_string=u"Ham: "))
    
    
    
    Ham: [youme it Ongeere,
    Gare crwirtCry thringone you eidpis, ond she-vytis Do
    Ainf so han, what shathe tore thee 'nged vfondgy in at but.
    Add the hy Loke Gore vr, frerd lliod
    
       Hoou I, wing
    
        Hat. Thal. jos my ay wa loulsst, Kisu thans wime Ioudt o tloplakes Katn as, sher o atind Weawbetteike yot vnmy,
    
    Whoot as as senot let as goy erpel Illlttigh,
    The Repo hich sttel?
       Ha. The bin on my ouncd, ke fitht im. CaRteme is yourd
    And ays hin on mamesar.
    SHiormuy bouere of eto pindd bare duofich Qe weeks Mawebuaseiccit:
    Athuss;
    Aaml's cerdter.
    As whay
    Axnnouer
    Luad to Lodned your?
    Eruew I frit incike
    yout to weirs Samen, Gile,
    Sos you the Ice willj'd the
    
       Hat. Ie that thiterd
    Lerkela ad this led: bome mmar asy wh shatile,
    Osit Foure,
    The brigeade dighe, I youl haw dewelce. The duo hist, ofderikh,
      Hor. A hpore shoue, vlod ig il dy mlaes my freuat what I awo ry Pewin, th ad souin, Serace afrase, youj: And ere all but dry theaMrat;
    Bane thon mat tondathese sor batfe whou ware rad halpe ro
    

#### **建立模型 GRU**

最简单的方式是只需要将 tf.keras.layers.LSTM 换成 tf.keras.layers.GRU 即可：

    
    
    def build_model_gru(vocab_size, embedding_dim, rnn_units, batch_size):
      model = tf.keras.Sequential([
        tf.keras.layers.Embedding(vocab_size, embedding_dim,
                                  batch_input_shape=[batch_size, None]),
        # tf.keras.layers.LSTM
        tf.keras.layers.GRU(rnn_units,
                            return_sequences=True,
                            stateful=True,
                            recurrent_initializer='glorot_uniform'),
        tf.keras.layers.Dropout(0.5),
        tf.keras.layers.Dense(vocab_size)
      ])
      return model
    
    
    model_gru = build_model_gru(
      vocab_size = len(vocab),
      embedding_dim=embedding_dim,
      rnn_units=rnn_units,
      batch_size=BATCH_SIZE)
    
    model_gru.summary()
    

![](https://images.gitbook.cn/b5db16c0-6a00-11ea-94ab-4f995a3d6ed9)

定义损失函数，配置模型，用 model.fit 训练模型：

    
    
    def loss(labels, logits):
      return tf.keras.losses.sparse_categorical_crossentropy(labels, logits, from_logits=True)
    
    model_gru.compile(optimizer=tf.keras.optimizers.Adam(lr=0.001, clipvalue=5),
                  loss=loss)
    
    # 检查点保存至的目录
    checkpoint_dir = './gru_training_checkpoints'
    
    # 检查点的文件名
    checkpoint_prefix = os.path.join(checkpoint_dir, "ckpt_{epoch}")
    
    checkpoint_callback=tf.keras.callbacks.ModelCheckpoint(
        filepath=checkpoint_prefix,
        save_weights_only=True)
    
    EPOCHS=10
    history_gru = model_gru.fit(dataset, epochs=EPOCHS, callbacks=[checkpoint_callback])
    

![](https://images.gitbook.cn/ca0d6080-6a00-11ea-af72-7dcc88a7b3ca)

以 `Ham:` 为开头生成文本：

    
    
    # 恢复最新的检查点
    tf.train.latest_checkpoint(checkpoint_dir)
    
    # 为了让预测步骤简单，将批次大小设定为 1。
    model_gru = build_model_gru(vocab_size, embedding_dim, rnn_units, batch_size=1)
    
    model_gru.load_weights(tf.train.latest_checkpoint(checkpoint_dir))
    
    model_gru.build(tf.TensorShape([1, None]))
    
    model_gru.summary()
    
    print(generate_text(model_gru, start_string=u"Ham: "))
    
    
    
    Ham: hrats son: I lot lowinkes,
    Ance ar zegarente
    StIrr, he, os then hyou aund roue  gporencinesbferssooll wies it efwild. Maunce thin mard ouprh Londue pelenderigellEakn my thim vpaut gouy Oee bey thor so Wim. It in ane ind Sofeam. In taur ierand, whand There,
    Fit thie ifeune ie feuro
    
      tidnt,
    Whe mrod'jmar)
    Bnor heauia ke turhe hloadt cort,
    Ardiagped,
    The thou hatke hir sarbulle 
    Phaige,
    '
     OrKid pree
    Buedse catee deustectbe.
    Sigeareefe, chem, theurd dore
    
       Ham. Momingher poin-wer
    O mes. Dout, ot vstthit ily deareI Ghald fime noue
    
      Par. Cold it;
    Ou ting hely, me math anr of Is: vame four,
    
       Meosreraue,
    pore bie in, thamn il selowinglithe sarr the amy Himesmaw
    thaue?
       Har.. Thes My. The krarw may wasme doue ve:, and hate hort whis dueden gesite you I ta sreuih bereo: ons
    Be ance ingo bus tour fseay, of Thee no cuigod 'uwer as wing Sollet rold los ath. In the Larlith for, of Golde fits ee dere mow whe tho bameing nim Bese
    me Mitheadut.
    I  vrts's
    
       Pmmalis
    
       Kim1n whith Caked vn
    

#### **建立模型 Peephole LSTM**

这里我们可以使用 Keras 内置的 Peephole 模型
tf.keras.experimental.PeepholeLSTMCell(rnn_units) 就可以轻松地搭建起来：

    
    
    def build_model_peephole(vocab_size, embedding_dim, rnn_units, batch_size):
      model = tf.keras.Sequential([
        tf.keras.layers.Embedding(vocab_size, embedding_dim,
                                  batch_input_shape=[batch_size, None]),
    
        # tf.keras.layers.GRU(rnn_units,
        #                     return_sequences=True,
        #                     stateful=True,
        #                     recurrent_initializer='glorot_uniform'),
        tf.keras.layers.RNN(tf.keras.experimental.PeepholeLSTMCell(rnn_units),
                            return_sequences=True,
                            stateful=True),
    
        tf.keras.layers.Dropout(0.5),
        tf.keras.layers.Dense(vocab_size)
      ])
      return model
    
    
    model_peephole = build_model_peephole(
      vocab_size = len(vocab),
      embedding_dim=embedding_dim,
      rnn_units=rnn_units,
      batch_size=BATCH_SIZE)
    
    model_peephole.summary()
    

![](https://images.gitbook.cn/f89db210-6a00-11ea-af72-7dcc88a7b3ca)

定义损失函数，配置模型，用 model.fit 训练模型：

    
    
    def loss(labels, logits):
      return tf.keras.losses.sparse_categorical_crossentropy(labels, logits, from_logits=True)
    
    model_peephole.compile(optimizer=tf.keras.optimizers.Adam(lr=0.001, clipvalue=5),
                  loss=loss)
    
    # 检查点保存至的目录
    checkpoint_dir = './peephole_training_checkpoints'
    
    # 检查点的文件名
    checkpoint_prefix = os.path.join(checkpoint_dir, "ckpt_{epoch}")
    
    checkpoint_callback=tf.keras.callbacks.ModelCheckpoint(
        filepath=checkpoint_prefix,
        save_weights_only=True)
    
    EPOCHS=10
    history_peephole = model_peephole.fit(dataset, epochs=EPOCHS, callbacks=[checkpoint_callback])
    

![](https://images.gitbook.cn/1032aed0-6a01-11ea-8181-9bde95dea606)

以 `Ham:` 为开头生成文本：

    
    
    # 恢复最新的检查点
    tf.train.latest_checkpoint(checkpoint_dir)
    
    # 为了让预测步骤简单，将批次大小设定为 1。
    model_peephole = build_model_peephole(vocab_size, embedding_dim, rnn_units, batch_size=1)
    
    model_peephole.load_weights(tf.train.latest_checkpoint(checkpoint_dir))
    
    model_peephole.build(tf.TensorShape([1, None]))
    
    model_peephole.summary()
    
    print(generate_text(model_peephole, start_string=u"Ham: "))
    
    
    
    Ham: kt . or muet Perar im 
     re yoris bur
     son ter me
    
     I  olu mardmy lynet oue nouxie f: cres ho rod,
    Te noid pouenu Inenlen le vrot wau. sithe, wond,
    
      Hope
    Deefstene wheel you : sout,
    S foutd, thalet Qucpss, ) istemake treqrale theer
    Hray why
    Ir pald, ucde huber daate feratheris har male Hacttnusg?
    
       HaIriet the wither,
    mirssts,
    Thil,
    hit e yomre theul woantereingh, ind you s coreicdes:
    
      poVg. Nold cutes Bo Duert the oth th iy fy sen tisge
    
       itre hat hike I with mutng, p
    Bomre dos af mea. me this t ou yindes In whe anpane may egh forew
    
       Hae. Buee sir arberey,;
    I eod citurel.
    Sood Nouescrer's
    -loume Iun ae Hou kot tits theeEpf eour ommle am yousd or. Shiches, cerue
    Terest in
    , uraupturt
    Ccog tham ingaypes
    
    h nemed Hay the mad, Gourt.
     VasFiies, hiy speacitg, angen tutererant, te of pord ckits af of are wicQeu nererd,
    it a diriert dond bay chakl If pise Goy th's
    Thorn  Hathe Furme bysenn? thos mind,
    Th the, Enoond Frperius te vnes ore,
    Bedeane he wioret youp I, ind maker
    Bengathe
    

#### **最后可以打印出三个模型的训练损失**

    
    
    import matplotlib.pyplot as plt
    %matplotlib inline
    
    # summarize history for loss
    plt.plot(history_peephole.history['loss'])
    plt.plot(history_gru.history['loss'])
    plt.plot(history.history['loss'])
    plt.title('model loss')
    plt.ylabel('loss')
    plt.xlabel('epoch')
    plt.legend(['Peephole', 'GRU', 'LSTM'], loc='upper right')
    plt.show()
    

![](https://images.gitbook.cn/3a702100-6a01-11ea-b31d-2b61fbcda176)

在这个小例子上，我们可以看出 GRU 的表现要比 LSTM 好一些，Peephole 虽然是对 LSTM 的改进，但是在这个例子上却没有 LSTM
表现的好，所以不同的模型会适用于不同的数据量和任务上，大家在使用时可以多多尝试。

* * *

参考文献：

  * Thushan Ganegedara, _Natural Language Processing with TensorFlow_

