不管数据分析、数据挖掘、机器学习等，都会经常遇到数组的 shape 操作。有些读者很疑惑，为什么一个一维数组，能 shape 成任意维度的高维数组。

NumPy 也提供高效的数据 shape 操作，那么它是如何实现这种操作，用到什么数据结构，原理又是什么？

弄明白这个问题后，再去使用 NumPy、TensorFlow，就会瞬间清晰很多。

今天我们先来弄明白这个操作的原理，然后学习 NumPy 中常用的与数据操作相关的函数。

### 揭秘 Shape

一个一维数组，长度为 12，为什么能变化为二维 (12,1) 或 (2,6) 等、三维 (12,1,1) 或 (2,3,2) 等、四维 (12,1,1,1)
或 (2,3,1,2) 等。总之，能变化为任意多维度。

reshape 是如何做到的？使用了什么魔法数据结构和算法吗？

**这篇文章对于 reshape 方法的原理解释，会很独到，尽可能让朋友们弄明白数组 reshape 的魔法。**

如同往常一样，导入 NumPy 包：

    
    
    import numpy as np
    

创建一个一维数组 a，从 0 开始，间隔为 2，含有 12 个元素的数组：

    
    
    a = np.arange(0,24,2)
    

打印数组 a：

    
    
    In [48]: a
    Out[48]: array([ 0,  2,  4,  6,  8, 10, 12, 14, 16, 18, 20, 22])
    

如上数组 a，NumPy 会将其解读成两个结构，一个 buffer，还有一个 view。

buffer 的示意图如下所示：

