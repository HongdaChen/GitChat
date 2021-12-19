近几年在自然语言处理领域中，HMM（隐马尔可夫模型）和
CRF（条件随机场）算法常常被用于分词、句法分析、命名实体识别、词性标注等。由于两者之间有很大的共同点，所以在很多应用上往往是重叠的，但在命名实体、句法分析等领域
CRF 似乎更胜一筹。通常来说如果做自然语言处理，这两个模型应该都要了解，下面我们来看看本文的内容。

### 从贝叶斯定义理解生成式模型和判别式模型

理解 HMM（隐马尔可夫模型）和 CRF（条件随机场）模型之前，我们先来看两个概念：生成式模型和判别式模型。

在机器学习中，生成式模型和判别式模型都用于有监督学习，有监督学习的任务就是从数据中学习一个模型（也叫分类器），应用这一模型，对给定的输入 X 预测相应的输出
Y。这个模型的一般形式为：决策函数 Y=f(X) 或者条件概率分布 P(Y|X)。

首先，简单从贝叶斯定理说起，若记 P(A)、P(B) 分别表示事件 A 和事件 B 发生的概率，则 P(A|B) 表示事件 B 发生的情况下事件 A
发生的概率；P(AB)表示事件 A 和事件 B 同时发生的概率。

根据贝叶斯公式可以得出：

