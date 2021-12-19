### SVM 实例

前面我们学习了 SVM 的理论，讲了线性可分 SVM、线性 SVM、非线性 SVM 和核函数。

在本文，我们将通过几个例子，来直观了解一下这些概念。

我们采用一维特征，这样可以将样本直接对应到直角坐标中的点，看起来非常直观，便于理解。

#### 线性可分 SVM

先来看一个最简单的例子： **线性可分 SVM** ：

    
    
        import numpy as np
        import matplotlib.pyplot as plt
        from sklearn.svm import SVC # "Support vector classifier"
    
        # 定义函数plot_svc_decision_function用于绘制分割超平面和其两侧的辅助超平面
        def plot_svc_decision_function(model, ax=None, plot_support=True):
            """Plot the decision function for a 2D SVC"""
            if ax is None:
                ax = plt.gca()
            xlim = ax.get_xlim()
            ylim = ax.get_ylim()
    
            # 创建网格用于评价模型
            x = np.linspace(xlim[0], xlim[1], 30)
            y = np.linspace(ylim[0], ylim[1], 30)
            Y, X = np.meshgrid(y, x)
            xy = np.vstack([X.ravel(), Y.ravel()]).T
            P = model.decision_function(xy).reshape(X.shape)
    
            #绘制超平面
            ax.contour(X, Y, P, colors='k',
                       levels=[-1, 0, 1], alpha=0.5,
                       linestyles=['--', '-', '--'])
    
            #标识出支持向量
            if plot_support:
                ax.scatter(model.support_vectors_[:, 0],
                           model.support_vectors_[:, 1],
                           s=300, linewidth=1,  edgecolors='blue', facecolors='none');
            ax.set_xlim(xlim)
            ax.set_ylim(ylim)
    
    
        # 用make_blobs生成样本数据
        from sklearn.datasets.samples_generator import make_blobs
        X, y = make_blobs(n_samples=50, centers=2,           
                          random_state=0, cluster_std=0.60)
    
        # 将样本数据绘制在直角坐标中
        plt.scatter(X[:, 0], X[:, 1], c=y, s=50, cmap='autumn');
        plt.show()
    
        # 用线性核函数的SVM来对样本进行分类
        model = SVC(kernel='linear')
        model.fit(X, y)
    
        # 在直角坐标中绘制出分割超平面、辅助超平面和支持向量
        plt.scatter(X[:, 0], X[:, 1], c=y, s=50, cmap='autumn')
        plot_svc_decision_function(model);
        plt.show()
    

用上面的代码生成的数据显示如下：

