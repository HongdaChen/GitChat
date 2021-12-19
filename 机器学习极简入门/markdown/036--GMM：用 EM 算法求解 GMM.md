所谓高斯混合模型（GMM），顾名思义，就是将若干个概率分布为高斯分布的分模型混合在一起的模型。

在具体讲解 GMM 之前，我们先来看看高斯分布。

### 高斯分布

#### 高斯分布密度函数

高斯分布（Gaussian Distribution），又名正态分布（Normal distribtion），它的密度函数为：

$f(x;\mu,{\sigma^2}) = \frac{1}{\sqrt{2\pi{\sigma^2}}} \, \exp \left(
-\frac{(x- \mu)^2}{2\sigma^2} \right)$

分布形式如下图（四个不同参数集的高斯分布概率密度函数）：

![enter image description
here](https://images.gitbook.cn/b70b1710-9497-11e8-8abb-7f35b2559c3a)

高斯分布的概率密度函数曲线呈钟形，因此人们经常将其称之为钟形曲线（类似于寺庙里的大钟，因此得名）。

图中红色曲线是 $\mu =0$、$\sigma^2 = 1$ 的高斯分布，这个分布有个专门的名字： **标准高斯分布** 。

#### 常见的分布

高斯分布是一种非常常见的概率分布，经常被用来定量自然界的现象。

现实生活中的许多自然现象都被发现近似地符合高斯分布，比如人类的寿命、身高、体重、智商等和我们生活息息相关的数据。

不止是人类体征或者生物特征，在金融、科研、工业等各个领域都有大量现实业务产生的数据被证明是符合高斯分布的。

#### 中心极限定理

##### 高斯分布的重要性质

高斯分布有一个 **非常重要的性质**
：在适当的条件下，大量相互独立的随机变量的均值经适当标准化后，依分布收敛于高斯分布（即使这些变量自己的分布并不是高斯分布）——这就是 **中心极限定理**
。

严格说起来，中心极限定理并不是一个定理，而是一类定理。

这类定理从数学上 **证明**
了：在自然界与人类生产活动中，一些现象受到许多相互独立的随机因素的影响，当每个因素所产生的影响都很微小时，总的影响可以看作是服从高斯分布的。

##### **经典中心极限定理**

中心极限定理中，最常用也最简单的是 **经典中心极限定理** ，这一定理说明了什么，我们看下面的解释。

设 $(x_1, …, x_n)$ 为一个独立同分布的随机变量样本序列，且这些样本值的期望为 $\mu$，有限方差为 $\sigma^2 $。

$S_n$ 为这些样本的算术平均值：

$S_{n} ={\frac {x_{1}+\cdots +x_{n}}{n}}$

> 注意：一般情况下 $n \geqslant 30$，而 $\mu$ 是 $S_n$ 的极限。

随着 $n$ 的增大，$\sqrt{n}(S_n − \mu)$ 的分布逐渐近似于均值为 $0$、方差是 $\sigma^2$ 的高斯分布，即：

${\sqrt {n}}\left(S_{n}-\mu \right)\ {\xrightarrow {d}}\ N\left(0,\sigma
^{2}\right)$

也就是说，无论 $x_i$ 的自身分布是什么，随着 $n$ 变大，这些样本平均值经过标准化处理——$\sqrt{n}(S_n −
\mu)$——后的分布，都会 **逐步接近高斯分布** 。

##### **一个例子**

定理说起来有点抽象，我们来看一个例子就明白了。

我们用一个依据均匀概率分布生成数字的生成器，来生成$0$到$100$之间的数字。

生成器每次运行都连续生成 $N$ 个数字——我们将每次运行称为一次“尝试”。

我们让这个生成器连续尝试$500$次，每次尝试后都计算出本次生成的 N 个数字的均值。

最后，将$500$次的统计结果放入二维坐标系，$x$ 轴（横轴）表示一次尝试中 $N$ 个数字的均值，而 $y$ 轴（纵轴）表示均值为 $x$
的尝试出现的次数（观察频次，Observed Frenquncy）。

下图展示了 $N=30、N=100$ 和 $N=250$ 时的三种情况。直观可见，分布基本上都是钟形曲线，而且 $N$ 越大，曲线越平滑稳定。

![enter image description
here](https://images.gitbook.cn/05338eb0-a503-11e8-80dc-8d254ca863fe)

##### **另一个例子**

再来看一个抛硬币的例子，这是最早发现的中心极限定理的特例，由法国数学家棣莫弗（Abraham de Moivre）发表在1733年出版的论文里。

一个人掷硬币，每次 TA 都一下子抛出一大把 $n$ 枚硬币，然后统计落地后“头”（Head，印有人头像的一面）朝上硬币出现的个数。

总共抛掷很多次，那么这一系列投掷活动中硬币“头”朝上的可能性（$ProportionOfHeads =
\frac{每次正面朝上个数}{n}$）将形成一条高斯曲线（如下图）:

![enter image description
here](https://images.gitbook.cn/b6ef7820-a121-11e8-9a44-c381fc9d5498)

#### 近似为高斯分布

中心极限定理是数理统计学和误差分析的理论基础，指出了大量随机变量之和近似服从高斯分布的条件。

这一定理的 **重要性** 在于：根据它的结论， **其他概率分布可以用高斯分布作为近似** 。

例如：

  * 参数为 $n$ 和 $p$ 的二项分布，在 $n$ 相当大而且 $p$ 接近 $0.5$ 时，近似于 $\mu = np$、$\sigma^2 = np (1 - p)$ 的高斯分布。

  * 参数为 $\lambda$ 的泊松分布，当取样样本数很大时，近似于 $\mu = \lambda$、$\sigma^2 = \lambda$ 的高斯分布。

这就使得高斯分布在事实上成为了一个 **方便模型** ——如果对某一变量做定量分析时其确定的分布情况未知，那么不妨先假设它服从高斯分布。

如此一来，当我们遇到一个问题的时候，只要掌握了大量的观测样本，都可以按照服从高斯分布来处理这些样本。

### 高斯混合模型（GMM）

#### 高斯分布的混合

鉴于高斯分布的广泛应用和独特性质，将它作为概率分布分模型混合出来的模型，自然可以处理很多的实际问题。

比如我们上一篇举的例子：

![enter image description
here](https://images.gitbook.cn/be172ba0-93ea-11e8-968c-a5eca6168c1e)

这里面的三个混合在一起的簇，无论原本每一簇自身的分布如何，我们都可以用高斯模型来近似表示它们。

因此整个混合模型，也就可以是一个 **高斯混合模型（GMM）** ——用高斯密度函数代替上一篇给出的混合模型中的 $\phi_k(\cdot)$，也就是：

$\phi_k(x|\mu_k, {\sigma^2}_k) = \frac{1}{\sqrt{2\pi{\sigma^2}_k}} \, \exp
\left( -\frac{(x- \mu_k)^2}{2{\sigma^2}_k} \right)$，

$\theta_k = (\mu_k, {\sigma^2}_k)$。

则 **GMM** 的形式化表达为：

$P(x|\theta) = \sum_{k=1}^{K}\alpha_k \cdot \phi_k(x | \theta_k) =
\sum_{k=1}^{K} \frac{\alpha_k}{\sqrt{2\pi{\sigma^2}_k}} \, \exp \left(
-\frac{(x- \mu_k)^2}{2{\sigma^2}_k} \right)$

其中，$\alpha_k$ 是系数，$ \alpha_k \geqslant 0$，$\sum_{k=1}^{K}\alpha_k = 1$。

#### GMM 的对数似然函数

混合模型的学习目标为：

$\arg\min{LL(\Theta|X,Z)}$

套用前一篇已经给出的对数似然函数的形式，GMM 的对数似然函数为：

$LL(\Theta|X,Z) = \log{P(X,Z|\Theta)} $

$= \sum_{k=1}^{K}[n_k \log{\alpha_k} + \sum_{i=1}^{N} z_{ik}
\log{(\phi_k(x^{(i)}|\theta_k))}] $

$= \sum_{k=1}^{K}[n_k \log{\alpha_k} + \sum_{i=1}^{N} z_{ik}
[\log{(\frac{1}{\sqrt{2\pi}})} - \frac{1}{2}\log{{\sigma^2}_k} -
\frac{1}{2{\sigma^2}_k}(x^{(i)} - \mu_k)^2]]$

其中：$\Theta = (\theta_k)$，$X = (x_i)$，$Z = (z_{ik})$。

![](https://images.gitbook.cn/b01a8ca0-abf4-11e8-b90c-4d816df0b568)

$i=1,2, …, N$；$ k = 1, 2, …, K$；

并有 $n_k = \sum_{i=1}^{N}z_{ik}$；$ \sum_{k=1}^{K} n_k = N$。

现在的这个对数似然函数中，$x^{(i)}$ 是已经观测到的样本观测数据，是已知的，$z_{ik}$ 却是未知的。

训练样本是“不完整的”，有没被观测到的隐变量（Hidden Variable）存在，这样的对数似然函数需要用之前学过的 EM 算法来优化。

### 用 EM 算法学习 GMM 的参数

#### 将 EM 算法应用于 GMM

最优化 GMM 目标函数的过程就是一个典型的 EM 算法，一共分为4步：

  * 各参数取初始值开始迭代；
  * E 步；
  * M 步；
  * 重复 E 步和 M 步，直到收敛。

过程清晰明确，关键就是 E 步和 M 步的细节。

#### GMM 的 E 步

E 步的任务是求 $Q(\Theta,\Theta^t)$。

按照 EM 算法的描述：

$Q(\Theta,\Theta^t) = E_{Z | X, \Theta^t}LL(\Theta|X,Z) = E_{Z | X,
\Theta^t}[\log{P(X,Z|\Theta)}] $

其中 $Z$ 为隐变量。

在 GMM 中，隐变量为 $Z = (z_{ik}), \;\; i=1,2, …, N； k = 1, 2, …, K$。

因此：

$Q(\Theta,\Theta^t) $

$= E_{Z | X, \Theta^t}[\log{P(X, Z|\Theta)}] $

$ = E_{Z | X, \Theta^t}\\{ \sum_{k=1}^{K}[n_k \log{\alpha_k} + \sum_{i=1}^{N}
z_{ik} \log{(\phi_k(x^{(i)}|\theta_k))}]\\}$

$=E_{Z | X, \Theta^t}\\{ \sum_{k=1}^{K}[n_k \log{\alpha_k} + \sum_{i=1}^{N}
z_{ik} [\log{(\frac{1}{\sqrt{2\pi}})} - \log{\sigma_k} -
\frac{1}{2\sigma_k^2}(x^{(i)} - \mu_k)^2]]\\} $

$ = \sum_{k=1}^{K}[\sum_{i=1}^n E(z_{ik}|X, \Theta^t) \log{\alpha_k} +
\sum_{i=1}^{N} E(z_{ik}|X, \Theta^t) [\log{(\frac{1}{\sqrt{2\pi}})} -
\log{\sigma_k} - \frac{1}{2\sigma_k^2}(x^{(i)} - \mu_k)^2]]$

$E(z_{ik}|X, \Theta^t)$ 是当前模型参数 $\Theta^t$ 下，第 $i$ 个观测数据——$x^{(i)}$——来自第 $k$
个分类的概率期望。

$E(z_{ik}|X, \Theta^t)$ 又叫做分模型 $k$ 对观测数据 $x^{(i)}$ 的影响度。这里需要计算它，为了方便表示，记作
$\hat{z_{ik}}$。

$\hat{z_{ik}} = P(z_{ik} = 1| X, \Theta^t) $

$= \frac{P(z_{ik} = 1, x^{(i)} | \Theta^t)}{\sum_{k=1}^{K} P(z_{ik} = 1,
x^{(i)} | \Theta^t)} $

$= \frac{P( x^{(i)} |z_{ik} = 1, \Theta^t)P(z_{ik} =
1|\Theta^t)}{\sum_{k=1}^{K} P(x^{(i)} | z_{ik} = 1, \Theta^t)P(z_{ik} =
1|\Theta^t)} $

$= \frac{\alpha_k^t \phi(x^{(i)}|\theta_k^t)}{\sum_{k=1}^{K} \alpha_k^t
\phi(x^{(i)}|\theta_k^t)} $

其中，$i = 1,2,..., N$；$k = 1,2,..., K$。

这里的 $x^{(i)}$ 是样本观测值，$\alpha_k^t$ 和 $\theta_k^t$ 是上一个迭代的 M
步计算出来的参数估计值，都是已知的，因此可以直接求解 $\hat{z_{ik}} $ 的值。

求出 $\hat{z_{ik}}$ 的解后，再将 $\hat{z_{ik}} = E(z_{ik}|X, \Theta^t)$ 以及 $n_k =
\sum_{i=1}^{N} E(z_{ik}|X, \Theta^t)$ 代回到 $Q(\Theta,\Theta^t) $式子里，得到：

$Q(\Theta,\Theta^t) = \sum_{k=1}^{K}\\{n_k\log{\alpha_k} +
\sum_{i=1}^{N}\hat{z_{ik}} [\log{(\frac{1}{\sqrt{2\pi}})} - \log{\sigma_k} -
\frac{1}{2\sigma_k^2}(x^{(i)} - \mu_k)^2]\\}$

至此 $Q(\Theta,\Theta^t)$ 成了关于 $ \alpha_k$ 和 $\theta_k =( \mu_k, {\sigma^2}_k)$
的函数，其中 $k = 1,2, ..., K$。

#### GMM 的 M 步

M 步的任务是： $\arg{\max_{\Theta} {Q(\Theta,\Theta^t)}}$。

根据上一步得出的结果，我们看到 $Q(\Theta,\Theta^t) $ 里面含有未知的参数 $\alpha_k, \mu_k,
{\sigma^2}_k, k = 1,2, ..., K$。

我们可以把这一步的任务看作是极大化一个包含着若干自变量的函数，那么就拿出我们惯常的办法——通过分别对各个自变量求偏导，再令导数为0，来求取各自变量的极值，然后再带回到函数中去求整体的极值。

> 注意：采用这种办法要注意，对 $\mu_k$ 和 $\sigma_k^2$ 直接求偏导，并令其为$0$即可。
>
> 对于 $\alpha_k$ 是在 $\sum_{k=1}^{K} \alpha_k = 1$ 的条件下求偏导数，此处可用拉格朗日乘子法。

最后得出表示 $\Theta_{t+1}$ 的各参数的结果为：

${\mu_k}^{t+1} = \frac{\sum_{i=1}^{N} \hat{z_{ik}} x^{(i)}}{ \sum_{i=1}^{N}
\hat{z_{ik}}}, \;\;k=1,2,..., K$

${{\sigma^2}_k}^{t+1} = \frac{\sum_{i=1}^{N} \hat{z_{ik}} (x^{(i)} -
\mu_k^{t+1})^2}{ \sum_{i=1}^{N} \hat{z_{ik}}}, \;\;k=1,2,..., K$

${\alpha_k}^{t+1} = \frac{n_k}{N} = \frac{\sum_{i=1}^{N} \hat{z_{ik}}}{N},
\;\;k=1,2,..., K$

### GMM 实例

下面我们来看一个 GMM 的例子：

    
    
        from matplotlib.pylab import array,diag
        import matplotlib.pyplot as plt
        import matplotlib as mpl
        import pypr.clustering.gmm as gmm
        import numpy as np
        from scipy import linalg
    
        from sklearn import mixture
    
        # 将样本点显示在二维坐标中
        def plot_results(X, Y_, means, covariances, colors, eclipsed, index, title):
            splot = plt.subplot(2, 1, 1 + index)
            for i, (mean, covar, color) in enumerate(zip(
                    means, covariances, colors)):
                v, w = linalg.eigh(covar)
                v = 2. * np.sqrt(2.) * np.sqrt(v)
                u = w[0] / linalg.norm(w[0])
                # as the DP will not use every component it has access to
                # unless it needs it, we shouldn't plot the redundant
                # components.
                if not np.any(Y_ == i):
                    continue
                plt.scatter(X[Y_ == i, 0], X[Y_ == i, 1], .8, color=color)
    
                if eclipsed:
                    # Plot an ellipse to show the Gaussian component
                    angle = np.arctan(u[1] / u[0])
                    angle = 180. * angle / np.pi  # convert to degrees
                    ell = mpl.patches.Ellipse(mean, v[0], v[1], 180. + angle, color=color)
                    ell.set_clip_box(splot.bbox)
                    ell.set_alpha(0.5)
                    splot.add_artist(ell)
            plt.xticks(())
            plt.yticks(())
            plt.title(title)
    
        #创建样本数据，一共3个簇
        mc = [0.4, 0.4, 0.2]
        centroids = [ array([0,0]), array([3,3]), array([0,4]) ]
        ccov = [ array([[1,0.4],[0.4,1]]), diag((1,2)), diag((0.4,0.1)) ]
    
        X = gmm.sample_gaussian_mixture(centroids, ccov, mc, samples=1000)
    
        #用plot_results函数显示未经聚类的样本数据
        gmm = mixture.GaussianMixture(n_components=1, covariance_type='full').fit(X)
        plot_results(X, gmm.predict(X), gmm.means_, gmm.covariances_, ['grey'],False, 0, "Sample data")
    
        #用EM算法的GMM将样本分为3个簇，并按不同颜色显示在二维坐标系中
        gmm = mixture.GaussianMixture(n_components=3, covariance_type='full').fit(X)
        plot_results(X, gmm.predict(X), gmm.means_, gmm.covariances_, ['red', 'blue', 'green'], True, 1, "GMM clustered data")
    
        plt.show()
    

最后的显示结果如下：

![enter image description
here](https://images.gitbook.cn/4e5ce820-9603-11e8-afe4-2b97d4c05a56)

注意：上面代码中用到了 PyPR 聚类包，该包基于 Python 2 实现。使用命令：pip3 install
pypr，可以对其进行安装，但如果在Python 3 环境中调用该命令，需手动修改 gmm.py 文件中的代码，使其符合 Python 3
语法才可以。你可以尝试自己修改 gmm.py。

如果你使用的是 pypr-0.1.1，可以用以下网址中下载的 gmm.py 文件替换该目录（…\Python36\Lib\site-
packages\pypr\clustering\gmm.py）下的 gmm.py。

>
> <https://github.com/juliali/MachineLearning101/blob/master/pypr-0.1.1-python3-update/gmm.py>

