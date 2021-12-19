今天我们来看自然语言处理中一个很重要的应用领域：情感分析。

情感分析也称为意见挖掘，是要从文本中识别和提取意见，要识别出用户对一件事一个物品或一个人的看法和态度，比如一个电影的评论，一个商品的评价，一次体验的感想等等。在这类任务中要对带有情感色彩的主观性文本进行分析，识别出用户的态度，是喜欢，讨厌，还是中立。情感分析的分析粒度可以是词语、句子、段落或篇章。

在实际生活中 **情感分析有很广泛的应用** ，例如通过对 Twitter
用户的情感分析，来预测股票走势、预测电影票房、选举结果等，还可以用来了解用户对公司、产品的喜好，分析结果可以用来改善产品和服务，还可以发现竞争对手的优劣势等等。在社交媒体监控，品牌监控，客户之声，客户服务，员工分析，产品分析，市场研究与分析等问题上都可以用情感分析。

**实现情感分析的方法有很多种，可分为基于规则和自动化系统。**

**1\. 基于规则：**

指人为地制定一些用来识别态度和意见主体的规则来执行情感分析任务，需要用到标注好的情感词典。例如分析电影评论时，专门构建电影行业的情感词典，效果会比通用情感词典好很多。

一般流程为：

  1. 定义两个态度极性的词列表（例如差、最差、丑陋等负面词，好、最佳、美丽等正面词）。
  2. 给一个文本，计算文本中出现的正面词数，计算文本中出现的负面词数。
  3. 如果正面词出现的数量大于负面单词出现的数量，则返回正面情绪，反之则返回负面情绪，相等则返回中立态度。

当然这个方法非常非常简单，会存在很多问题，一点都没有考虑到这些情感单词是如何与其他词组合的。

**2\. 自动系统：**

指应用机器学习等技术从数据中学习出情感类别，这时需要大量的人工标注的语料作为训练集，然后提取文本特征，构建分类器，进行分类。可以使用 Naïve
Bayes、Logistic Regression、Support Vector Machines、Neural Networks 等很多模型。

当然还可以组成混合系统，将基于规则和自动的方法搭配使用。

在本文中我们将主要应用注意力机制来做情感分析的任务。

### 情感分析 Kaggle 实战

