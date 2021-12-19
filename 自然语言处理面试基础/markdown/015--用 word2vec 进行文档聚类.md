在前面几篇文章中我们学习了 word2vec 的两种模型 CBOW 和 Skip-Gram，了解了它们的基本原理，数学思想，还有代码实现。

**word2vec 有很多优点**
，它的概念比较容易理解，训练速度快，既适用于小数据集也适用于大数据集，在捕获语义相似性方面做得很好，而且是一种非监督式学习方法，不需要人类标记数据。

**当然也有一些不足** ，如虽然易于开发，但难以调试；如果一个单词有多种含义，嵌入向量只能反映出它们的平均值。

今天我们来进一步学习 word2vec 的应用。

* * *

### word2vec 的应用

**word2vec 有很多应用场景** ，因为它可以捕获语义相似性，因此当我们遇见涉及分布式语义的任务时，就可以用它来生成特征，输入到各种算法模型中。

  * 例如在依存解析任务中，使用 word2vec 可以生成单词之间更好更准确的依赖关系。
  * 命名实体识别任务中，因为 word2vec 非常擅长找出相似的实体，可以把相似的聚集在一起，获得更好的结果。
  * 情感分析中，使用 word2vec 来保持语义相似性，可以产生更好的情绪结果，因为语义相似性能够帮助我们了解人们一般会使用哪些短语或单词来表达自己什么样的观点。
  * 在文档分类任务中，也可以用 word2vec 省去人工标签。
  * 谷歌也使用 word2vec 来改进他们的机器翻译产品。
  * 此外在自动摘要、语音识别、自动问答、音乐视频推荐系统等很多任务中也有广泛的应用。

这里我们将学习如何将 word2vec 用于文档分类/聚类任务中。

### 文档分类

**文档分类，是指给文档分配一个或者多个类别标签。** 在我们的日常生活中，文档分类任务有很多应用场景，例如：

  * 过滤垃圾邮件，自动识别电子邮件中哪些是垃圾邮件；
  * 邮件路径自动化，根据邮件的主题将其发送到特定地址或者邮箱；
  * 可读性推荐，例如为不同年龄的或不同兴趣类型的读者找到合适的材料；
  * 情感分析，确定说话主体关于某个内容的态度；
  * 识别文档的领域是属于体育、金融还是科技等等。

可以说需要给文档数据识别类别的任务都属于文档分类，处理文档分类的方法也有很多种，大体可以划分为两大类：监督式学习和非监督式学习方法。

在 **监督式学习** 中，类别是提前定义好的，训练集中的文档需要手动打上标签，然后在训练集上训练分类器模型。主要流程为：

**1\. 建立数据集**

数据集需要足够大，例如有 500 种文档类别，那么每个类别可能需要 100 个文档。文档还需要有足够高的质量，以便区分不同类别中的文档之间的差异。

**2\. 预处理**

这一步常用的步骤有将文本数据进行大小写转化、分词、转化成数字向量等，再构建一些特征，可以用词向量，如 word2vec、GloVe、FastText 等。

**3\. 算法**

有很多种算法都可以用来做文档分类，如：

  * 逻辑回归
  * 朴素贝叶斯分类器
  * 支持向量机
  * KNN 近邻
  * 决策树
  * Ensemble 方法：随机森林、Gradient boosting、Adaboost
  * 一般的前馈神经网络
  * 还有我们现在学的循环神经网络：LSTM、GRU
  * 还可以用 seq2seq 加上注意力机制
  * 除了用 RNN，还可以用 CNN，以及 RCNN
  * 还有目前很火的 Transformer、BERT 等模型，在后面的章节中我们会讲到。

**非监督式学习** 用于文档数据也叫做文档聚类，不需要给数据打标签，例如找到相似的新闻/微博，分析顾客的商品评价，挖掘文档中有意义的隐式主题等。

常用的方法有两种，一种是基于层级的算法，会通过聚集或划分等步骤将文档表示为适合浏览的层次结构，可以学习出更深入的信息和细节，不过通常效率不高。另一种方法是
K-means 及其拓展算法，这种就比较高效。

