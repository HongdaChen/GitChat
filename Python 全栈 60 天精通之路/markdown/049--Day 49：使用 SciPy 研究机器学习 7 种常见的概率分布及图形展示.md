上一天介绍机器学习任务中常用概念，使用 Kaggle 上的数据集阐述概念的具体意义，介绍如何使用训练数据和测试数据实践机器学习任务的 pipeline 等。

今天一起探索机器学习任务实践前的其他必知必备知识，主要包括大多数机器学习结果可行性成立的基本前提，由此引出数据分布和常见的离散性随机分布、连续性随机分布，借助
SciPy 包展示常见分布。

一起开始今天的学习。

### 独立同分布

“独立同分布”这个概念在各大书籍、网络中非常常见，它的英文名称：independent and identically distributed，简称为
i.i.d。首先确保以后读文献、看博客等要认识这种简写。

分别解释什么是独立、什么是同分布。

抛掷一枚硬币，记出现正面为事件 $X$，事件 $X$ 发生的概率为 $P(X)$ 且为 0.4。接下来开始做抛掷硬币实验，第一次抛掷硬币出现正面的概率为
0.4，第二次抛掷硬币出现正面的概率 $P(X_2)$ 也为 0.4，第 $i$ 次正面出现概率 $ P(X_i)$ 也为 0.4，也就说每第 $i$
次抛掷与前面 $i-1$ 次抛掷无任何关系，与它们相互独立。

如果抛掷一枚智能硬币，如果第 $i-1$ 次出现反面，则第 $i$ 次一定为正面；如果第 $i-1$ 次出现正面，则第 $i$
次一定为反面。那么这就不是独立的，第 $i$ 次出现正反面强依赖于第 $i-1$ 次的实验结果。

同分布是指每次抛掷试验，使用的都是这枚正面出现概率 $P(X)$ 为 0.4 的硬币。相对的，如果第 $i-1$ 次使用这枚硬币，而第 $i$
次抛掷却实用正面出现概率 $P(X)$ 为 0.6 的硬币，那么这就不是同分布。

### 为什么关心分布

大部分机器学习算法都是根据已有历史数据，去学习它们的分布规律，也就是分布的参数。一旦学习到分布的参数后，当再进来新的、未知的数据时，学到的算法模型便会预测或决策出一个结果。这是大部分机器学习的学习过程。

请留意，评价机器学习模型好快使用的数据是新进来的、对模型完全未知的数据。

考虑下面这种情况，如果我们再拿训练使用的数据来评价模型好快时，得分肯定高，但是完全没有意义，相信也不会有人这么做，因为它们已经对模型完全学习到、完全已熟悉。

再考虑另一种情况，如果测试用的数据来自完全不同的数据分布，模型预测它们的结果得分往往不会好，虽然也会得到一个分数。测试数据集的分布和训练数据集的数据分布差异太大，训练的模型即便泛化的再好，预测与己分布差异很大数据时也无能为力。

基于以上两种极端情况，我们的希望便是测试数据集要尽可能匹配训练模型所使用的的数据分布，在这个前提下，再去优化调参模型才更有意义，努力才不会白费。

### 常见分布

如果满足前提：训练数据集和测试数据集的分布近似相同，算法模型才更能发挥威力，所以常见的数据分布很有必要去研究去了解。

#### **正态分布**

提到数据分布，大多数读者可能脑海里会闪现出下面这幅图，不过长时间没接触过数理统计的读者可能会遗忘 $y$ 轴含义。