接下来我们就用[这个 Kaggle 数据集](https://www.kaggle.com/c/movie-review-sentiment-
analysis-kernels-only/data)来看一下如何做情感分析。这是一个电影评论的数据集，[由 Pang and L. Lee.
收集](http://www.cs.cornell.edu/home/llee/papers/pang-lee-stars.pdf)，[Socher et
al.](https://nlp.stanford.edu/~socherr/EMNLP2013_RNTN.pdf) 用斯坦福的 Mechanical
Turk
为语料库中所有已解析的短语创建了细粒度标签。斯坦福NLP还提供了一个[网页](http://nlp.stanford.edu:8080/sentiment/rntnDemo.html)，可以输入或者上传一段影评，提交后可以画出
Recursive Neural Tensor Network 的可视化结果，还可以看到每个节点的详细分类的概率。

![](https://images.gitbook.cn/923bc7d0-7ca9-11ea-9164-d34ec3ae1078)

我们的目标是， **给每个短语打上这五种标签：很差、有点差、中立、有点好、很好。**

### 宏观把握目标

我们最后需要做到的是，将评论的短语数据输入给模型，可以返回这个短语的情感类别，评估的标准就是分类结果有多准确：

    
    
    # 类别代号：
    0 - negative
    1 - somewhat negative
    2 - neutral
    3 - somewhat positive
    4 - positive
    
    # 提交结果形式：
    PhraseId,Sentiment
    156061,2
    156062,2
    156063,2
    ...
    

### 了解数据

我们在[比赛的 Data](https://www.kaggle.com/c/movie-review-sentiment-analysis-
kernels-only/data) 这里先大体查看一下数据：

**训练集的数据中，一共有四列：**

  * 第二列 SentenceId 是每个完整评论的编号
  * 第一列 PhraseId 是每个完整句子被解析成若干个短语后，所有短语的编号
  * 第三列 Phrase 就是解析后的短语，当然每个 SentenceId 对应的第一行 Phrase 是相应完整的评论
  * 第四列 Sentiment 是每个短语的情绪分类标签

![](https://images.gitbook.cn/b7ee81c0-7ca9-11ea-9792-81939fbf7f0c)

**测试集的数据有三列，** 除了情感标签，剩下三列和训练集是一样的：

![](https://images.gitbook.cn/c86faa60-7ca9-11ea-a711-0f902cb8434c)

这个情感分析的比赛和常见的情感分析任务有些不一样，一般的是预测完整评论的分类就可以了，这个将每句话解析成若干个短语，还要预测各个短语的标签。

我们将训练集的数据下拉浏览一下，可以看到还有很多短语只有一个单词，这样的短语几乎都被判定为类别 2。也有一些短语可能只是多了一个标点符号，情感标签就从 2
到了 3。有些短语的开头是标点。还有的短语里出现了 POS 标签，甚至只有这些标签，比如 RRB-、-LRB- 等等。

已经明确了问题，那么我们可以先想一下都可以用什么模型，情感分析这个问题可以选择的模型有很多，比如前面提到的机器学习模型，除了这些，我们还可以用很多深度学习模型：

  * WordEmbeddings + dynamic word n-grams + CNN1D；

  * WordEmbeddings + LSTM/GRU；

  * 对短语用不同粒度的 embeddings 方法，如 Phrase2Vec 或 sentence2Vec，不同的词嵌入模型 FastText、Glove、Word2Vec；

  * 用预训练语言模型 ElMo。

  * 对模型进行多种组合：BiLSTM + Conv1D，Conv1D + BiLSTM，Conv1D + BiGRU，Multi channel Conv1D 或者 parallel ConV1D，Heirarchical Conv1D。

前面我们刚刚学习过注意力机制，那么这个实战中我们先应用注意力机制，大家如果感兴趣，可以多多尝试。因为我们还需要用到[这个
Fasttext](https://www.kaggle.com/yekenot/fasttext-crawl-300d-2m) 的 300d
的词向量（4.21G），如果觉得下载到本地的数据量大，可以直接在 Kaggle 上运行，步骤很简单：

  1. 进入这个[比赛页面](https://www.kaggle.com/c/movie-review-sentiment-analysis-kernels-only)
  2. 点击 Late Submission
  3. 点击 New Notebook，选择 Python，Kernel 即可建立一个新的 Notebook

![](https://images.gitbook.cn/ddfe4fd0-7ca9-11ea-b2e6-fdc2f968c34f)

#### **导入所需的库**

新建一个 Notebook 后默认会打印出当前文件夹里的数据：

    
    
    # This Python 3 environment comes with many helpful analytics libraries installed
    # It is defined by the kaggle/python docker image: https://github.com/kaggle/docker-python
    # For example, here's several helpful packages to load in 
    
    import numpy as np # linear algebra
    import pandas as pd # data processing, CSV file I/O (e.g. pd.read_csv)
    
    # Input data files are available in the "../input/" directory.
    # For example, running this (by clicking run or pressing Shift+Enter) will list all files under the input directory
    
    import os
    for dirname, _, filenames in os.walk('/kaggle/input'):
        for filename in filenames:
            print(os.path.join(dirname, filename))
    
    # Any results you write to the current directory are saved as output.
    
    
    
    /kaggle/input/movie-review-sentiment-analysis-kernels-only/train.tsv.zip
    /kaggle/input/movie-review-sentiment-analysis-kernels-only/sampleSubmission.csv
    /kaggle/input/movie-review-sentiment-analysis-kernels-only/test.tsv.zip
    
    
    
    import re
    import os
    import gc
    import logging
    import warnings
    warnings.filterwarnings("ignore")
    
    import numpy as np 
    import pandas as pd 
    import tensorflow as tf
    import keras.backend as K
    
    from keras.preprocessing.sequence import pad_sequences
    from keras.preprocessing import sequence,text
    from keras.preprocessing.text import Tokenizer
    from keras.models import Sequential
    from keras.layers import *
    from keras.callbacks import EarlyStopping
    from keras.utils import to_categorical
    from keras.losses import categorical_crossentropy
    from keras.optimizers import Adam
    from sklearn.model_selection import train_test_split
    from sklearn.metrics import accuracy_score,confusion_matrix,classification_report,f1_score
    from keras.engine.topology import Layer
    from keras import initializers, regularizers, constraints
    from keras.models import Model
    
    import string
    from textblob import TextBlob
    import nltk
    from nltk.tokenize import sent_tokenize, word_tokenize
    from nltk import FreqDist
    from nltk.stem import SnowballStemmer,WordNetLemmatizer
    stemmer=SnowballStemmer('english')
    lemma=WordNetLemmatizer()
    from string import punctuation
    

#### **读取数据**

其中各列的含义我们在前面已经介绍过了，注意这里的数据现在是压缩文件。

    
    
    test = pd.read_csv(r"/kaggle/input/movie-review-sentiment-analysis-kernels-only/test.tsv.zip",sep="\t")
    train = pd.read_csv(r"/kaggle/input/movie-review-sentiment-analysis-kernels-only/train.tsv.zip",sep="\t")
    
    train.head(10)
    

![](https://images.gitbook.cn/f0300ea0-7ca9-11ea-b2e6-fdc2f968c34f)

### 数据可视化分析

这一步将进行进一步的可视化分析：

  1. 数据清洗
  2. 字数统计
  3. 情感统计图
  4. 每个情绪的句子长度的最小值，平均值，最大值
  5. 统计训练集和测试集中标点的个数
  6. 用不同的图表样式展示每个情绪类别的文字云

#### **1\. 数据清洗**

首先稍作一些数据清洗，将训练集和测试集的数据中全部转化成小写，并做词形还原。

    
    
    def clean_review(review_col):
        review_corpus=[] 
        for i in range(0,len(review_col)):
            review=str(review_col[i])
            review=re.sub('[^a-zA-Z]',' ',review)
            #review=[stemmer.stem(w) for w in word_tokenize(str(review).lower())]
            review=[lemma.lemmatize(w) for w in word_tokenize(str(review).lower())]
            review=' '.join(review)
            review_corpus.append(review)
        return review_corpus
    
    train['csen']=clean_review(train.Phrase.values)
    train.head()
    

![](https://images.gitbook.cn/41b46f50-7caa-11ea-9687-397b36d0cdab)

    
    
    test['csen']=clean_review(test.Phrase.values)
    test.head()
    

![](https://images.gitbook.cn/89449840-7caa-11ea-b348-9df247d9e896)

#### **2\. 字数统计**

然后统计一下每个短语包含的单词个数：

    
    
    train['word'] = train['csen'].apply(lambda x: len(TextBlob(x).words))
    test['word'] = test['csen'].apply(lambda x: len(TextBlob(x).words))
    
    train = train[train['word'] >= 1]
    train = train.reset_index(drop=True)
    
    train.head()
    

![](https://images.gitbook.cn/a4a47fb0-7caa-11ea-9687-397b36d0cdab)

    
    
    test.head()
    

![](https://images.gitbook.cn/b77e6560-7caa-11ea-bb65-a9795596171f)

#### **3\. 情感统计图**

这部分我们会用 Plotly 来做可视化，它是一款基于 D3.js 框架的 Python
库，可以做出多种酷炫的数据图表，能够显示出数据细节，还可以进行缩放查看细节。

    
    
    import plotly.offline as py
    py.init_notebook_mode(connected=True)
    from plotly.offline import init_notebook_mode, iplot
    import plotly.figure_factory as ff
    import matplotlib as plt
    import plotly.graph_objs as go
    import plotly.tools as tls
    %matplotlib inline
    import cufflinks as cf
    cf.go_offline()
    

先统计出每个情感类别的个数：

    
    
    s_a = train.groupby('Sentiment')['word'].describe().reset_index()
    s_a
    

![](https://images.gitbook.cn/d35f63b0-7caa-11ea-bb65-a9795596171f)

下图可以看出：类别 2 出现次数是最多的，类别 1 和 3 次之，最少的是类别 0 和 4。

    
    
    trace1 = go.Bar(
        x=s_a['Sentiment'],
        y=s_a['count'] 
    )
    
    data = [trace1]
    layout = go.Layout(
        title = 'Sentiment_Count'
    )
    
    fig = go.Figure(data=data, layout=layout)
    py.iplot(fig, filename='grouped-bar')
    

![](https://images.gitbook.cn/e8e8b1f0-7caa-11ea-b2e6-fdc2f968c34f)

#### **4\. 每个情绪的句子长度的最小值，平均值，最大值**

在这五个类别中，短语的最小长度都是 1，就是被拆成了一个词；最大长度差不多；平均长度的对比状态和上一个图中类别总数的对比状态正好相反。

    
    
    trace1 = go.Bar(
        x=s_a['Sentiment'],
        y=s_a['min'],
        name='Min Sentence length'
    )
    
    trace2 = go.Bar(
        x=s_a['Sentiment'],
        y=s_a['mean'],
        name='Average Sentence length'
    )
    
    trace3 = go.Bar(
        x=s_a['Sentiment'],
        y=s_a['max'],
        name='Max Sentence length'
    )
    
    data = [trace1, trace2,trace3]
    layout = go.Layout(
        barmode='group',
        title ='Sentence length anlysis sentiment wise'
    )
    
    fig = go.Figure(data=data, layout=layout)
    py.iplot(fig, filename='grouped-bar')
    

![](https://images.gitbook.cn/028f77b0-7cab-11ea-b2e6-fdc2f968c34f)

#### **5\. 统计训练集和测试集中标点的个数**

从图中可以看出，情感类别 2 中的标点要比类别 1、3 多，类别 0 和 4 是最少的。

    
    
    count = lambda l1, l2: len(list(filter(lambda c: c in l2, l1)))
    train['pun_count'] = train['Phrase'].apply(lambda x: count(x, string.punctuation)) 
    test['pun_count'] = test['Phrase'].apply(lambda x: count(x, string.punctuation))
    
    pun_count = train.groupby('Sentiment')['pun_count'].sum().reset_index()
    trace1 = go.Bar(
        x=pun_count['Sentiment'],
        y=pun_count['pun_count'],
        marker=dict(
            color='rgba(222,45,38,0.8)',
        )
    )
    
    data = [trace1]
    layout = go.Layout(
        title = 'Sentiment wise Punctuation Count'
    )
    
    fig = go.Figure(data=data, layout=layout)
    py.iplot(fig, filename='grouped-bar')
    

![](https://images.gitbook.cn/1e29f720-7cab-11ea-9687-397b36d0cdab)

#### **6\. 每个情绪类别的文字云**

接下来我们用不同的图表样式展示各个类别的文字云，在观察高频词的同时，也可以看看不同的形式可以得到什么样的信息。

首先需要将文字云所需要的形状图片加载进来，图片数据[在 Kaggle
上面](https://www.kaggle.com/aashita/masks)也有，我们可以点击右侧的 Add Data，搜索 Masks for
word clouds，可以显示出数据的地址，点击 Add 即可添加到自己的 Kernel 里面：

![](https://images.gitbook.cn/3eada280-7cab-11ea-9164-d34ec3ae1078)

    
    
    from PIL import Image
    from wordcloud import WordCloud, STOPWORDS, ImageColorGenerator
    from os import path
    import warnings 
    from os import path
    from PIL import Image
    import numpy as np
    import matplotlib.pyplot as plt
    import os
    
    from wordcloud import WordCloud, STOPWORDS
    warnings.filterwarnings('ignore')
    stopwords = set(STOPWORDS)
    

**类别 0：**

类别 0 是评论最差的级别，从这个图中可以看到虽然最明显的一些词是 movie 等没有情感的名词，但是 worst、bad、meaningless
这种负面词也是清晰可见的。

    
    
    def plot_wordcloud(text, mask=None, max_words=400, max_font_size=120, figure_size=(24.0,16.0), 
                       title = None, title_size=40, image_color=False):
        wordcloud = WordCloud(background_color='white',
                        stopwords = stopwords,
                        max_words = 200,
                        max_font_size = 200, 
                        random_state = 42,
                        mask = mask,contour_width=2)
        wordcloud.generate(text)
    
        plt.figure(figsize=figure_size)
        if image_color:
            image_colors = ImageColorGenerator(mask);
            plt.imshow(wordcloud.recolor(color_func=image_colors), interpolation="bilinear");
            plt.title(title, fontdict={'size': title_size,  
                                      'verticalalignment': 'bottom'})
        else:
            plt.imshow(wordcloud);
            plt.title(title, fontdict={'size': title_size, 'color': 'green', 
                                      'verticalalignment': 'bottom'})
        plt.axis('off');
        plt.tight_layout()  
    
    comments_text = str(train[train['Sentiment'] == 0]['csen'].tolist())
    comments_mask = np.array(Image.open('../input/masks/user.png'))
    plot_wordcloud(comments_text, comments_mask, max_words=400, max_font_size=120,title = 'Word Cloud Plot for Sentiment 0', title_size=15)
    

![](https://images.gitbook.cn/2a6a3c90-7cbd-11ea-9dcd-17c924164c9c)

**类别 1：**

类别 1 是评价比较差的，可以看到文字云中显示的几乎没有出现类别 0 的那种 worst 这么强烈的负面词，而是 lack 等语气相对弱的评论。

    
    
    comments_text = str(train[train['Sentiment'] == 1]['csen'].tolist())
    comments_mask = np.array(Image.open('../input/masks/comment.png'))
    plot_wordcloud(comments_text, comments_mask, max_words=400, max_font_size=120, 
                   title = 'Word Cloud Plot for Sentiment 1', title_size=15,image_color=True)
    

![](https://images.gitbook.cn/53645950-7cbd-11ea-8f86-6d1fee05af4f)

**类别 2：**

类别 2 是中立态度，没有特别明显的正面也没有特别明显的负面，but 是相对多一些的，可见一般中立的评论里面差不多正面负面的观点都会有，更多的是
life、story 这种名词。

    
    
    text = str(train[train['Sentiment'] == 2]['csen'].tolist())
    text = text.lower()
    wordcloud = WordCloud(background_color="white", height=2700, width=3600).generate(text)
    plt.figure( figsize=(16,10) )
    plt.title('Word Cloud Plot for Sentiment 2',fontsize = 15)
    plt.imshow(wordcloud.recolor(colormap=plt.get_cmap('Set2')), interpolation='bilinear')
    plt.axis("off")
    

![](https://images.gitbook.cn/71d418d0-7cbd-11ea-b348-9df247d9e896)

**类别 3：**

类别 3 是比较积极的评论，用这种饼图可以显示出每个词的比例，这个图是交互式的，将鼠标移动到图上面，可以显示出当前词是什么，可以看到能够明显体现态度的词是
good 在第九位，funny 在第十三位。

    
    
    text = str(train[train['Sentiment'] == 3]['csen'].tolist())
    text = text.lower()
    wordcloud = WordCloud(max_words=50).generate(text)
    
    labels = list(wordcloud.words_.keys())
    values = list(wordcloud.words_.values())
    trace = go.Pie(labels=labels, values=values,textinfo='value', hoverinfo='label+value',textposition = 'inside')
    layout = go.Layout(
        title = 'Word Pie Plot for Sentiment 3'
    )
    data = [trace]
    
    fig = go.Figure(data=data, layout=layout)
    py.iplot(fig, filename='basic_pie_chart')
    

![](https://images.gitbook.cn/90e0dba0-7cbd-11ea-a711-0f902cb8434c)

**类别 4：**

类别 4 是评价最好的级别，用这种条形图可以清晰地看出每个单词的次数大小关系，这里的 good、great
是出现比较多的正面形容词，位置在第十位和第十五位，但是 performance 和 character
出现的次数比较多，可见赢得大家好评的元素主要是演技和角色这两个。

    
    
    text = str(train[train['Sentiment'] == 4]['csen'].tolist())
    text = text.lower()
    wordcloud = WordCloud(max_words=50).generate(text)
    
    labels = list(wordcloud.words_.keys())
    values = list(wordcloud.words_.values())
    trace1 = go.Bar(
        x=labels,
        y=values
    
    )
    
    layout = go.Layout(
        title = 'Word Bar Chart for Sentiment 4'
    )
    data = [trace1]
    fig = go.Figure(data=data, layout=layout)
    py.iplot(fig, filename='basic_pie_chart')
    

![](https://images.gitbook.cn/c257ab50-7cbd-11ea-bb65-a9795596171f)

再来做一些统计。

**1\. 先看有多少个句子，多少个短语：**

    
    
    print('Number of phrases in train: {}. Number of sentences in train: {}.'.format(train.shape[0], len(train.SentenceId.unique())))
    print('Number of phrases in test: {}. Number of sentences in test: {}.'.format(test.shape[0], len(test.SentenceId.unique())))
    

> Number of phrases in train: 156060. Number of sentences in train: 8529.
> Number of phrases in test: 66292. Number of sentences in test: 3310.

**2\. 统计一下训练集和测试集中每个句子平均含有多少短语：**

    
    
    print('Average count of phrases per sentence in train is {0:.0f}.'.format(train.groupby('SentenceId')['Phrase'].count().mean()))
    print('Average count of phrases per sentence in test is {0:.0f}.'.format(test.groupby('SentenceId')['Phrase'].count().mean()))
    

> Average count of phrases per sentence in train is 18. Average count of
> phrases per sentence in test is 20.

**3\. 每个短语的平均长度：**

    
    
    print('Average word length of phrases in train is {0:.0f}.'.format(np.mean(train['Phrase'].apply(lambda x: len(x.split())))))
    print('Average word length of phrases in test is {0:.0f}.'.format(np.mean(test['Phrase'].apply(lambda x: len(x.split())))))
    

> Average word length of phrases in train is 7. Average word length of phrases
> in test is 7.

经过这些初步的数据可视化和统计可以发现：

  * 由于一个句子会被切分成多个短语，所以重复的短语会比较多，因此需要保留 stopwords。
  * 有的短语可能只有一个单词，那么用字数统计，句子长度等特征应该是没什么效果的。
  * 因为在了解数据的时候，发现“有一些短语可能只是多了一个标点符号，情感标签就从 2 到了 3”，这就意味着标签符号对预测结果也是很重要的，所以在数据清洗的时候，不能一般做法那样把标点符号删掉.

![](https://upload-
images.jianshu.io/upload_images/1667471-a8498ca7df2186f7.png?imageMogr2/auto-
orient/strip%7CimageView2/2/w/1240)

  * 另外因为这些短语是随机被打乱的，同一句话中拆出来的短语标签也会不同，可以说明预测的结果和单词的顺序也是有关系的，所以不能用不体现顺序的模型算法，如 bag-of-words，TF-IDF 等。

* * *

### 4\. 数据预处理

接下来主要做下面这几步预处理：

  * Nigation handling 否定处理
  * Replacing numbers 替换数字
  * Tokenization 标记化
  * Zero padding 零填充
  * Embedding 词嵌入

**1\. Nigation handling 否定处理** ，是将否定形式转化为标准形式，比如 "aren't" 转化为 "are not"。

    
    
    train.Phrase = train.Phrase.str.replace("n't", 'not')
    test.Phrase = test.Phrase.str.replace("n't", 'not')
    

**2\. Replacing numbers 替换数字** ，是将数字转换为特定的字符，例如 “1924” 和 “123” 被转换为
“0”，有助于减少词汇量。

    
    
    train.Phrase = train.Phrase.apply(lambda x: re.sub(r'[0-9]+', '0', x))
    test.Phrase = test.Phrase.apply(lambda x: re.sub(r'[0-9]+', '0', x))
    
    x_train = train['Phrase'].values
    x_test  = test['Phrase'].values
    y_train = train['Sentiment'].values
    x = np.r_[x_train, x_test]
    

**3\. Tokenization 是将文本分解为单个单词的过程** ，一般按空格或标点符号拆分。

    
    
    tokenizer = Tokenizer(lower=True, filters='\n\t')
    tokenizer.fit_on_texts(x)
    x_train = tokenizer.texts_to_sequences(x_train)
    x_test  = tokenizer.texts_to_sequences(x_test)
    vocab_size = len(tokenizer.word_index) + 1  # +1 is for zero padding.
    print('vocabulary size: {}'.format(vocab_size))
    

**4\. Zero padding 零填充** ，是给文本填充“0”，目的是确保所有句子的长度一样。

    
    
    maxlen = len(max((s for s in np.r_[x_train, x_test]), key=len))
    x_train = sequence.pad_sequences(x_train, maxlen=maxlen, padding='post')
    x_test = sequence.pad_sequences(x_test, maxlen=maxlen, padding='post')
    print('maxlen: {}'.format(maxlen))
    print(x_train.shape)
    print(x_test.shape)
    

**5\. Embedding** ，这里我们将使用[这个
Fasttext](https://www.kaggle.com/yekenot/fasttext-crawl-300d-2m) 的 300d 词向量。

    
    
    EMBEDDING_FILE =  '../input/fasttext-crawl-300d-2m/crawl-300d-2M.vec'
    
    def load_embeddings(filename):
        embeddings = {}
        with open(filename) as f:
            for line in f:
                values = line.rstrip().split(' ')
                word = values[0]
                vector = np.asarray(values[1:], dtype='float32')
                embeddings[word] = vector
        return embeddings
    
    embeddings = load_embeddings(EMBEDDING_FILE)
    
    def filter_embeddings(embeddings, word_index, vocab_size, dim=300):
        embedding_matrix = np.zeros([vocab_size, dim])
        for word, i in word_index.items():
            if i >= vocab_size:
                continue
            vector = embeddings.get(word)
            if vector is not None:
                embedding_matrix[i] = vector
        return embedding_matrix
    
    embedding_size = 300
    embedding_matrix = filter_embeddings(embeddings, tokenizer.word_index,
                                         vocab_size, embedding_size)
    print('OOV: {}'.format(len(set(tokenizer.word_index) - set(embeddings))))
    

### 训练模型

**首先定义注意力层：** 这里用到的注意力模型来自 [Raffel et
al.](https://arxiv.org/abs/1512.08756)，这个模型在 Kaggle 中比较受欢迎。

    
    
    class Attention(Layer):
        """
        # Input shape
            3D tensor with shape: `(samples, steps, features)`.
        # Output shape
            2D tensor with shape: `(samples, features)`.
        """
    
        def __init__(self, step_dim,
                     W_regularizer=None, b_regularizer=None,
                     W_constraint=None, b_constraint=None,
                     bias=True, **kwargs):
            self.supports_masking = True
            self.init = initializers.get('glorot_uniform')
    
            self.W_regularizer = regularizers.get(W_regularizer)
            self.b_regularizer = regularizers.get(b_regularizer)
    
            self.W_constraint = constraints.get(W_constraint)
            self.b_constraint = constraints.get(b_constraint)
    
            self.bias = bias
            self.step_dim = step_dim
            self.features_dim = 0
            super(Attention, self).__init__(**kwargs)
    
        def build(self, input_shape):
            assert len(input_shape) == 3
    
            self.W = self.add_weight((input_shape[-1],),
                                     initializer=self.init,
                                     name='{}_W'.format(self.name),
                                     regularizer=self.W_regularizer,
                                     constraint=self.W_constraint)
            self.features_dim = input_shape[-1]
    
            if self.bias:
                self.b = self.add_weight((input_shape[1],),
                                         initializer='zero',
                                         name='{}_b'.format(self.name),
                                         regularizer=self.b_regularizer,
                                         constraint=self.b_constraint)
            else:
                self.b = None
            self.built = True
    
        def compute_mask(self, input, input_mask=None):
            return None
    
        def call(self, x, mask=None):
            features_dim = self.features_dim
            step_dim = self.step_dim
            eij = K.reshape(K.dot(K.reshape(x, (-1, features_dim)),
                            K.reshape(self.W, (features_dim, 1))), (-1, step_dim))
            if self.bias:
                eij += self.b
            eij = K.tanh(eij)
            a = K.exp(eij)
            if mask is not None:
                a *= K.cast(mask, K.floatx())
            a /= K.cast(K.sum(a, axis=1, keepdims=True) + K.epsilon(), K.floatx())
            a = K.expand_dims(a)
            weighted_input = x * a
            return K.sum(weighted_input, axis=1)
    
        def compute_output_shape(self, input_shape):
            return input_shape[0],  self.features_dim
    

**然后定义整个模型：**

    
    
    def build_model(maxlen, vocab_size, embedding_size, embedding_matrix):
        input_words = Input((maxlen, ))
        x_words = Embedding(vocab_size,
                            embedding_size,
                            weights=[embedding_matrix],
                            mask_zero=True,
                            trainable=False)(input_words)
        x_words = SpatialDropout1D(0.3)(x_words)
        x_words = Bidirectional(LSTM(50, return_sequences=True))(x_words)
        x = Attention(maxlen)(x_words)
        x = Dropout(0.2)(x)
        x = Dense(50, activation='relu')(x)
        x = Dropout(0.2)(x)
        pred = Dense(5, activation='softmax')(x)
    
        model = Model(inputs=input_words, outputs=pred)
        return model
    
    model = build_model(maxlen, vocab_size, embedding_size, embedding_matrix)
    model.compile(optimizer='nadam', loss='sparse_categorical_crossentropy', metrics=['accuracy'])
    model.summary()
    

![](https://images.gitbook.cn/05ecb1d0-7cbe-11ea-b2e6-fdc2f968c34f)

**训练模型：**

    
    
    save_file = 'model.h5'
    history = model.fit(x_train, y_train,
                        epochs=10, verbose=1,
                        batch_size=1024, shuffle=True)
    

**提交结果：**

    
    
    y_pred = model.predict(x_test, batch_size=1024)
    y_pred = y_pred.argmax(axis=1).astype(int)
    y_pred.shape
    
    mapping = {phrase: sentiment for _, _, phrase, sentiment, _, _, _ in train.values}
    
    # Overlapping
    for i, phrase in enumerate(test.Phrase.values):
        if phrase in mapping:
            y_pred[i] = mapping[phrase]
    
    test['Sentiment'] = y_pred
    test[['PhraseId', 'Sentiment']].to_csv('submission.csv', index=False)
    

* * *

参考文献：

  * NikitPatel，[EDA _Cleaning_ Keras=(LSTM+Clustering)](https://www.kaggle.com/nikitpatel/eda-cleaning-keras-lstm-clustering/notebook)