### Doc2vec

我们知道 word2vec
的作用主要是学习出单词的嵌入向量表示作为单词的特征，在后面的文档分类实战中，我们会用它来学习出文档中每个单词的词向量，再用文档中的所有词向量取平均值作为文档的嵌入表示，其实
word2vec 的思想还可以扩展到不同级别的文本，例如段落级别，或者直接到达文档级别。

**Doc2Vec 就是 word2vec 的扩展** ，也叫做 paragraph2vec 或者 sentence embeddings，是在 [2014
年由 Quoc Le 和 Tomas Mikolov](https://arxiv.org/pdf/1405.4053.pdf) 提出的。这个算法是在
Word2vec 的基础上做出的改进，所以思路是差不多的， **Doc2Vec 目的是要得到句子或者文档的向量表示。**

和 word2vec 类似，Doc2Vec 也有两种模型：一种是 **DM：Distributed Memory，是给定上下文和文档向量，预测单词的概率**
，这个模型和 word2vec 的CBOW 很像。

![](https://images.gitbook.cn/ebf5c950-5efa-11ea-a753-95551dab5dce)

如图所示，DM 模型的输入是每个文档 ID 和语料库中所有词的向量，每个段落或句子可以用矩阵 D 的一列来表示，每个单词用矩阵 W
的一列来表示，然后取平均值或者直接拼接成一个向量，再输出下一个单词的 softmax 概率。

DM 与 word2vec
的区别是在输入中除了单词外，还输入了代表文档的节点，这个节点的作用是代表这个段落或篇章的主题。它将文档看作为另一种上下文，在抽象意义上单词和文档之间没有区别，它的目标函数和训练步骤也和
word2vec 一样。

另一种是 **DBOW：Distributed Bag of Words，即给定文档向量，预测文档中一组随机单词的概率** ，这和 word2vec 的
Skip-Gram 很像。此模型的输入是文档的向量，预测的是该文档中随机抽样的词。

![](https://images.gitbook.cn/10170dd0-5efb-11ea-a6e8-55c7d85fdb0f)

Doc2vec 也可以用于文档聚类，情感分析，推荐系统等任务，在学习完下面的实战后，如果对 Doc2vec 感兴趣可以尝试应用。

### 用 word2vec 进行文档聚类

在这个实战中我们将对文档进行聚类，整体思路是，首先用所有文本数据学习出各单词的词向量，然后用文档中出现单词的词向量平均值作为文档的嵌入表示，最后用
K-means 算法对文档的嵌入向量进行聚类，找到相近的文档。所以 word2vec
在这个任务的作用是在学习出表示文档的特征，然后输入给机器学习算法中得到预测结果。

代码的整体流程和前面的 CBOW、Skip-Gram、GloVe 基本一致，主要有四大步骤：

数据：

1\. 导入库  
2\. 下载数据  
3\. 数据预处理  
4\. 建立字典  
5\. 生成批次数据

建立并训练 CBOW 模型：

6\. 设置超参数  
7\. 定义输入输出  
8\. 定义模型参数  
9\. 定义模型  
10\. 定义损失和优化算法  
11\. 定义相似度  
12\. 模型训练

可视化：

13\. 将文档嵌入结果可视化

对文档进行聚类：

14\. K-means

#### **1\. 导入所需要的库**

    
    
    %matplotlib inline
    from __future__ import print_function
    import collections
    import math
    import numpy as np
    import os
    import random
    import tensorflow as tf
    import zipfile
    from matplotlib import pylab
    from six.moves import range
    from six.moves.urllib.request import urlretrieve
    from sklearn.manifold import TSNE
    from sklearn.cluster import KMeans
    import nltk             # 用于预处理
    nltk.download('punkt')
    import operator         # 用于对字典里的项目进行排序
    from math import ceil
    

#### **2\. 下载数据**

我们所使用的数据是 BBC 数据集，这个数据集由 D. Greene 和 P. Cunningham 创建，包括 2225
个新闻文档，分别属于五大类：商业、娱乐、科技、体育和政治。每一类的文档保存在各自的文件夹中，每个文档用数字编号，如：'bbc/tech/211.txt'。各类所包含的文档数目为：体育类
511、商业类 510、政治类 417、科技类 401、娱乐类
386。数据可以通过下面的代码获得，或者在[这里](https://www.kaggle.com/yufengdev/bbc-fulltext-and-
category)下载 CSV 格式的数据：

    
    
    url = 'http://mlg.ucd.ie/files/datasets/'
    
    def maybe_download(filename, expected_bytes):
      """Download a file if not present, and make sure it's the right size."""
      if not os.path.exists(filename):
        filename, _ = urlretrieve(url + filename, filename)
      statinfo = os.stat(filename)
      if statinfo.st_size == expected_bytes:
        print('Found and verified %s' % filename)
      else:
        print(statinfo.st_size)
        raise Exception(
          'Failed to verify ' + filename + '. Can you get to it with a browser?')
      return filename
    
    filename = maybe_download('bbc-fulltext.zip', 2874078)
    

#### **3\. 读取数据，并用 NLTK 进行预处理**

read_data 是要从 `['business','entertainment','politics','sport','tech']`
的每一类中分别读取 249 个文档，将文本内容进行解码，转化成小写，分词等简单的预处理，建立 data。

    
    
    def read_data(filename):
      data = []
      files_to_read_for_topic = 250
      topics = ['business','entertainment','politics','sport','tech']
      with zipfile.ZipFile(filename) as z:
        parent_dir = z.namelist()[0]
        for t in topics:
            print('\tFinished reading data for topic: ',t)
            for fi in range(1,files_to_read_for_topic):
                with z.open(parent_dir + t + '/'+ format(fi,'03d')+'.txt') as f:
                    file_string = f.read().decode('latin-1')
                    file_string = file_string.lower()
                    file_string = nltk.word_tokenize(file_string)
                    data.extend(file_string)
        return data
    

read_test_data 的作用和前者是一样的，只不过是从前面每一类读取的 249 个文档中各自随机选择 10 个文档作为测试集，所以测试数据有 50
个文档，和 data 不同的是 test_data 是个字典，key 是类别—文档编号，value 是该文档的分词序列，如：

    
    
    'sport-221': ['lennon', 'brands', 'rangers', 'favourites', 'celtic', "'s", 'neil', ...
    
    
    
    def read_test_data(filename):
      test_data = {}
      files_to_read_for_topic = 250
      topics = ['business','entertainment','politics','sport','tech']
      with zipfile.ZipFile(filename) as z:
        parent_dir = z.namelist()[0]
        for t in topics:
            print('\tFinished reading data for topic: ',t)
    
            for fi in np.random.randint(1,files_to_read_for_topic,(10)).tolist():
                with z.open(parent_dir + t + '/'+ format(fi,'03d')+'.txt') as f:
                    file_string = f.read().decode('latin-1')
                    file_string = file_string.lower()
                    file_string = nltk.word_tokenize(file_string)
                    test_data[t+'-'+str(fi)] = file_string                
        return test_data
    
    
    
    print('Processing training data...')
    words = read_data(filename)
    print('\nProcessing testing data...')
    test_words = read_test_data(filename)
    
    print('Example words (start): ',words[:10])
    print('Example words (end): ',words[-10:])
    
    
    
    Example words (start):  ['ad', 'sales', 'boost', 'time', 'warner', 'profit', 'quarterly', 'profits', 'at', 'us']
    Example words (end):  ['almost', '200,000', 'people', 'are', 'registered', 'players', 'on', 'project', 'entropia', '.']
    

最后我们得到的 words 训练数据集大小为 521827，test_words 为 50 个文档组成的测试数据集。

#### **4\. 建立字典**

这一步的 build_dataset 和前面几篇词向量中所用都是一样的，目的就是将训练集和测试集的文本数据都转化为数字序列。

假设我们输入的文本是 "I like to go to school"：

  * dictionary：将单词映射为 ID，例如 {I:0, like:1, to:2, go:3, school:4}
  * reverse_dictionary：将 ID 映射为单词，例如 {0:I, 1:like, 2:to, 3:go, 4:school}
  * count：计数每个单词出现了几次，例如 [(I,1),(like,1),(to,2),(go,1),(school,1)]
  * data：将读取的文本数据转化为 ID 后的数据列表，例如 [0, 1, 2, 3, 2, 4] 此外，我们用 UNK 代表比较少见的单词。

多了一个 build_dataset_with_existing_dictionary 用来将单词串根据已有的字典转化为 ID 串。

    
    
    # 将词汇表大小限制在 25000
    vocabulary_size = 25000                                                             
    
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
    
    
    # 根据已有的字典将单词串转化为 ID
    def build_dataset_with_existing_dictionary(words, dictionary):
        data = list()
        for word in words:
            if word in dictionary:
              index = dictionary[word]
            else:
              index = 0  # dictionary['UNK']
            data.append(index)
        return data
    
    # 得到训练数据
    data, count, dictionary, reverse_dictionary = build_dataset(words)
    
    # 得到测试数据
    test_data = {}
    for k,v in test_words.items():
        print('Building Test Dataset for ',k,' topic')
        test_data[k] = build_dataset_with_existing_dictionary(test_words[k],dictionary)
    
    print('Most common words (+UNK)', count[:5])
    print('Sample data', data[:10])
    print('test keys: ',test_data.keys())
    del words  
    del test_words
    

#### **5\. 生成批次数据**

在这个例子中，因为数据量不是特别大，我们用 CBOW 比 Skip-Gram 的效果会好一些，所以就用 CBOW 来训练。generate_batch 和
CBOW 那篇文章中是一样的，就是要构造模型所需要的数据格式，即由上下文得到中心词。

    
    
    data_index = 0
    
    def generate_batch(data, batch_size, window_size):
    
        # 每次读取一个数据集后增加 1
        global data_index
    
        # span 是指中心词及其两侧窗口的总长度
        span = 2 * window_size + 1 
    
        # batch 为上下文数据集，一共有 span - 1 个列
        # labels 为中心词
        batch = np.ndarray(shape=(batch_size,span-1), dtype=np.int32)
        labels = np.ndarray(shape=(batch_size, 1), dtype=np.int32)
    
        # buffer 用来存储 span 内部的数据
        buffer = collections.deque(maxlen=span)
    
        # 填充 buffer 更新 data_index
        for _ in range(span):
            buffer.append(data[data_index])
            data_index = (data_index + 1) % len(data)
    
        # 批次读取数据，对于每一个 batch index 遍历 span 的所有元素，填充到 batch 的各个列中
        for i in range(batch_size):
            target = window_size                  # 在 buffer 的中心是中心词
            target_to_avoid = [ window_size ]     # 因为只需要考虑中心词的上下文，这时不需要考虑中心词
    
            # 将选定的中心词加入到 avoid_list 下次时用
            col_idx = 0
            for j in range(span):
                # 建立批次数据时忽略中心词
                if j==span//2:
                    continue
                batch[i,col_idx] = buffer[j] 
                col_idx += 1
            labels[i, 0] = buffer[target]
    
            # 每次读取一个数据点，需要移动 span 一次，建立一个新的 span
            buffer.append(data[data_index])
            data_index = (data_index + 1) % len(data)
    
        assert batch.shape[0]==batch_size and batch.shape[1]== span-1
        return batch, labels
    
    for window_size in [1,2]:
        data_index = 0
        batch, labels = generate_batch(data, batch_size=8, window_size=window_size)
        print('\nwith window_size = %d:' % (window_size))
        print('    batch:', [[reverse_dictionary[bii] for bii in bi] for bi in batch])
        print('    labels:', [reverse_dictionary[li] for li in labels.reshape(8)])
    
    
    
    with window_size = 1:
        batch: [['ad', 'boost'], ['sales', 'time'], ['boost', 'warner'], ['time', 'profit'], ['warner', 'quarterly'], ['profit', 'profits'], ['quarterly', 'at'], ['profits', 'us']]
        labels: ['sales', 'boost', 'time', 'warner', 'profit', 'quarterly', 'profits', 'at']
    
    with window_size = 2:
        batch: [['ad', 'sales', 'time', 'warner'], ['sales', 'boost', 'warner', 'profit'], ['boost', 'time', 'profit', 'quarterly'], ['time', 'warner', 'quarterly', 'profits'], ['warner', 'profit', 'profits', 'at'], ['profit', 'quarterly', 'at', 'us'], ['quarterly', 'profits', 'us', 'media'], ['profits', 'at', 'media', 'giant']]
        labels: ['boost', 'time', 'warner', 'profit', 'quarterly', 'profits', 'at', 'us']
    

此处多出了一个 generate_test_batch 函数，是用来从测试数据中生成批次数据。

    
    
    # 从测试数据中生成批次数据
    test_data_index = 0
    
    def generate_test_batch(data, batch_size):
        global test_data_index
    
        batch = np.ndarray(shape=(batch_size,), dtype=np.int32)
        # 得到 index 为 0 到 span 的单词
        for bi in range(batch_size):
            batch[bi] = data[test_data_index]
            test_data_index = (test_data_index + 1) % len(data)
    
        return batch
    
    test_data_index = 0
    test_batch = generate_test_batch(test_data[list(test_data.keys())[0]], batch_size=8)
    print('\nwith window_size = %d:' % (window_size))
    print('    labels:', [reverse_dictionary[li] for li in test_batch.reshape(8)])
    

输出：

    
    
    with window_size = 2:
        labels: ['uk', 'gets', 'official', 'virus', 'alert', 'site', 'a', 'rapid']
    

#### **6\. 设置 CBOW 的超参数**

    
    
    batch_size = 128                             
    embedding_size = 128             # embedding 向量的维度
    window_size = 4                 # 中心词的左右两边各取 4 个
    
    valid_size = 16                 # 随机选择一个 validation 集来评估单词的相似性
    valid_window = 50                # 从一个大窗口随机采样 valid 数据
    
    # 选择 valid 样本时, 取一些频率比较大的单词，也选择一些适度罕见的单词
    valid_examples = np.array(random.sample(range(valid_window), valid_size))
    valid_examples = np.append(valid_examples,random.sample(range(1000, 1000+valid_window), valid_size),axis=0)
    
    num_sampled = 32                 # 负采样的样本个数
    

#### **7\. 定义输入输出**

这里多了一个 test_labels，在后续计算中会用到。

    
    
    tf.reset_default_graph()
    
    # 用于训练的上下文数据，有 2*window_size 列
    train_dataset = tf.placeholder(tf.int32, shape=[batch_size,2*window_size])            
    
    # 用于训练的中心词
    train_labels = tf.placeholder(tf.int32, shape=[batch_size, 1])    
    
    # Validation 不需要用 placeholder，因为前面已经定义了 valid_examples
    valid_dataset = tf.constant(valid_examples, dtype=tf.int32)    
    
    # 测试数据，通过对一个给定文档中的词向量做平均来计算文档的嵌入
    test_labels = tf.placeholder(tf.int32, shape=[batch_size],name='test_dataset')
    

#### **8\. 定义模型的参数**

定义一个 embedding 层和神经网络的参数。

    
    
    embeddings = tf.Variable(tf.random_uniform([vocabulary_size, embedding_size], -1.0, 1.0,dtype=tf.float32))
    # Softmax 的权重和 Biases
    softmax_weights = tf.Variable(tf.truncated_normal([vocabulary_size, embedding_size],
                         stddev=1.0 / math.sqrt(embedding_size),dtype=tf.float32))
    softmax_biases = tf.Variable(tf.zeros([vocabulary_size],dtype=tf.float32))
    

#### **9\. 定义模型**

这里多了 mean_batch_embedding，通过对测试文档中所有的词向量做平均来计算出一个文档的词嵌入表示。

    
    
    # 对测试文档中所有的词向量做平均来计算文档的嵌入
    mean_batch_embedding = tf.reduce_mean(tf.nn.embedding_lookup(embeddings,test_labels),axis=0)
    
    # 为 input 的每一列做 embedding lookups
    # 然后求平均，得到大小为 embedding_size 的词向量
    stacked_embedings = None
    print('Defining %d embedding lookups representing each word in the context'%(2*window_size))
    for i in range(2*window_size):
        embedding_i = tf.nn.embedding_lookup(embeddings, train_dataset[:,i])        
        x_size,y_size = embedding_i.get_shape().as_list()
        if stacked_embedings is None:
            stacked_embedings = tf.reshape(embedding_i,[x_size,y_size,1])
        else:
            stacked_embedings = tf.concat(axis=2,values=[stacked_embedings,tf.reshape(embedding_i,[x_size,y_size,1])])
    
    # 确保 taked embeddings 有 2*window_size 列
    assert stacked_embedings.get_shape().as_list()[2]==2*window_size
    print("Stacked embedding size: %s"%stacked_embedings.get_shape().as_list())
    
    # 计算 stacked_embedings 的平均嵌入
    mean_embeddings =  tf.reduce_mean(stacked_embedings,2,keepdims=False)
    print("Reduced mean embedding size: %s"%mean_embeddings.get_shape().as_list())
    

输出：

    
    
    Defining 8 embedding lookups representing each word in the context
    Stacked embedding size: [128, 128, 8]
    Reduced mean embedding size: [128, 128]
    

#### **10\. 定义损失和优化算法**

    
    
    # 每次用一些负采样样本计算 softmax loss, 
    # 输入是训练数据的 embeddings
    # 用这个 loss 来优化 weights, biases, embeddings
    loss = tf.reduce_mean(
        tf.nn.sampled_softmax_loss(weights=softmax_weights, biases=softmax_biases, inputs=mean_embeddings,
                               labels=train_labels, num_sampled=num_sampled, num_classes=vocabulary_size))
    
    optimizer = tf.train.AdagradOptimizer(1.0).minimize(loss)                        
    

#### **11\. 定义计算单词相似性的函数**

用余弦距离来计算样本的相似性。

    
    
    norm = tf.sqrt(tf.reduce_sum(tf.square(embeddings), 1, keepdims=True))
    normalized_embeddings = embeddings / norm
    valid_embeddings = tf.nn.embedding_lookup(normalized_embeddings, valid_dataset)
    similarity = tf.matmul(valid_embeddings, tf.transpose(normalized_embeddings))
    

#### **12\. 在文档数据上运行 CBOW 算法**

这段代码比 CBOW 的训练步骤多出来一部分，是要从每个文档中取出 batch_size*num_test_steps
个单词的词向量取平均，来计算文档的嵌入向量。

    
    
    num_steps = 100001
    cbow_loss = []
    
    config=tf.ConfigProto(allow_soft_placement=True)
    config.gpu_options.allow_growth = True
    
    with tf.Session(config=config) as session:
    
        tf.global_variables_initializer().run()
        print('Initialized')
    
        average_loss = 0
    
        # 训练 num_step 次
        for step in range(num_steps):
    
            # 生成一批数据
            batch_data, batch_labels = generate_batch(data, batch_size, window_size)
    
            # 运行优化算法，计算损失
            feed_dict = {train_dataset : batch_data, train_labels : batch_labels}
            _, l = session.run([optimizer, loss], feed_dict=feed_dict)
    
            # 更新平均损失变量
            average_loss += l
    
            if (step+1) % 2000 == 0:
                if step > 0:
                    average_loss = average_loss / 2000
                    # 计算一下平均损失
                print('Average loss at step %d: %f' % (step+1, average_loss))
                cbow_loss.append(average_loss)
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
    
        # 从每个文档中取出 batch_size*num_test_steps 个单词的词向量取平均，来计算文档的嵌入向量
        num_test_steps = 100
    
        # document_embeddings 用来存储文档的嵌入，格式为 {document_id: embedding}
        document_embeddings = {}
        print('Testing Phase (Compute document embeddings)')
    
        # 对每个测试文档，计算文档的嵌入向量
        for k,v in test_data.items():
            print('\tCalculating mean embedding for document ',k,' with ', num_test_steps, ' steps.')
            test_data_index = 0
            topic_mean_batch_embeddings = np.empty((num_test_steps,embedding_size),dtype=np.float32)
    
            for test_step in range(num_test_steps):
                test_batch_labels = generate_test_batch(test_data[k],batch_size)
                batch_mean = session.run(mean_batch_embedding,feed_dict={test_labels:test_batch_labels})
                topic_mean_batch_embeddings[test_step,:] = batch_mean
    
            document_embeddings[k] = np.mean(topic_mean_batch_embeddings,axis=0)
    

输出：

    
    
    Average loss at step 92000: 1.480679
    Average loss at step 94000: 1.445840
    Average loss at step 96000: 1.462062
    Average loss at step 98000: 1.415321
    Average loss at step 100000: 1.422292
    Nearest to for: 81, high-powered, kipchoge, births, unwind, physicians, net-using, maintains,
    Nearest to was: am, is, capitalisations, were, bmo, flash, became, dial,
    Nearest to they: you, renegade, commentating, we, mobiles, councils, divulge, 'cantona,
    Nearest to he: she, surprisngly, collins, underscores, consistency, tribunals, 75-year-old, logistical,
    ...
    Calculating mean embedding for document  tech-34  with  100  steps.
    Calculating mean embedding for document  sport-166  with  100  steps.
    Calculating mean embedding for document  sport-87  with  100  steps.
    ...
    

#### **13\. 用 t-SNE 将文档可视化**

这里我们要定义一个 t-SNE 将学习出的长度为 128 的文档嵌入向量映射到二维平面上。

    
    
    # 要可视化的数据个数
    num_points = 1000 
    
    # 建立一个 t-SNE
    tsne = TSNE(perplexity=30, n_components=2, init='pca', n_iter=5000)
    
    print('Fitting embeddings to T-SNE')
    doc_ids, doc_embeddings = zip(*document_embeddings.items())
    two_d_embeddings = tsne.fit_transform(doc_embeddings)
    print('\tDone')
    

其中 two_d_embeddings 为测试集中 50 个文档降维后的二维向量表示：

    
    
    array([[ -11.36216  ,  131.78575  ],
           [ -23.075674 ,   88.1492   ],
           [   4.4817004,   39.693035 ],
           [  46.797825 ,   88.773224 ],
           [ -36.411736 ,   65.47163  ],
           [ -73.96673  ,   63.287937 ],
           [  11.076577 ,   86.579575 ],
           ...)
    

画出 t-SNE 的结果。

    
    
    def plot(embeddings, labels):
    
      n_clusters = 5         # clusters 的数量
    
      # 为每个 cluster 自动分配颜色和标记
      label_colors = [pylab.cm.spectral(float(i) /n_clusters) for i in range(n_clusters)]
      label_markers = ['o','^','d','s','x']
    
      # 确保文档嵌入的数量和标签的数目一样
      assert embeddings.shape[0] >= len(labels), 'More labels than embeddings'
    
      pylab.figure(figsize=(15,15))  # in inches
    
      # 给每个类别设置一个 cluster_id
      def get_label_id_from_key(key):
        if 'business' in key:
            return 0
        elif 'entertainment' in key:
            return 1
        elif 'politics' in key:
            return 2
        elif 'sport' in key:
            return 3
        elif 'tech' in key:
            return 4
    
      # 画出所有文档的嵌入
      for i, label in enumerate(labels):
        x, y = embeddings[i,:]
        pylab.scatter(x, y, c=label_colors[get_label_id_from_key(label)],s=50,
                      marker=label_markers[get_label_id_from_key(label)])    
    
        # 在散点图上标记出每个点
        pylab.annotate(label, xy=(x, y), xytext=(5, 2), textcoords='offset points',
                       ha='right', va='bottom',fontsize=16)
    
      pylab.title('Document Embeddings visualized with t-SNE',fontsize=24)
    
      pylab.savefig('document_embeddings.png')
      pylab.show()
    
    plot(two_d_embeddings, doc_ids)
    

![](https://images.gitbook.cn/8242b200-5efc-11ea-a388-4dcdef07331c)

上图就是用 t-SNE 算法将 CBOW
学习出的测试集文档的嵌入表示可视化出来，用不同符号表示五种类别。可以看出同一类别中的大多数文档都聚得比较近，当然也有一些异常点，例如 sport-176
离运动类别的其他文档就比较远，离娱乐和科技比较近。针对这些异常值，可以打开文档查看一下里面的内容，会发现虽然分别标签是
sport，但是里面的内容和词语更多的是关于娱乐方面的，也正是算法学习出来的结果。

#### **14\. 对文档进行聚类**

上面是通过可视化的方法大概看一下每个类别的聚拢情况，这里我们用 K-means 对文档的嵌入向量进行聚类，并列出每个类中都有哪些文档，下面的结果中 0
表示娱乐类，1 表示体育类，2 表示商业类，3 表示科技类，4
表示政治类，课件政治类中的异常值多一些，商业、体育、娱乐都包含了两篇文档。当遇到新文档时，将其加入到测试集中，重新进行
K-means，这个新文档所在类别里面找到最多的那个种类就是它的预测标签。

    
    
    # 训练 K-means
    kmeans = KMeans(n_clusters=5, random_state=43643, max_iter=10000, n_init=100, algorithm='elkan')
    kmeans.fit(np.array(list(document_embeddings.values())))
    
    # 计算每个簇中的文档
    document_classes = {}
    
    for inp, lbl in zip(list(document_embeddings.keys()), kmeans.labels_):
        if lbl not in document_classes:
            document_classes[lbl] = [inp]
        else:
            document_classes[lbl].append(inp)
    
    for k,v in document_classes.items():    
        print('\nDocuments in Cluster ',k)
        print('\t',v)
    

输出：

    
    
    Documents in Cluster  4
         ['business-87', 'business-81', 'business-15', 'entertainment-137', 'entertainment-221', 'politics-82', 'politics-97', 'politics-167', 'politics-148', 'politics-207', 'politics-238', 'politics-78', 'politics-171', 'politics-9', 'politics-168', 'sport-203', 'sport-81']
    
    Documents in Cluster  2
         ['business-228', 'business-152', 'business-208', 'business-71', 'business-85', 'business-190', 'business-79']
    
    Documents in Cluster  0
         ['entertainment-119', 'entertainment-99', 'entertainment-157', 'entertainment-183', 'entertainment-218', 'entertainment-230']
    
    Documents in Cluster  1
         ['entertainment-180', 'sport-75', 'sport-76', 'sport-176', 'sport-236', 'sport-169', 'sport-44', 'sport-123', 'sport-100']
    
    Documents in Cluster  3
         ['entertainment-202', 'tech-22', 'tech-152', 'tech-200', 'tech-157', 'tech-14', 'tech-177', 'tech-44', 'tech-87', 'tech-134', 'tech-66']
    

好了，词嵌入这一章到这里就先告一段落，接下来我们进入新的章节——LSTM 的扩展模型，poophole、beam search 等概念，还会实现 POS。

**面试题：**

  1. 如何进行文本分类，文本聚类？

* * *

参考文献：

  * D. Greene and P. Cunningham. "Practical Solutions to the Problem of Diagonal Dominance in Kernel Document Clustering", Proc. ICML 2006. 
  * Thushan Ganegedara, _Natural Language Processing with TensorFlow_
  * Quoc Le, Tomas Mikolov, [_Distributed Representations of Sentences and Documents_](https://arxiv.org/pdf/1405.4053.pdf)

