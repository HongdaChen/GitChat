### 前言

从本文开始，我们就要真正进入实战部分。首先，我们按照中文自然语言处理流程的第一步获取语料，然后重点进行中文分词的学习。中文分词有很多种，常见的比如有中科院计算所
NLPIR、哈工大 LTP、清华大学 THULAC、斯坦福分词器、Hanlp 分词器、jieba 分词、IKAnalyzer 等。这里针对 jieba 和
HanLP 分别介绍不同场景下的中文分词应用。

### jieba 分词

#### jieba 安装

（1）Python 2.x 下 jieba 的三种安装方式，如下：

  * **全自动安装** ：执行命令 `easy_install jieba` 或者 `pip install jieba` / `pip3 install jieba`，可实现全自动安装。

  * **半自动安装** ：先[下载 jieba](https://pypi.python.org/pypi/jieba/)，解压后运行 `python setup.py install`。

  * **手动安装** ：将 jieba 目录放置于当前目录或者 site-packages 目录。

安装完通过 `import jieba` 验证安装成功与否。

（2）Python 3.x 下的安装方式。

Github 上 jieba 的 Python3.x 版本的路径是：https://github.com/fxsjy/jieba/tree/jieba3k。

通过 `git clone https://github.com/fxsjy/jieba.git` 命令下载到本地，然后解压，再通过命令行进入解压目录，执行
`python setup.py install` 命令，即可安装成功。

#### jieba 的分词算法

主要有以下三种：

  1. 基于统计词典，构造前缀词典，基于前缀词典对句子进行切分，得到所有切分可能，根据切分位置，构造一个有向无环图（DAG）；
  2. 基于DAG图，采用动态规划计算最大概率路径（最有可能的分词结果），根据最大概率路径分词；
  3. 对于新词(词库中没有的词），采用有汉字成词能力的 HMM 模型进行切分。

#### jieba 分词

下面我们进行 jieba 分词练习，第一步首先引入 jieba 和语料:

    
    
        import jieba
        content = "现如今，机器学习和深度学习带动人工智能飞速的发展，并在图片处理、语音识别领域取得巨大成功。"
    

（1） **精确分词**

精确分词：精确模式试图将句子最精确地切开，精确分词也是默认分词。

    
    
    segs_1 = jieba.cut(content, cut_all=False)
    print("/".join(segs_1))
    

其结果为：

> 现如今/，/机器/学习/和/深度/学习/带动/人工智能/飞速/的/发展/，/并/在/图片/处理/、/语音/识别/领域/取得/巨大成功/。

（2） **全模式**

全模式分词：把句子中所有的可能是词语的都扫描出来，速度非常快，但不能解决歧义。

    
    
        segs_3 = jieba.cut(content, cut_all=True)
        print("/".join(segs_3))
    

结果为：

>
> 现如今/如今///机器/学习/和/深度/学习/带动/动人/人工/人工智能/智能/飞速/的/发展///并/在/图片/处理///语音/识别/领域/取得/巨大/巨大成功/大成/成功//

（3） **搜索引擎模式**

搜索引擎模式：在精确模式的基础上，对长词再次切分，提高召回率，适合用于搜索引擎分词。

    
    
        segs_4 = jieba.cut_for_search(content)
        print("/".join(segs_4))
    

结果为：

>
> 如今/现如今/，/机器/学习/和/深度/学习/带动/人工/智能/人工智能/飞速/的/发展/，/并/在/图片/处理/、/语音/识别/领域/取得/巨大/大成/成功/巨大成功/。

（4） **用 lcut 生成 list**

jieba.cut 以及 `jieba.cut_for_search` 返回的结构都是一个可迭代的 Generator，可以使用 for
循环来获得分词后得到的每一个词语（Unicode）。jieba.lcut 对 cut 的结果做了封装，l 代表 list，即返回的结果是一个 list
集合。同样的，用 `jieba.lcut_for_search` 也直接返回 list 集合。

    
    
        segs_5 = jieba.lcut(content)
        print(segs_5)
    

结果为：

> ['现如今', '，', '机器', '学习', '和', '深度', '学习', '带动', '人工智能', '飞速', '的', '发展',
> '，', '并', '在', '图片', '处理', '、', '语音', '识别', '领域', '取得', '巨大成功', '。']

（5） **获取词性**

jieba 可以很方便地获取中文词性，通过 jieba.posseg 模块实现词性标注。

    
    
        import jieba.posseg as psg
        print([(x.word,x.flag) for x in psg.lcut(content)])
    

结果为：

> [('现如今', 't'), ('，', 'x'), ('机器', 'n'), ('学习', 'v'), ('和', 'c'), ('深度',
> 'ns'), ('学习', 'v'), ('带动', 'v'), ('人工智能', 'n'), ('飞速', 'n'), ('的', 'uj'),
> ('发展', 'vn'), ('，', 'x'), ('并', 'c'), ('在', 'p'), ('图片', 'n'), ('处理', 'v'),
> ('、', 'x'), ('语音', 'n'), ('识别', 'v'), ('领域', 'n'), ('取得', 'v'), ('巨大成功',
> 'nr'), ('。', 'x')]

（6） **并行分词**

并行分词原理为文本按行分隔后，分配到多个 Python 进程并行分词，最后归并结果。

用法：

    
    
    jieba.enable_parallel(4) # 开启并行分词模式，参数为并行进程数 。
    jieba.disable_parallel() # 关闭并行分词模式 。
    

注意： 并行分词仅支持默认分词器 jieba.dt 和 jieba.posseg.dt。目前暂不支持 Windows。

（7） **获取分词结果中词列表的 top n**

    
    
        from collections import Counter
        top5= Counter(segs_5).most_common(5)
        print(top5)
    

结果为：

> [('，', 2), ('学习', 2), ('现如今', 1), ('机器', 1), ('和', 1)]

（8） **自定义添加词和字典**

默认情况下，使用默认分词，是识别不出这句话中的“铁甲网”这个新词，这里使用用户字典提高分词准确性。

    
    
        txt = "铁甲网是中国最大的工程机械交易平台。"
        print(jieba.lcut(txt))
    

结果为：

> ['铁甲', '网是', '中国', '最大', '的', '工程机械', '交易平台', '。']

如果添加一个词到字典，看结果就不一样了。

    
    
        jieba.add_word("铁甲网")
        print(jieba.lcut(txt))
    

结果为：

> ['铁甲网', '是', '中国', '最大', '的', '工程机械', '交易平台', '。']

但是，如果要添加很多个词，一个个添加效率就不够高了，这时候可以定义一个文件，然后通过 `load_userdict()`函数，加载自定义词典，如下：

    
    
        jieba.load_userdict('user_dict.txt')
        print(jieba.lcut(txt))
    

结果为：

> ['铁甲网', '是', '中国', '最大', '的', '工程机械', '交易平台', '。']

**注意事项：**

jieba.cut 方法接受三个输入参数: 需要分词的字符串；`cut_all` 参数用来控制是否采用全模式；HMM 参数用来控制是否使用 HMM 模型。

`jieba.cut_for_search` 方法接受两个参数：需要分词的字符串；是否使用 HMM
模型。该方法适合用于搜索引擎构建倒排索引的分词，粒度比较细。

### HanLP 分词

#### pyhanlp 安装

其为 HanLP 的 Python 接口，支持自动下载与升级 HanLP，兼容 Python2、Python3。

安装命令为 `pip install pyhanlp`，使用命令 hanlp 来验证安装。

pyhanlp 目前使用 jpype1 这个 Python 包来调用 HanLP，如果遇到：

> building '_jpype' extensionerror: Microsoft Visual C++ 14.0 is required. Get
> it with "Microsoft VisualC++ Build Tools":
> http://landinghub.visualstudio.com/visual-cpp-build-tools

**则推荐利用轻量级的 Miniconda 来下载编译好的 jpype1。**

    
    
        conda install -c conda-forge jpype1
        pip install pyhanlp
    

**未安装 Java 时会报错** ：

> jpype._jvmfinder.JVMNotFoundException: No JVM shared library file (jvm.dll)
> found. Try setting up the JAVA_HOME environment variable properly.

HanLP 主项目采用 Java 开发，所以需要 Java 运行环境，请安装 JDK。

#### 命令行交互式分词模式

在命令行界面，使用命令 hanlp segment 进入交互分词模式，输入一个句子并回车，HanLP 会输出分词结果：

![enter image description
here](http://images.gitbook.cn/009c2f60-616f-11e8-b864-0bd1f4b74dfb)

可见，pyhanlp 分词结果是带有词性的。

#### 服务器模式

通过 hanlp serve 来启动内置的 HTTP 服务器，默认本地访问地址为：http://localhost:8765 。

![enter image description
here](http://images.gitbook.cn/d29d52f0-616f-11e8-b864-0bd1f4b74dfb)

![enter image description
here](http://images.gitbook.cn/e79a06b0-6171-11e8-b864-0bd1f4b74dfb)

也可以访问官网演示页面：<http://hanlp.hankcs.com/>。

#### 通过工具类 HanLP 调用常用接口

通过工具类 HanLP 调用常用接口，这种方式应该是我们在项目中最常用的方式。

（1）分词

    
    
        from pyhanlp import *
        content = "现如今，机器学习和深度学习带动人工智能飞速的发展，并在图片处理、语音识别领域取得巨大成功。"
        print(HanLP.segment(content))
    

结果为：

> [现如今/t, ，/w, 机器学习/gi, 和/cc, 深度/n, 学习/v, 带动/v, 人工智能/n, 飞速/d, 的/ude1, 发展/vn,
> ，/w, 并/cc, 在/p, 图片/n, 处理/vn, 、/w, 语音/n, 识别/vn, 领域/n, 取得/v, 巨大/a, 成功/a, 。/w]

（2）自定义词典分词

在没有使用自定义字典时的分词。

    
    
        txt = "铁甲网是中国最大的工程机械交易平台。"
        print(HanLP.segment(txt))
    

结果为：

> [铁甲/n, 网/n, 是/vshi, 中国/ns, 最大/gm, 的/ude1, 工程/n, 机械/n, 交易/vn, 平台/n, 。/w]

添加自定义新词：

    
    
        CustomDictionary.add("铁甲网")
        CustomDictionary.insert("工程机械", "nz 1024")
        CustomDictionary.add("交易平台", "nz 1024 n 1")
        print(HanLP.segment(txt))
    

结果为：

> [铁甲网/nz, 是/vshi, 中国/ns, 最大/gm, 的/ude1, 工程机械/nz, 交易平台/nz, 。/w]

当然了，jieba 和 pyhanlp 能做的事还有很多，关键词提取、自动摘要、依存句法分析、情感分析等，后面章节我们将会讲到，这里不再赘述。

参考文献：

  1. <https://github.com/fxsjy/jieba>
  2. <https://github.com/hankcs/pyhanlp>

### 示例数据下载

> <https://github.com/sujeek/chinese_nlp>