![](https://images.gitbook.cn/c5eb90d0-db8e-11ea-bd81-030d631eb386)

假定 $x$ 是一个连续型随机变量，$y$ 就表示 $x$ 的概率密度函数，说的直白些，它是取值密集情况的一种度量。下图是标准正态分布的 pdf 曲线，在
$x$ 等于 0 处，$y$ 值最大。

首先附上绘制上面图形的过程，导入包：

    
    
    import matplotlib.pyplot as plt
    from scipy import stats 
    

使用 scipy.stats 的 norm 方法，loc 和 scale 默认值分别 0 和 1，也就是标准正态分布情况。

    
    
    plt.figure(figsize=(12,8))
    x = np.linspace(-5,5,100)
    y = stats.norm.pdf(x) # pdf 表示概率密度函数
    plt.grid()
    plt.xlabel('x')
    plt.ylabel('pdf')
    plt.title('probability density function')
    plt.xticks(ticks=np.arange(-5,5))
    plt.plot(x,y,color='blue')
    

使用 SciPy 会很方便地求出各个关键取值，比如计算 $x$ 等于 0.5 时概率密度值，结果为：0.352。

    
    
    stats.norm.ppf(0.5) 
    

使用 ppf 方法，求出累计分布函数（也就是曲线与x轴所围成面积）等于 0.5 时，$x$ 的值，显然面积值等于一半时，$x$ 等于 0.5。

    
    
    stats.norm.ppf(0.5) 
    

求出标准正态分布的均值和标准差：

    
    
    stats.norm.mean() # 0.
    stats.norm.std() # 1.
    

上面提到累计分布函数，它的定义为 $P(X<x)$，密度的累积，绘制标准正态分布的累计分布函数：

![](https://images.gitbook.cn/d8b4ada0-db8e-11ea-9369-b5210af59199)

绘制代码如下：

    
    
    plt.figure(figsize=(12,8))
    x = np.linspace(-5,5,100)
    y_cdf = stats.norm.cdf(x)
    plt.grid()
    plt.xlabel('x')
    plt.ylabel('cdf')
    plt.title('cumulative distribution function')
    plt.xticks(ticks=np.arange(-5,5))
    plt.plot(x,y_cdf,color='blue')
    

正态分布应用广泛，现实中当我们对数据分布一无所知时，往往假定数据符合正态分布。今天介绍一个使用正态分布生成实验使用的几簇数据集，用于做聚类等任务的实验数据。

基本原理：想生成几簇数据，就创建几条线段，然后在 `y` 上做一个高斯随机偏移。

生成两簇数据如下，使用的另一个关键方法 `rvs`，表示生成满足指定分布、参数指定个数的样本。

    
    
    plt.figure(figsize=(12,8))
    blob1 = 500
    x1 = np.linspace(0.5,3,blob1)
    y1 = 2 * x1 + 10 + stats.norm.rvs(0.,2.,size=(blob1,)) # rvs 生成正态分布的 blob1 个数据样本
    
    blob2 = 800
    x2 = np.linspace(5,8,blob2) 
    y2 = 2 * x2 - 4 + stats.norm.rvs(0.,3.0,size=(blob2,))
    
    plt.scatter(x1,y1,label='cluster0')
    plt.scatter(x2,y2,label='cluster1')
    plt.legend()
    plt.show()
    

![](https://images.gitbook.cn/ead99b30-db8e-11ea-856c-1b0ca96fbf95)

同样方法，生成如下三簇数据：

    
    
    plt.figure(figsize=(12,8))
    blob1 = 500
    x1 = np.linspace(0.5,3,blob1)
    y1 = 2 * x1 + 10 + stats.norm.rvs(0.,2.,size=(blob1,))
    
    blob2 = 800
    x2 = np.linspace(5,8,blob2) 
    y2 = 2 * x2 - 4 + stats.norm.rvs(0.,3.0,size=(blob2,))
    
    blob3 =300
    x3 = np.linspace(2,3,blob3) 
    y3 = 2 * x3 -1 + stats.norm.rvs(0.,1.0,size=(blob3,))
    
    plt.scatter(x1,y1,label='cluster0')
    plt.scatter(x2,y2,label='cluster1')
    plt.scatter(x3,y3,label='cluster2')
    
    plt.legend()
    plt.show()
    

![](https://images.gitbook.cn/0051ab60-db8f-11ea-8b08-a305d0150119)

正态分布的两个参数 loc 和 scale 分别表示均值和标准差，几种不同组合的对比分析：

    
    
    plt.figure(figsize=(12,8))
    x = np.linspace(-5,5,100)
    y1 = stats.norm.pdf(x)
    y2 = stats.norm.pdf(x,loc=0.0,scale=2.)
    y3 = stats.norm.pdf(x,loc=1.0,scale=2.)
    plt.grid()
    plt.xlabel('x')
    plt.ylabel('pdf')
    plt.title('probability density function')
    plt.xticks(ticks=np.arange(-5,5))
    plt.plot(x,y1,label='u=0;std=1')
    plt.plot(x,y2,label='u=0;std=2')
    plt.plot(x,y3,label='u=1,std=2')
    plt.legend()
    plt.show()
    

![](https://images.gitbook.cn/178c7a30-db8f-11ea-b89a-ffa3ff92b2c1)

方差越大，取值越离散，表现出来的形状就更矮胖。

#### **拉普拉斯分布**

标准的正态分布概率密度函数为：

![](https://images.gitbook.cn/27d2e3c0-db8f-11ea-bedf-6d5f234f6331)

标准的拉普拉斯分布的概率密度函数为：

![](https://images.gitbook.cn/387f3b10-db8f-11ea-9f8d-6db671417c55)

如果仅仅对比上面的概率密度公式仍然没有感觉，下面绘图对比两者的区别：

    
    
    plt.figure(figsize=(12,8))
    x = np.linspace(-5,5,100)
    y1 = stats.norm.pdf(x)
    y2 = stats.laplace.pdf(x)
    plt.grid()
    plt.xlabel('x')
    plt.ylabel('pdf')
    plt.title('probability density function')
    plt.xticks(ticks=np.arange(-5,5))
    plt.plot(x,y1,color='blue',label='norm')
    plt.plot(x,y2,color='red',label='laplace')
    plt.legend()
    plt.show()
    

![](https://images.gitbook.cn/4a8f8f30-db8f-11ea-bd41-fdeec52ccca1)

比较顶端，正态分布相比拉普拉斯分布更加平滑，拉普拉斯概率分布却形成一个尖端。

关于两者的对比应用之一就是正则化，其中 L1 正则化可看做是拉普拉斯先验，L2
正则化看作是正态分布的先验。所以要想深度理解正则化，首先要理解这两个概率分布。

拉普拉斯的两个形状参数与正态分布意义相似：

![](https://images.gitbook.cn/5ca847c0-db8f-11ea-8d16-77936a705f77)

绘制上图的代码：

    
    
    plt.figure(figsize=(12,8))
    x = np.linspace(-5,5,100)
    y1 = stats.laplace.pdf(x)
    y2 = stats.laplace.pdf(x,loc=0.0,scale=2.)
    y3 = stats.laplace.pdf(x,loc=1.0,scale=2.)
    plt.grid()
    plt.xlabel('x')
    plt.ylabel('pdf')
    plt.title('probability density function')
    plt.xticks(ticks=np.arange(-5,5))
    plt.plot(x,y1,label='u=0;r=1')
    plt.plot(x,y2,label='u=0;r=2')
    plt.plot(x,y3,label='u=1,r=2')
    plt.legend()
    plt.show()
    

#### **伯努利分布**

理解完相对复杂一些的两种分布，接下来伯努利相对更加简单，首先它描述的是离散型变量且发生1次的概率分布，且 $X$ 取值只有 2 个，要么为 0，要么为
1。且 $P(X=1) = p$，$P(X=0) = 1-p$，其中 $p$ 为此分布的唯一参数：

![](https://images.gitbook.cn/73421ba0-db8f-11ea-bedf-6d5f234f6331)

    
    
    bern = stats.bernoulli(0.4)
    bern.rvs(size=(10,))
    

创建分布参数 $p=0.4$ 的伯努利分布，生成满足此分布的 10 个样本点：

    
    
    array([0, 0, 1, 0, 1, 1, 0, 0, 1, 0])
    

#### **二项分布**

二项分布也是假设试验只有两种结果，分布参数 $p$，且 $P(X=1)=p$，$P(X=0)=1-p$。不过请注意，二项分布描述的事件是试验独立重复地进行
$n$ 次，且出现 $X=1$ 的次数为 $x$ 次的概率：

![](https://images.gitbook.cn/83e8a690-db8f-11ea-9f8d-6db671417c55)

独立的重复 50 次试验，三个不同 $p$ 参数的对比分析图如下所示：

![](https://images.gitbook.cn/95628710-db8f-11ea-856c-1b0ca96fbf95)

绘图代码如下：

    
    
    plt.figure(figsize=(12,8))
    x = np.arange(1,51)
    y1 = stats.binom.pmf(x,p=0.4,n=50)
    y2 = stats.binom.pmf(x,p=0.6,n=50)
    y3 = stats.binom.pmf(x,p=0.8,n=50)
    plt.grid()
    plt.xlabel('x')
    plt.ylabel('pdf')
    plt.title('probability mass function')
    plt.xticks(ticks=np.arange(1,51,2))
    plt.plot(x,y1,label='p=0.4')
    plt.plot(x,y2,label='p=0.6')
    plt.plot(x,y3,label='p=0.8')
    plt.legend()
    plt.show()
    

三种不同分布参数的二项分布均值和标准差分别为：

    
    
    mean,std = stats.binom.stats(n=50,p=0.4)
    mean,std
    #(array(20.), array(12.))
    

#### **均匀分布**

均匀分布的变量可以是离散的，也可以是连续的。

离散型变量均匀分布的概率质量函数为：

![](https://images.gitbook.cn/a824c610-db8f-11ea-bedf-6d5f234f6331)

    
    
    uniform_int = stats.randint(1,10)
    uniform_int.mean(),uniform_int.std() # 均值，标准差
    # (5.0, 2.581988897471611)
    

#### **泊松分布**

假设已知事件在单位时间（或者单位面积）内发生的平均次数为 λ，则泊松分布描述了事件在单位时间（或者单位面积）内发生的具体次数为 $k$ 的概率。

概率质量函数如下，其中 $k>0$：

![](https://images.gitbook.cn/baba28b0-db8f-11ea-bedf-6d5f234f6331)

分布参数 $\mu$ 表示发生次数的期望值，如下绘制不同 $\mu$ 值下的概率质量分布图：

    
    
    plt.figure(figsize=(12,8))
    x = np.arange(1,51)
    y1 = stats.poisson.pmf(x,10.)
    y2 = stats.poisson.pmf(x,20)
    y3 = stats.poisson.pmf(x,30)
    plt.grid()
    plt.xlabel('x')
    plt.ylabel('pmf')
    plt.title('probability mass function')
    plt.xticks(ticks=np.arange(1,51,2))
    plt.plot(x,y1,label='u=10')
    plt.plot(x,y2,label='u=20')
    plt.plot(x,y3,label='u=30')
    plt.legend()
    plt.show()
    

![image-20200425172655106](https://images.gitbook.cn/5caac720-883d-11ea-987d-cf5e8d931cfa)

泊松分布与二项分布的对比如下：

    
    
    plt.figure(figsize=(12,8))
    x = np.arange(1,51)
    y1 = stats.binom.pmf(x,p=0.4,n=50)
    y2 = stats.poisson.pmf(x,20)
    plt.grid()
    plt.xlabel('x')
    plt.ylabel('pmf')
    plt.title('probability mass function')
    plt.xticks(ticks=np.arange(1,51,2))
    plt.plot(x,y1,label='binom:p=0.4,n=50')
    plt.plot(x,y2,label='u=20')
    plt.legend()
    plt.show()
    

![](https://images.gitbook.cn/cd63fdb0-db8f-11ea-bedf-6d5f234f6331)

泊松分布在现实生活中的应用也极为广泛，比如交通流的预测，医学中某个区域内某种细胞的个数等等，感兴趣的读者可去：

> <https://www.zhihu.com/question/26441147>

查看更加详细的实际应用解释，在此不再详细展开。

#### **指数分布**

若事件 $X$ 服从泊松分布，则该事件前后两次发生的时间间隔事件 $T$ 服从指数分布。由于时间间隔是浮点数，因此指数分布是连续分布，其概率密度函数为：

![](https://images.gitbook.cn/df919dd0-db8f-11ea-8d40-2b3467c34b4c)

标准形式为如下：

![](https://images.gitbook.cn/ee26ade0-db8f-11ea-8b08-a305d0150119)

    
    
    plt.figure(figsize=(12,8))
    x = np.linspace(1,11,100)
    y1 = stats.expon.pdf(x,0.5)
    plt.grid()
    plt.xlabel('x')
    plt.ylabel('pmf')
    plt.title('probability density function')
    #plt.scatter(x,y1,color='',edgecolor='orange')
    plt.plot(x,y1)
    plt.show()
    

![](https://images.gitbook.cn/006c42d0-db90-11ea-bedf-6d5f234f6331)

指数分布的期望和方差分别为：$\frac{1}{\lambda}$ 和 $\frac{1}{\lambda^2}$。

### 小结

今天主要介绍独立同分布概念，数据分布训练集和测试集上一致的重要性，以及常见的 7 种概率分布，其中常见的离散型随机变量分布有：

  * 伯努利分布
  * 二项分布
  * 泊松分布
  * 均匀分布

连续型分布：

  * 正态分布
  * 拉普拉斯分布
  * 均匀分布
  * 指数分布

今天所有代码都整理到 notebook 中，下载链接：

> <https://pan.baidu.com/s/1K66d29iYdKjTyA8woPfFXw>
>
> 提取码：2syy