![](https://images.gitbook.cn/e068dcd0-66d4-11ea-be7b-8da9b9303186)

view 是解释 buffer 的一个结构，比如数据类型，flags 信息等：

    
    
    In [50]: a.dtype
    Out[50]: dtype('int32')
    
    In [51]: a.flags
    Out[51]:
      C_CONTIGUOUS : True
      F_CONTIGUOUS : True
      OWNDATA : True
      WRITEABLE : True
      ALIGNED : True
      WRITEBACKIFCOPY : False
      UPDATEIFCOPY : False
    

使用 a[6] 访问数组 a 中 index 为 6 的元素。从背后实现看，NumPy 会辅助一个轴，轴的取值为 0 到 11。

从概念上看，它的示意图如下所示：

![](https://images.gitbook.cn/826e8610-66d5-11ea-8989-43da5eeeebb3)

所以，借助这个轴 i，a[6] 就会被索引到元素 12，如下所示：

![](https://images.gitbook.cn/98e4a550-66d5-11ea-a50b-d5d577e93b41)

至此，大家要建立一个轴的概念。

接下来，做一次 reshape 变化，变化数组 a 的 shape 为 (2,6)：

    
    
    b = a.reshape(2,6)
    

打印 b：

    
    
    In [53]: b
    Out[53]:
    array([[ 0,  2,  4,  6,  8, 10],
           [12, 14, 16, 18, 20, 22]])
    

此时，NumPy 会建立两个轴，假设为 i、j，i 的取值为 0 到 1，j 的取值为 0 到 5，示意图如下：

![](https://images.gitbook.cn/b8b665d0-66d5-11ea-a1ac-77a357f47394)

使用 b[1][2] 获取元素到 16：

    
    
    In [54]: b[1][2]
    Out[54]: 16
    

两个轴的取值分为 1、2，如下图所示，定位到元素 16：

![](https://images.gitbook.cn/d293f300-66d5-11ea-9366-31e8fd379c79)

平时，有些读者朋友可能会混淆两个 shape，(12,) 和 (12,1)，其实前者一个轴，后者两个轴，示意图分别如下。

一个轴，取值从 0 到 11：

![](https://images.gitbook.cn/a5426d90-66d6-11ea-970b-33f4f9e40da2)

两个轴，i 轴取值从 0 到 11，j 轴取值从 0 到 0：

![](https://images.gitbook.cn/becb9930-66d6-11ea-be7b-8da9b9303186)

至此，大家要建立两个轴的概念。

**并且，通过上面几幅图看到，无论 shape 如何变化，变化的是视图，底下的 buffer 始终未变。**

接下来，上升到三个轴，变化数组 a 的 shape 为 (2,3,2)：

    
    
    c = a.reshape(2,3,2)
    

打印 c：

    
    
    In [55]: c = a.reshape(2,3,2)
    
    In [56]: c
    Out[56]:
    array([[[ 0,  2],
            [ 4,  6],
            [ 8, 10]],
    
           [[12, 14],
            [16, 18],
            [20, 22]]])
    

数组 c 有三个轴，取值分别为 0 到 1，0 到 2，0 到 1，示意图如下所示：

![](https://images.gitbook.cn/dd4543c0-66d6-11ea-9534-7f7d1862fd8c)

读者们注意体会，i、j、k 三个轴，其值的分布规律。如果去掉 i 轴取值为 1 的单元格后，

![](https://images.gitbook.cn/f421c5f0-66d6-11ea-9534-7f7d1862fd8c)

实际就对应到数组 c 的前半部分元素：

    
    
    array([[[ 0,  2],
            [ 4,  6],
            [ 8, 10]],
    

也就是如下的索引组合：

![](https://images.gitbook.cn/04ed3e00-66d7-11ea-8989-43da5eeeebb3)

至此，三个轴的 reshape 已经讲完，再说一个有意思的问题。

还记得，原始的一维数组 a 吗？它一共有 12 个元素，后来，我们变化它为数组 c，shape 为 (2,3,2)，那么如何升级为 4 维或任意维呢？

4 维可以为：(1,2,3,2)，示意图如下：

![](https://images.gitbook.cn/1723ded0-66d7-11ea-9416-9b7091176d0d)

看到，轴 i 索引取值只有 0，它被称为自由维度，可以任意插入到原数组的任意轴间。

比如，5 维可以为：(1,2,1,3,2)：

![](https://images.gitbook.cn/25bc2330-66d7-11ea-8874-af361683e17a)

至此，你应该完全理解 reshape 操作后的魔法：

  * buffer 是个一维数组，永远不变；
  * 变化的 shape 通过 view 传达；
  * 取值仅有 0 的轴为自由轴，它能变化出任意维度。

关于 reshape 操作，最后再说一点，reshape 后的数组，仅仅是原来数组的视图 view，并没有发生复制元素的行为，这样才能保证 reshape
操作更为高效。

    
    
    In [58]: v1 = np.arange(10)
    
    In [59]: v2 = v1.reshape(2,5)
    
    In [60]: v2
    Out[60]:
    array([[0, 1, 2, 3, 4],
           [5, 6, 7, 8, 9]])
    

改变 v2 的第一个元素：

    
    
    In [61]: v2[0,0] = 10
    
    In [62]: v2
    Out[62]:
    array([[10,  1,  2,  3,  4],
           [ 5,  6,  7,  8,  9]])
    

如果 v2 是 v1 的视图，那么 v1 也会改变，如下，v1 的第一个元素也发生相应改变，所以得证。

    
    
    In [63]: v1
    Out[63]: array([10,  1,  2,  3,  4,  5,  6,  7,  8,  9])
    

在了解完 reshape 操作的奥秘后，相信大家都建立轴和多轴的概念，这对灵活使用高维数组很有帮助。

下面分析一些 NumPy 中高频使用的方法。

### 元素级操作

NumPy 中两个数组加减乘除等，默认都是对应元素的操作：

    
    
    In [59]: v1 = np.arange(5)
    
    In [60]: v1
    Out[60]: array([0, 1, 2, 3, 4])
    

执行 `v1+2` 操作，按照元素顺序逐个加 2：

    
    
    In [57]: v1+2
    Out[57]: array([2, 3, 4, 5, 6])
    

执行 `v1 * v1`，注意是按照元素逐个相乘：

    
    
    In [58]: v1 * v1
    Out[58]: array([ 0,  1,  4,  9, 16])
    

### 矩阵运算

线性代数中，矩阵的乘法操作在 NumPy 中怎么实现？

常见两种方法：使用 dot 函数，另一种是转化为 matrix 对象。

dot 操作：

    
    
    # 数值 [1,10) 内，生成 shape 为 (5,2) 的随机整数数组
    In [1]: import numpy as np
    
    In [2]: v1 = np.arange(5)
    
    In [3]: v2 = np.random.randint(1,10,(5,2))
    
    In [4]: np.dot(v1,v2)
    Out[4]: array([49, 51])
    

另一种方法，将 v1 和 v2 分别转化为 matrix 对象：

    
    
    In [6]: np.matrix(v1)*np.matrix(v2)
    Out[6]: matrix([[49, 51]])
    

需要注意，数组 v1 经过 matrix 转化后，shape 由原来 (5,) 变化为 (1,5)，的确变得更像线性代数中的矩阵：

    
    
    In [83]: np.matrix(v1).shape
    Out[83]: (1, 5)
    

首先，导入与求行列式相关的模块 linalg，求矩阵的行列式，要求数组的最后两个维度相等。

    
    
    In [10]: from numpy import linalg 
    
    In [11]: v1 = np.arange(12)
    In [12]: v2 = v1.reshape(3,2,2)
    
    In [13]: linalg.det(v2)
    Out[13]: array([-2., -2., -2.])
    
    In [14]: v3 = np.arange(9).reshape(3,3)
    In [15]: linalg.det(v3)
    Out[15]: 0.0
    

### 统计变量

NumPy 能方便地求出统计学常见的描述性统计量。

#### **求平均值**

    
    
    In [21]: m1 = np.random.randint(1,10,(3,4))
    
    In [22]: m1
    Out[22]:
    array([[8, 4, 8, 2],
           [4, 9, 2, 1],
           [7, 7, 7, 5]])
    
    In [23]: m1.mean() # 默认求出数组所有元素的平均值
    Out[23]: 5.333333333333333
    
    In [24]: m1.sum() / 12
    Out[24]: 5.333333333333333
    

若想求某一维度的平均值，设置 axis 参数，求 axis 等于 1 的平均值：

    
    
    In [26]: m1.mean(axis = 1)
    Out[26]: array([5.5, 4. , 6.5])
    

#### **求标准差**

如下，分别求所有元素的标准差、某一维度上的标准差：

    
    
    In [28]: m1.std()
    Out[28]: 2.592724864350674
    
    In [29]: m1.std(axis=1)
    Out[29]: array([2.59807621, 3.082207  , 0.8660254 ])
    

#### **求方差**

如下，分别求所有元素的方差、某一维度上的方差：

    
    
    In [30]: m1.var()
    Out[30]: 6.722222222222221
    
    In [31]: m1.var(axis=1)
    Out[31]: array([6.75, 9.5 , 0.75])
    

#### **求最大值**

如下，分别求所有元素的最大值、某一维度上的最大值：

    
    
    In [34]: m1.max()
    Out[34]: 9
    
    In [35]: m1.max(axis=1)
    Out[35]: array([8, 9, 7])
    

#### **求最小值**

如下，分别求所有元素的最小值、某一维度上的最小值：

    
    
    In [36]: m1.min()
    Out[36]: 1
    
    In [37]: m1.min(axis=1)
    Out[37]: array([2, 1, 5])
    

#### **求和**

如下，分别求所有维度上元素的和、某一维度上的元素和：

    
    
    In [38]: m1.sum()
    Out[38]: 64
    
    In [39]: m1.sum(axis=1)
    Out[39]: array([22, 16, 26])
    

#### **求累乘**

如下，分别求所有维度上元素的累乘、某一维度上的累乘：

    
    
    In [22]: m1
    Out[22]:
    array([[8, 4, 8, 2],
           [4, 9, 2, 1],
           [7, 7, 7, 5]])
    
    In [40]: m1.cumprod()
    Out[40]:
    array([       8,       32,      256,      512,     2048,    18432,
              36864,    36864,   258048,  1806336, 12644352, 63221760],
          dtype=int32)
    
    In [42]: m1.cumprod(axis=1)
    Out[42]:
    array([[   8,   32,  256,  512],
           [   4,   36,   72,   72],
           [   7,   49,  343, 1715]], dtype=int32)
    

#### **求累和**

如下，分别求所有维度上元素的累加和、某一维度上的累加和：

    
    
    In [43]: m1.cumsum()
    Out[43]: array([ 8, 12, 20, 22, 26, 35, 37, 38, 45, 52, 59, 64], dtype=int32)
    
    In [44]: m1.cumsum(axis=1)
    Out[44]:
    array([[ 8, 12, 20, 22],
           [ 4, 13, 15, 16],
           [ 7, 14, 21, 26]], dtype=int32)
    

#### **求迹**

对角线上元素的和：

    
    
    In [22]: m1
    Out[22]:
    array([[8, 4, 8, 2],
           [4, 9, 2, 1],
           [7, 7, 7, 5]])
    
    In [45]: m1.trace()
    Out[45]: 24
    

### 改变数组

#### **flatten 函数**

NumPy 的 flatten 函数也有改变 shape 的能力，它将高维数组变为向量。但是，它会发生数组复制行为。

    
    
    In [68]: v1 = np.random.randint(1,10,(2,3))
    
    In [69]: v1
    Out[69]:
    array([[3, 8, 5],
           [2, 3, 4]])
    
    In [70]: v2 = v1.flatten()
    
    In [71]: v2
    Out[71]: array([3, 8, 5, 2, 3, 4])
    

v2[0] 被修改为 30 后，原数组 v1 没有任何改变。

    
    
    In [73]: v2[0] = 30
    
    In [74]: v1
    Out[74]:
    array([[3, 8, 5],
           [2, 3, 4]])
    

#### **newaxis**

使用 newaxis 增加一个维度，维度的索引只有 0，本篇的开头已经详细解释过，不再赘述。

    
    
    In [81]: v1 = np.arange(10) # shape 为一维，(10, )
    In [82]: v2 = v1[:,np.newaxis] # shape 为二维，(10,1)
    

#### **repeat**

repeat 操作，实现某一维上的元素复制操作。

在维度 0 上复制元素 2 次：

    
    
    In [132]: a = np.array([[1,2],[3,4]])
    
    In [138]: np.repeat(a,2,axis=0)
    Out[138]:
    array([[1, 2],
           [1, 2],
           [3, 4],
           [3, 4]])
    

在维度 1 上复制元素 2 次：

    
    
    In [137]: np.repeat(a,2,axis=1)
    Out[137]:
    array([[1, 1, 2, 2],
           [3, 3, 4, 4]])
    

#### **tile**

tile 实现按块复制元素：

    
    
    In [4]: a = np.array([[1,2],[3,4]])
    
    In [89]: np.tile(a,3)
    Out[89]:
    array([[1, 2, 1, 2, 1, 2],
           [3, 4, 3, 4, 3, 4]])
    
    In [6]: np.tile(a,(2,3))
    Out[6]:
    array([[1, 2, 1, 2, 1, 2],
           [3, 4, 3, 4, 3, 4],
           [1, 2, 1, 2, 1, 2],
           [3, 4, 3, 4, 3, 4]])
    

#### **vstack**

vstack，vertical stack，沿竖直方向合并多个数组：

    
    
    In [4]: a = np.array([[1,2],[3,4]])
    In [5]: b = np.array([[-1,-2]])
    In [6]: c = np.vstack((a,b)) # 注意参数类型：元组
    Out[5]:
    array([[ 1,  2],
           [ 3,  4],
           [-1, -2]])
    

#### **hstack**

hstack 沿水平方向合并多个数组。

值得注意，不管是 vstack，还是 hstack，沿着合并方向的维度，其元素的长度要一致。

    
    
    In [4]: a = np.array([[1,2],[3,4]])
    In [5]: b = np.array([[5,6,7],[8,9,10]])
    In [4]: c = np.hstack((a,b)) 
    In [5]: c
    Out[5]:
    array([[ 1,  2,  5,  6,  7],
           [ 3,  4,  8,  9, 10]])
    

#### **concatenate**

concatenate 指定在哪个维度上合作数组。

    
    
    In [4]: a = np.array([[1,2],[3,4]])
    In [5]: b = np.array([[-1,-2]])
    In [6]: np.concatenate((a,b),axis=0) # 效果等于 vstack
    Out[6]:
    array([[ 1,  2],
           [ 3,  4],
           [-1, -2]])
    
    In [7]:  c = np.array([[5,6,7],[8,9,10]])
    In [108]: np.concatenate((a,c),axis=1) # 效果等于 hstack
    Out[108]:
    array([[ 1,  2,  5,  6,  7],
           [ 3,  4,  8,  9, 10]])
    

NumPy 还有一些小 track，比如 r_ 类、c_ 类，也能实现合并操作。

    
    
    In [4]: a = np.array([[1,2],[3,4]])
    In [5]: b = np.array([[-1,-2]])    
    In [6]: np.r_[a,b] # r_类
    Out[6]:
    array([[ 1,  2],
           [ 3,  4],
           [-1, -2]])    
    
    
    
    In [7]: a = np.array([[1,2],[3,4]])
    In [8]: c = np.array([[5,6,7],[8,9,10]])
    In [9]: np.c_[a,c] # c_ 类
    Out[9]:
    array([[ 1,  2,  5,  6,  7],
           [ 3,  4,  8,  9, 10]])    
    
    
    
    In [10]: np.r_[a,c[:,:2]]
    Out[10]:
    array([[1, 2],
           [3, 4],
           [5, 6],
           [8, 9]])
    

#### **argmax、argmin**

argmax 返回数组中某个维度的最大值索引，当未指明维度时，返回 buffer 中最大值索引。如下所示：

    
    
    In [131]: a = np.random.randint(1,10,(2,3))
    
    In [132]: a
    Out[132]:
    array([[8, 1, 4],
           [5, 4, 3]])
    
    In [133]: a.argmax()
    Out[133]: 0
    
    In [134]: a.argmax(axis = 0)
    Out[134]: array([0, 1, 0], dtype=int64)
    
    In [135]: a.argmax(axis = 1)
    Out[135]: array([0, 0], dtype=int64)
    

### 小结

今天总结了 NumPy 的 reshape 操作原理，从而建立起多维度、buffer、view 等概念；讨论了元素级操作、矩阵运算 dot、matrix；8
个统计学常用的统计量求法。

以及改变数组元素、融合数组的方法：repeat、tile、vstack、hstack、concatenate、r_、c_、argmax、argmin。