![enter image description
here](http://images.gitbook.cn/2f553780-745b-11e8-9ce0-8f87da4d301a)

用线性核 SVM 分类后，显示如下：

![enter image description
here](http://images.gitbook.cn/b105f110-745c-11e8-92f7-efdcca03d400)

我们可以看到，因为本来就是线性可分的两类样本，因此最大分割超平面和相应的辅助超平面都很清晰，相应的支持变量也正好落在两类样本的边缘处。

看起来非常理想。

#### 线性 SVM

那么，如果样本之间不是区分的这么清晰，而是有些不同类型样本发生了重叠，又会如何？

让我们再来生成一些需要用软间隔来处理的数据看看。

    
    
        X, y = make_blobs(n_samples=100, centers=2,
                          random_state=0, cluster_std=0.9)
        plt.scatter(X[:, 0], X[:, 1], c=y, s=50, cmap='autumn');
        plt.show()
    

这段代码生成的数据显示如下：

![enter image description
here](http://images.gitbook.cn/31281400-7461-11e8-ac74-0ffa13d0bc74)

我们再用同样的方法去分割它们：

    
    
        model = SVC(kernel='linear')
        model.fit(X, y)
    
        plt.scatter(X[:, 0], X[:, 1], c=y, s=50, cmap='autumn')
        plot_svc_decision_function(model)
        plt.show()
    

结果是这样的：

![enter image description
here](http://images.gitbook.cn/4e2b9950-7461-11e8-9ce0-8f87da4d301a)

我们所用的 SVC 类，有一个 C 参数，对应的是错误项（Error Term）的惩罚系数。这个系数设置得越高，容错性也就越小，分隔空间的硬度也就越强。

C 的 Default Value 是1.0，在没有显性设置的情况下，C=1.0。

在线性可分 SVM 的例子中，C 取默认值就工作得很好了。但是到了现在这里的例子里，直观上，如果用默认值，Error Term
也太多了点，如果我们加大惩罚系数，将它提高10倍，设置 C=10.0，又会如何呢？

让我们来试一试，代码如下：

    
    
        model = SVC(kernel='linear', C=10.0)
        model.fit(X, y)
    
        plt.scatter(X[:, 0], X[:, 1], c=y, s=50, cmap='autumn')
        plot_svc_decision_function(model)
        plt.show()
    

生成图形为：

![enter image description
here](http://images.gitbook.cn/83a51010-7462-11e8-92f7-efdcca03d400)

确实比之前顺眼了不少。这就是惩罚系数的作用。

#### 完全线性不可分的数据

如果我们的样本在二维空间内完全是线性不可分的，又会怎样呢？

比如，我们生成下列这些数据：

    
    
        from sklearn.datasets.samples_generator import make_circles
        X, y = make_circles(100, factor=.1, noise=.1)
    
        plt.scatter(X[:, 0], X[:, 1], c=y, s=50, cmap='autumn');
        plt.show()
    

生成图形为：

![enter image description
here](http://images.gitbook.cn/2080c8f0-7465-11e8-ac74-0ffa13d0bc74)

这样分布的样本，我们指望在二维空间中线性分隔它们，是不可能了。

如果非要尝试一下，那么只能得出如下这种可怜的结果：

    
    
        model = SVC(kernel='linear')
        model.fit(X, y)
    
        plt.scatter(X[:, 0], X[:, 1], c=y, s=50, cmap='autumn')
        plot_svc_decision_function(model);
        plt.show()
    

结果是这样的：

![enter image description
here](http://images.gitbook.cn/9e8fa3a0-7466-11e8-a58c-b35f5fcc908a)

#### 核函数的作用

从上面的样本分布图可以看出，虽然正负例样本在二维空间中完全线性不可分。但是，它们相对集中，如果将它们投射到三维空间中，则很可能是可分的。

我们用下面的代码来可视化一下这些样本投射到三维后的景象：

    
    
        from mpl_toolkits import mplot3d
        def plot_3D(elev=30, azim=30, X=None, y=None):
            ax = plt.subplot(projection='3d')
            r = np.exp(-(X ** 2).sum(1))
            ax.scatter3D(X[:, 0], X[:, 1], r, c=y, s=50, cmap='autumn')
            ax.view_init(elev=elev, azim=azim)
            ax.set_xlabel('x')
            ax.set_ylabel('y')
            ax.set_zlabel('r')
    
        plot_3D(X=X, y=y)
        plt.show()
    

绘制出的三维分布如下：

![enter image description
here](http://images.gitbook.cn/6766e450-7467-11e8-a58c-b35f5fcc908a)

正负例样本在三维空间被分为了两簇。从直观角度看，如果我们在两簇中间的位置“横切一刀”，完全有可能将它们分开。

如何将这些样本在更高维度空间的投射体现在代码中呢？我们需要重新生成二维的 X 再用 SVC fit 吗？

其实不用那么麻烦，我们可以直接采用 **RBF 核** ！

只要将上面的代码修改一个参数，改为：

    
    
        model = SVC(kernel='rbf')
        model.fit(X, y)
    
        plt.scatter(X[:, 0], X[:, 1], c=y, s=50, cmap='autumn')
        plot_svc_decision_function(model);
        plt.show()
    

结果就会好许多：

![enter image description
here](http://images.gitbook.cn/6815f390-7468-11e8-ac74-0ffa13d0bc74)

好像这个结果对于 Error Term 容忍度还是有点高啊，我们再调高一点惩罚系数：

    
    
        model = SVC(kernel='rbf', C=10)
    

结果就是：

![enter image description
here](http://images.gitbook.cn/b81b7c70-7468-11e8-9ce0-8f87da4d301a)

如果把惩罚系数再调高10倍呢？

    
    
        model = SVC(kernel='rbf', C=100)
    

这样看起来，就已经相当合理了：

![enter image description
here](http://images.gitbook.cn/e8901e60-7468-11e8-a58c-b35f5fcc908a)

#### RBF 核函数的威力

RBF 核函数是不是只适合在低维空间线性不可分，需要映射到高维空间去进行分割的样本呢？

其实还真不是这样。要知道，对于我们用的 sklearn.svm.SVC 类而言，kernel 参数的默认值就是
rbf，也就是说，如果不是很明确核函数到底用什么，那就干脆都用 rbf。

我们将开始两个例子中的样本都用 `C=100，kernel=‘rbf’` 来尝试一下，结果分别如下：

![enter image description
here](http://images.gitbook.cn/ecdf6960-746a-11e8-9ce0-8f87da4d301a)

![enter image description
here](http://images.gitbook.cn/f607ecb0-746a-11e8-a58c-b35f5fcc908a)

可见，效果都不错。RBF 核函数，就是这么给力！

如果是 SVM 新手，对于我们不熟悉的数据，不妨直接就用上 RBF 核，再多调几次惩罚系数 C，说不定就能有个可以接受的结果。

#### 其他核函数

当然，核函数肯定不止线性核和 RBF 两种。就是这个 sklearn.svm.SVC 类，支持的 kernel 参数就有
linear、poly、rbf、sigmoid、precomputed 及自定义等多种。

那么其他集中核函数到底能在我们的数据上产生什么效果呢？具体的实践过程就留给同学们自己去做了，不过可以先给大家小小剧透一下：poly——多项式核对于上述三种数据的分类，都有点鸡肋。

### SVR 实例

最后看下这个实例：

    
    
        import numpy as np
        from sklearn.svm import SVR
        import matplotlib.pyplot as plt
    
        # 生成样本数据
        X = np.sort(5 * np.random.rand(40, 1), axis=0)
        y = np.ravel(2*X + 3)
    
        # 加入部分噪音
        y[::5] += 3 * (0.5 - np.random.rand(8))
    
        # 调用模型
        svr_rbf = SVR(kernel='rbf', C=1e3, gamma=0.1)
        svr_lin = SVR(kernel='linear', C=1e3)
        svr_poly = SVR(kernel='poly', C=1e3, degree=2)
        y_rbf = svr_rbf.fit(X, y).predict(X)
        y_lin = svr_lin.fit(X, y).predict(X)
        y_poly = svr_poly.fit(X, y).predict(X)
    
        # 可视化结果
        lw = 2
        plt.scatter(X, y, color='darkorange', label='data')
        plt.plot(X, y_rbf, color='navy', lw=lw, label='RBF model')
        plt.plot(X, y_lin, color='c', lw=lw, label='Linear model')
        plt.plot(X, y_poly, color='cornflowerblue', lw=lw, label='Polynomial model')
        plt.xlabel('data')
        plt.ylabel('target')
        plt.title('Support Vector Regression')
        plt.legend()
        plt.show()
    

结果为：

![enter image description
here](http://images.gitbook.cn/11124320-7f79-11e8-8f4f-398ed9d7e1c5)

将 `y = np.ravel(2*X + 3)` 替换为：

    
    
    y = np.polyval([2,3,5,2], X).ravel()
    

结果变为：

![enter image description
here](http://images.gitbook.cn/1ad70670-7f79-11e8-90d1-cbe88bc2e7cd)

同一条语句再替换为：

    
    
        y = np.sin(X).ravel()
    

结果则变为了：

![enter image description
here](http://images.gitbook.cn/2260f4f0-7f79-11e8-8f4f-398ed9d7e1c5)