![enter image description
here](http://images.gitbook.cn/ba258630-7e8d-11e8-abd3-eb6d72babbec)

**生成式模型：** 估计的是联合概率分布，P(Y, X)=P(Y|X)*P(X)，由联合概率密度分布 P(X,Y)，然后求出条件概率分布 P(Y|X)
作为预测的模型，即生成模型公式为：P(Y|X)= P(X,Y)/ P(X)。基本思想是首先建立样本的联合概率密度模型 P(X,Y)，然后再得到后验概率
P(Y|X)，再利用它进行分类，其主要关心的是给定输入 X 产生输出 Y 的生成关系。

**判别式模型：** 估计的是条件概率分布， P(Y|X)，是给定观测变量 X 和目标变量 Y 的条件模型。由数据直接学习决策函数 Y=f(X)
或者条件概率分布 P(Y|X) 作为预测的模型，其主要关心的是对于给定的输入 X，应该预测什么样的输出 Y。

所以，HMM 使用隐含变量生成可观测状态，其生成概率有标注集统计得到，是一个生成模型。其他常见的生成式模型有：Gaussian、 Naive
Bayes、Mixtures of multinomials 等。

而 CRF 就像一个反向的隐马尔可夫模型（HMM），通过可观测状态判别隐含变量，其概率亦通过标注集统计得来，是一个判别模型。其他常见的判别式模型有：K
近邻法、感知机、决策树、逻辑斯谛回归模型、最大熵模型、支持向量机、提升方法等。

HMM（隐马尔可夫模型）和 CRF（条件随机场）的理论部分，推荐看周志华老师的西瓜书《机器学习》。

### 动手实战：基于 HMM 训练自己的 Python 中文分词器

#### 模型介绍

HMM 模型是由一个“五元组”组成的集合：

  * StatusSet：状态值集合，状态值集合为 (B, M, E, S)，其中 B 为词的首个字，M 为词中间的字，E 为词语中最后一个字，S 为单个字，B、M、E、S 每个状态代表的是该字在词语中的位置。

举个例子，对“中国的人工智能发展进入高潮阶段”，分词可以标注为：“中B国E的S人B工E智B能E发B展E进B入E高B潮E阶B段E”，最后的分词结果为：['中国',
'的', '人工', '智能', '发展', '进入', '高潮', '阶段']。

  * ObservedSet：观察值集合，观察值集合就是所有语料的汉字，甚至包括标点符号所组成的集合。

  * TransProbMatrix：转移概率矩阵，状态转移概率矩阵的含义就是从状态 X 转移到状态 Y 的概率，是一个4×4的矩阵，即 {B,E,M,S}×{B,E,M,S}。

  * EmitProbMatrix：发射概率矩阵，发射概率矩阵的每个元素都是一个条件概率，代表 P(Observed[i]|Status[j]) 概率。

  * InitStatus：初始状态分布，初始状态概率分布表示句子的第一个字属于 {B,E,M,S} 这四种状态的概率。

将 HMM
应用在分词上，要解决的问题是：参数（ObservedSet、TransProbMatrix、EmitRobMatrix、InitStatus）已知的情况下，求解状态值序列。

解决这个问题的最有名的方法是 Viterbi 算法。

#### 语料准备

本次训练使用的预料 `syj_trainCorpus_utf8.txt` 是我爬取的短文本处理生成的。整个语料大小
264M，包含1116903条数据，UTF-8 编码，词与词之间用空格隔开，用来训练分词模型。

语料已上传到 CSDN
资源上，下载地址请点击：[中文自然语言处理中文分词训练语料](https://download.csdn.net/download/qq_36330643/10514771)
。

语料格式，用空格隔开的：

> 如果 继续 听任 资产阶级 自由化 的 思潮 泛滥 ，
>
> 党 就 失去 了 凝聚力 和 战斗力 ，
>
> 怎么 能 成为 全国 人民 的 领导 核心 ？
>
> 中国 又 会 成为 一盘散沙 ，
>
> 那 还有 什么 希望 ？

#### 编码实现

（1）预定义

首先引出库，这两个库的作用是用来模型保存的：

    
    
        import pickle
        import json
    

接下来定义 HMM 中的状态，初始化概率，以及中文停顿词：

    
    
        STATES = {'B', 'M', 'E', 'S'}
        EPS = 0.0001
        #定义停顿标点
        seg_stop_words = {" ","，","。","“","”",'“', "？", "！", "：", "《", "》", "、", "；", "·", "‘ ", "’", "──", ",", ".", "?", "!", "`", "~", "@", "#", "$", "%", "^", "&", "*", "(", ")", "-", "_", "+", "=", "[", "]", "{", "}", '"', "'", "<", ">", "\\", "|" "\r", "\n","\t"}
    

（2）面向对象封装成类

首先，将 HMM 模型封装为独立的类 `HMM_Model`，下面先给出类的结构定义：

    
    
        class HMM_Model:
            def __init__(self):
                pass
            #初始化    
            def setup(self):
                pass
             #模型保存   
            def save(self, filename, code):
                pass
            #模型加载
            def load(self, filename, code):
                pass
            #模型训练
            def do_train(self, observes, states):
                pass
            #HMM计算
            def get_prob(self):
                pass
            #模型预测
            def do_predict(self, sequence):
                pass
    

第一个方法 `__init__()`
是一种特殊的方法，被称为类的构造函数或初始化方法，当创建了这个类的实例时就会调用该方法，其中定义了数据结构和初始变量，实现如下：

    
    
        def __init__(self):
                self.trans_mat = {}  
                self.emit_mat = {} 
                self.init_vec = {}  
                self.state_count = {} 
                self.states = {}
                self.inited = False
    

其中的数据结构定义：

  * `trans_mat`：状态转移矩阵，`trans_mat[state1][state2]` 表示训练集中由 state1 转移到 state2 的次数。 

  * `emit_mat`：观测矩阵，`emit_mat[state][char]` 表示训练集中单字 char 被标注为 state 的次数。

  * `init_vec`：初始状态分布向量，`init_vec[state]` 表示状态 state 在训练集中出现的次数。

  * `state_count`：状态统计向量，`state_count[state]`表示状态 state 出现的次数。

  * `word_set`：词集合，包含所有单词。

第二个方法 setup()，初始化第一个方法中的数据结构，具体实现如下：

    
    
            #初始化数据结构    
            def setup(self):
                for state in self.states:
                    # build trans_mat
                    self.trans_mat[state] = {}
                    for target in self.states:
                        self.trans_mat[state][target] = 0.0
                    self.emit_mat[state] = {}
                    self.init_vec[state] = 0
                    self.state_count[state] = 0
                self.inited = True
    

第三个方法 save()，用来保存训练好的模型，filename 指定模型名称，默认模型名称为 hmm.json，这里提供两种格式的保存类型，JSON 或者
pickle 格式，通过参数 code 来决定，code 的值为 `code='json'` 或者 `code = 'pickle'`，默认为
`code='json'`，具体实现如下：

    
    
        #模型保存   
        def save(self, filename="hmm.json", code='json'):
            fw = open(filename, 'w', encoding='utf-8')
            data = {
                "trans_mat": self.trans_mat,
                "emit_mat": self.emit_mat,
                "init_vec": self.init_vec,
                "state_count": self.state_count
            }
            if code == "json":
                txt = json.dumps(data)
                txt = txt.encode('utf-8').decode('unicode-escape')
                fw.write(txt)
            elif code == "pickle":
                pickle.dump(data, fw)
            fw.close()
    

第四个方法 load()，与第三个 save() 方法对应，用来加载模型，filename 指定模型名称，默认模型名称为
hmm.json，这里提供两种格式的保存类型，JSON 或者 pickle 格式，通过参数 code 来决定，code 的值为 `code='json'`
或者 `code = 'pickle'`，默认为 `code='json'`，具体实现如下：

    
    
        #模型加载
        def load(self, filename="hmm.json", code="json"):
            fr = open(filename, 'r', encoding='utf-8')
            if code == "json":
                txt = fr.read()
                model = json.loads(txt)
            elif code == "pickle":
                model = pickle.load(fr)
            self.trans_mat = model["trans_mat"]
            self.emit_mat = model["emit_mat"]
            self.init_vec = model["init_vec"]
            self.state_count = model["state_count"]
            self.inited = True
            fr.close()
    

第五个方法 `do_train()`，用来训练模型，因为使用的标注数据集， 因此可以使用更简单的监督学习算法，训练函数输入观测序列和状态序列进行训练，
依次更新各矩阵数据。类中维护的模型参数均为频数而非频率，
这样的设计使得模型可以进行在线训练，使得模型随时都可以接受新的训练数据继续训练，不会丢失前次训练的结果。具体实现如下：

    
    
        #模型训练
        def do_train(self, observes, states):
            if not self.inited:
                self.setup()
    
            for i in range(len(states)):
                if i == 0:
                    self.init_vec[states[0]] += 1
                    self.state_count[states[0]] += 1
                else:
                    self.trans_mat[states[i - 1]][states[i]] += 1
                    self.state_count[states[i]] += 1
                    if observes[i] not in self.emit_mat[states[i]]:
                        self.emit_mat[states[i]][observes[i]] = 1
                    else:
                        self.emit_mat[states[i]][observes[i]] += 1
    

第六个方法 `get_prob()`，在进行预测前，需将数据结构的频数转换为频率，具体实现如下：

    
    
        #频数转频率
        def get_prob(self):
            init_vec = {}
            trans_mat = {}
            emit_mat = {}
            default = max(self.state_count.values())  
    
            for key in self.init_vec:
                if self.state_count[key] != 0:
                    init_vec[key] = float(self.init_vec[key]) / self.state_count[key]
                else:
                    init_vec[key] = float(self.init_vec[key]) / default
    
            for key1 in self.trans_mat:
                trans_mat[key1] = {}
                for key2 in self.trans_mat[key1]:
                    if self.state_count[key1] != 0:
                        trans_mat[key1][key2] = float(self.trans_mat[key1][key2]) / self.state_count[key1]
                    else:
                        trans_mat[key1][key2] = float(self.trans_mat[key1][key2]) / default
    
            for key1 in self.emit_mat:
                emit_mat[key1] = {}
                for key2 in self.emit_mat[key1]:
                    if self.state_count[key1] != 0:
                        emit_mat[key1][key2] = float(self.emit_mat[key1][key2]) / self.state_count[key1]
                    else:
                        emit_mat[key1][key2] = float(self.emit_mat[key1][key2]) / default
            return init_vec, trans_mat, emit_mat
    

第七个方法 `do_predict()`，预测采用 Viterbi 算法求得最优路径， 具体实现如下：

    
    
        #模型预测
        def do_predict(self, sequence):
            tab = [{}]
            path = {}
            init_vec, trans_mat, emit_mat = self.get_prob()
    
            # 初始化
            for state in self.states:
                tab[0][state] = init_vec[state] * emit_mat[state].get(sequence[0], EPS)
                path[state] = [state]
    
            # 创建动态搜索表
            for t in range(1, len(sequence)):
                tab.append({})
                new_path = {}
                for state1 in self.states:
                    items = []
                    for state2 in self.states:
                        if tab[t - 1][state2] == 0:
                            continue
                        prob = tab[t - 1][state2] * trans_mat[state2].get(state1, EPS) * emit_mat[state1].get(sequence[t], EPS)
                        items.append((prob, state2))
                    best = max(items)  
                    tab[t][state1] = best[0]
                    new_path[state1] = path[best[1]] + [state1]
                path = new_path
    
            # 搜索最有路径
            prob, state = max([(tab[len(sequence) - 1][state], state) for state in self.states])
            return path[state]
    

上面实现了类 `HMM_Model` 的7个方法，接下来我们来实现分词器，这里先定义两个函数，这两个函数是独立的，不在类中。

（1）定义一个工具函数

对输入的训练语料中的每个词进行标注，因为训练数据是空格隔开的，可以进行转态标注，该方法用在训练数据的标注，具体实现如下：

    
    
        def get_tags(src):
            tags = []
            if len(src) == 1:
                tags = ['S']
            elif len(src) == 2:
                tags = ['B', 'E']
            else:
                m_num = len(src) - 2
                tags.append('B')
                tags.extend(['M'] * m_num)
                tags.append('E')
            return tags
    

（2）定义一个工具函数

根据预测得到的标注序列将输入的句子分割为词语列表，也就是预测得到的状态序列，解析成一个 list 列表进行返回，具体实现如下：

    
    
        def cut_sent(src, tags):
            word_list = []
            start = -1
            started = False
    
            if len(tags) != len(src):
                return None
    
            if tags[-1] not in {'S', 'E'}:
                if tags[-2] in {'S', 'E'}:
                    tags[-1] = 'S'  
                else:
                    tags[-1] = 'E'  
    
            for i in range(len(tags)):
                if tags[i] == 'S':
                    if started:
                        started = False
                        word_list.append(src[start:i])  
                    word_list.append(src[i])
                elif tags[i] == 'B':
                    if started:
                        word_list.append(src[start:i])  
                    start = i
                    started = True
                elif tags[i] == 'E':
                    started = False
                    word = src[start:i+1]
                    word_list.append(word)
                elif tags[i] == 'M':
                    continue
            return word_list
    

最后，我们来定义分词器类 HMMSoyoger，继承 `HMM_Model` 类并实现中文分词器训练、分词功能，先给出 HMMSoyoger 类的结构定义：

    
    
        class HMMSoyoger(HMM_Model):
            def __init__(self, *args, **kwargs):
                pass
            #加载训练数据
            def read_txt(self, filename):
                pass
            #模型训练函数
            def train(self):
                pass
            #模型分词预测
            def lcut(self, sentence):
                pass
    

第一个方法 init()，构造函数，定义了初始化变量，具体实现如下：

    
    
        def __init__(self, *args, **kwargs):
                super(HMMSoyoger, self).__init__(*args, **kwargs)
                self.states = STATES
                self.data = None
    

第二个方法 `read_txt()`，加载训练语料，读入文件为 txt，并且 UTF-8 编码，防止中文出现乱码，具体实现如下：

    
    
        #加载语料
        def read_txt(self, filename):
                self.data = open(filename, 'r', encoding="utf-8")
    

第三个方法 train()，根据单词生成观测序列和状态序列，并通过父类的 `do_train()` 方法进行训练，具体实现如下：

    
    
        def train(self):
                if not self.inited:
                    self.setup()
    
                for line in self.data:
                    line = line.strip()
                    if not line:
                        continue
    
                   #观测序列
                    observes = []
                    for i in range(len(line)):
                        if line[i] == " ":
                            continue
                        observes.append(line[i])
    
                    #状态序列
                    words = line.split(" ")  
    
                    states = []
                    for word in words:
                        if word in seg_stop_words:
                            continue
                        states.extend(get_tags(word))
                    #开始训练
                    if(len(observes) >= len(states)):
                        self.do_train(observes, states)
                    else:
                        pass
    

第四个方法 lcut()，模型训练好之后，通过该方法进行分词测试，具体实现如下：

    
    
        def lcut(self, sentence):
                try:
                    tags = self.do_predict(sentence)
                    return cut_sent(sentence, tags)
                except:
                    return sentence
    

通过上面两个类和两个方法，就完成了基于 HMM 的中文分词器编码，下面我们来进行模型训练和测试。

#### 训练模型

首先实例化 HMMSoyoger 类，然后通过 `read_txt()` 方法加载语料，再通过 train()
进行在线训练，如果训练语料比较大，可能需要等待一点时间，具体实现如下：

    
    
        soyoger = HMMSoyoger()
        soyoger.read_txt("syj_trainCorpus_utf8.txt")
        soyoger.train()
    

#### 模型测试

模型训练完成之后，我们就可以进行测试：

    
    
        soyoger.lcut("中国的人工智能发展进入高潮阶段。")
    

得到结果为：

> ['中国', '的', '人工', '智能', '发展', '进入', '高潮', '阶段', '。']
    
    
    soyoger.lcut("中文自然语言处理是人工智能技术的一个重要分支。")
    

得到结果为：

> ['中文', '自然', '语言', '处理', '是人', '工智', '能技', '术的', '一个', '重要', '分支', '。']

可见，最后的结果还是不错的，如果想取得更好的结果，可自行制备更大更丰富的训练数据集。

### 基于 CRF 的开源中文分词工具 Genius 实践

Genius 是一个基于 CRF 的开源中文分词工具，采用了 Wapiti 做训练与序列标注，支持 Python 2.x、Python 3.x。

#### 安装

（1）下载源码

在 [Github](https://github.com/duanhongyi/genius) 上下载源码地址，解压源码，然后通过 `python
setup.py install` 安装。

（2）Pypi 安装

通过执行命令：`easy_install genius` 或者 `pip install genius` 安装。

#### 分词

首先引入 Genius，然后对 text 文本进行分词。

    
    
    import genius
    text = u"""中文自然语言处理是人工智能技术的一个重要分支。"""
    seg_list = genius.seg_text(
        text,
        use_combine=True,
        use_pinyin_segment=True,
        use_tagging=True,
        use_break=True
    )
    print(' '.join([word.text for word in seg_list])
    

其中，`genius.seg_text` 函数接受5个参数，其中 text 是必填参数：

  * text 第一个参数为需要分词的字。
  * `use_break` 代表对分词结构进行打断处理，默认值 True。
  * `use_combine` 代表是否使用字典进行词合并，默认值 False。
  * `use_tagging` 代表是否进行词性标注，默认值 True。
  * `use_pinyin_segment` 代表是否对拼音进行分词处理，默认值 True。

### 总结

本文首先通过贝叶斯定理，理解了判别式模型和生成式模型的区别，接着通过动手实战——基于 HMM 训练出自己的 Python
中文分词器，并进行了模型验证，最后给出一个基于 CRF 的开源中文分词工具。

### 参考文献

  1. [Genius](https://github.com/duanhongyi/genius)
  2. 周志华《机器学习》

### [示例数据下载](https://github.com/sujeek/chinese_nlp)

