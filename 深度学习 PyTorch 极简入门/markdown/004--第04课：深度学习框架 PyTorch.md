随着深度学习的研究热潮持续高涨，各种开源深度学习框架也层出不穷，包括
TensorFlow、PyTorch、Caffe2、Keras、CNTK、MXNet、Paddle、DeepLearning4、Lasagne、Neon
等等。其中，谷歌推出的 TensorFlow
无疑在关注度和用户数上都占据绝对优势，最为流行。但是，今天我将给大家介绍的却是另外一个发展与流行势头强劲的深度学习框架：PyTorch。

### 4.1 为什么选择 PyTorch

首先，我们来看一张图：

![enter image description
here](http://images.gitbook.cn/9c01df70-7f80-11e8-83cb-75948241b5cb)

这张图来自斯坦福 Stanford CS231n（Spring 2017），我们可以看到如今几个主流的深度学习框架。其中，Caffe2 由 Facebook
推出，它的前身是 UC Berkeley 推出的 Caffe。PyTorch 也由 Facebook 推出，它的前身是 NYU 和 Facebook
一起推出的 Torch。TensorFlow 由 Google 推出，它的前身是 U Montreal 推出的 Theano。另外，还有百度推出的
Paddle，Microsoft 推出的 CNTK，Amazon 推出的 MXNet，等等。总体上，深度学习框架呈现出从学术研究到工业应用的发展趋势。

下面，主要介绍一下与 TensorFlow 相比，PyTorch 的优势有哪些。总的来说，PyTorch
更有利于研究人员、爱好者、小规模项目等快速搞出原型。而 TensorFlow 更适合大规模部署，特别是需要跨平台和嵌入式部署时。

#### 1\. 难易程度

PyTorch 实际上是 Numpy 的替代者，而且支持 GPU，带有高级功能，可以用来搭建和训练深度神经网络。如果你熟悉 Numpy、Python
以及常见的深度学习概念（卷积层、循环层、SGD 等），会非常容易上手 PyTorch。

而 TensorFlow 可以看成是一个嵌入 Python 的编程语言。你写的 TensorFlow 代码会被 Python 编译成一张图，然后由
TensorFlow 执行引擎运行。我见过好多新手因为这个增加的间接层而困扰，也正是因为同样的原因，TensorFlow
有一些额外的概念需要学习，例如会话、图、变量作用域（Variable Scoping）、占位符等。另外还需要更多的样板代码才能让一个基本的模型运行。所以
TensorFlow 的上手时间，肯定要比 PyTorch 长。

#### 2\. 图的创建和调试

创建和运行计算图可能是两个框架最不同的地方。在 PyTorch 中，图结构是动态的，这意味着图在运行时构建。而在 TensorFlow
中，图结构是静态的，这意味着图先被 “ 编译 ” 然后再运行。

举一个简单的例子，在 PyTorch 中你可以用标准的 Python 语法编写一个 for 循环结构：

    
    
    for _ in range(T):
        h = torch.matmul(W, h) + b
    

此处 T 可以在每次执行代码时改变。而 TensorFlow 中，这需要使用 “ 控制流操作 ” 来构建图，例如
tf.while_loop。TensorFlow 确实提供了 dynamic_rnn 用于常见结构，但是创建自定义动态计算真的更加困难。

PyTorch 中简单的图结构更容易理解，更重要的是，还更容易调试。调试 PyTorch 代码就像调试 Python 代码一样。你可以使用 pdb
并在任何地方设置断点。调试 TensorFlow 代码可不容易，要么得从会话请求要检查的变量，要么学会使用 TensorFlow 的调试器（tfdbg）。

![enter image description
here](http://images.gitbook.cn/214a8570-7f85-11e8-83cb-75948241b5cb)

总的来说，选择 PyTorch 的原因很简单，因为简单易懂。而且，它还弥补了 Tensorflow 静态构图的致命弱点。PyTorch
是可以构建动态计算图。也就是说你可以随时改变神经网络的结构，而不影响其计算过程。而 Tensorflow
这种静态图模块，一旦搭建好了神经网络，你想修改结构也不行。PyTorch 既可以修改结构，也可以使用简单易懂的搭建方法来创建神经网络。而且你也可以用 GPU
来进行加速运算, 性能一点也不比 Tensorflow 差。

### 4.2 PyTorch 历史

2017年1月，Facebook 人工智能研究院（FAIR）团队在 GitHub 上开源了 PyTorch，并迅速占领 GitHub 热度榜榜首。

![enter image description
here](http://images.gitbook.cn/c2f39a50-7f86-11e8-a0b2-3f34cb03c3bd)

PyTorch 的前身是 Torch，但是 Torch 是基于 Lua 语言。Lua 简洁高效，但由于其过于小众，用的人不是很多，以至于很多人听说要掌握
Torch 必须新学一门语言就望而却步（其实 Lua 是一门比 Python 还简单的语言）。考虑到 Python
在计算科学领域的领先地位，以及其生态完整性和接口易用性，几乎任何框架都不可避免地要提供 Python 接口。终于，在 2017 年，Torch
的幕后团队推出了 PyTorch。PyTorch 不是简单地封装 Lua Torch 提供 Python 接口，而是对 Tensor
之上的所有模块进行了重构，并新增了最先进的自动求导系统，成为当下最流行的动态图框架。

如今，已经有很多科研机构和大公司在使用 PyTroch，用一张图来表示。

![enter image description
here](http://images.gitbook.cn/e3c08e40-7f87-11e8-a0b2-3f34cb03c3bd)

### 4.3 PyTorch 的安装

PyTorch 的安装非常简单，目前已经支持 Windows 安装了。打开 PyTorch 的官方网站：

> <https://pytorch.org/>

然后，选择对应的操作系统 OS、包管理器 Package Manager、Python 版本、CUDA
版本，根据提示的命令，直接运行该命令即可完成安装。CUDA（Compute Unified Device Architecture）是显卡厂商 NVIDIA
推出的运算平台，用于 GPU 运算。目前的 PyTorch 版本已经更新到 0.4.0 了。

![enter image description
here](http://images.gitbook.cn/cd225b50-7f8c-11e8-83cb-75948241b5cb)

需要注意的是，由于防火墙限制，PyTorch 官网有时候会无法查看 “ Run this command ”
之后的指令。所以，建议使用其他安装方法，请读者自行搜索，这里就不占篇幅赘述了。

除了安装 PyTorch 之外，建议也安装 torchvision 包。torchvision 包是服务于 PyTorch
深度学习框架的，用来生成图片、视频数据集和一些流行的模型类和预训练模型。简单来说，torchvision 由
torchvision.datasets、torchvision.models、torchvision.transforms、torchvision.utils
四个模块组成。

### 4.4 PyTorch 基础知识

#### 1\. 张量 Tensor

张量（Tensor）是所有深度学习框架中最核心的组件，因为后续的所有运算和优化算法都是基于张量进行的。几何代数中定义的张量是基于向量和矩阵的推广，通俗一点理解的话，我们可以将标量视为零阶张量，矢量视为一阶张量，那么矩阵就是二阶张量。

PyTorch 中的张量（Tensor）类似 NumPy 中的 ndarrays，之所以称之为 Tensor 的另一个原因是它可以运行在 GPU
中，以加速运算。

构建一个 5x3 的随机初始化矩阵，类型 dtype 是长整型 float：

    
    
    import torch
    
    x = torch.randn(5, 3, dtype=torch.float)
    print(x)
    

输出如下：

    
    
    tensor([[-0.9508,  1.0049, -0.8923],
            [-0.2570,  0.3362,  0.3103],
            [-1.1511,  0.1061, -1.0930],
            [-0.9184,  0.4947, -0.8810],
            [-0.8143,  0.7078,  0.1530]])
    

直接把现成数据构建成 Tensor：

    
    
    x = torch.tensor([0, 1, 2, 3, 4])
    print(x)
    

输出如下：

    
    
    tensor([ 0,  1,  2,  3,  4])
    

Tensor 可以进行多种运算操作，以加法为例。

语法 1：提供一个输出 Tensor 作为参数：

    
    
    x = torch.randn(5, 3)
    y = torch.randn(5, 3)
    result = torch.empty(5, 3)
    torch.add(x, y, out=result)
    print(result)
    

输出如下：

    
    
    tensor([[ 0.2954, -0.7934, -1.5770],
            [-0.2942,  0.7494,  0.7648],
            [ 1.7711, -2.0219, -1.0717],
            [ 2.0904, -1.4883, -1.8856],
            [-0.0372,  0.0854,  2.9093]])
    

语法 2：替换

    
    
    # adds x to y
    y.add_(x)
    print(y)
    

输出如下：

    
    
    tensor([[ 0.2954, -0.7934, -1.5770],
            [-0.2942,  0.7494,  0.7648],
            [ 1.7711, -2.0219, -1.0717],
            [ 2.0904, -1.4883, -1.8856],
            [-0.0372,  0.0854,  2.9093]])
    

注意：任何操作符都固定地在前面加上 _ 来表示替换。例如：y.copy_(x)，y.t_()，都将改变 y。

另外，Tensor 还支持多种运算，包括转置、索引、切片、数学运算、线性代数、随机数，等等。

#### 2\. Tensor 与 NumPy

Tensor 可以与 NumPy 相互转化。

Tensor 转化为 NumPy：

    
    
    a = torch.ones(5)
    print(a)
    b = a.numpy()
    print(b)
    

输出如下：

    
    
    tensor([ 1.,  1.,  1.,  1.,  1.])
    [ 1.  1.  1.  1.  1.]
    

NumPy 转化为 Tensor：

    
    
    import numpy as np
    a = np.ones(5)
    print(a)
    b = torch.from_numpy(a)
    print(b)
    

输出如下：

    
    
    [ 1.  1.  1.  1.  1.]
    tensor([ 1.,  1.,  1.,  1.,  1.], dtype=torch.float64)
    

值得注意的是，Torch 中的 Tensor 和 NumPy 中的 Array 共享内存位置，一个改变，另一个也同样改变。

    
    
    a.add_(1)
    print(a)
    print(b)
    

输出如下：

    
    
    [ 2.  2.  2.  2.  2.]
    tensor([ 2.,  2.,  2.,  2.,  2.], dtype=torch.float64)
    

Tensor 可以被移动到任何设备中，例如 GPU 以加速运算，使用 `.to` 方法即可。

    
    
    x = torch.randn(1)
    if torch.cuda.is_available():
        device = torch.device("cuda")            
        y = torch.ones_like(x, device=device)    # directly create a tensor on GPU
        x = x.to(device)                         
        z = x + y
        print(z)
        print(z.to("cpu", torch.double))         
    

输出如下：

    
    
    tensor([ 2.0718], device='cuda:0')
    tensor([ 2.0718], dtype=torch.float64)
    

#### 3\. 自动求导 Autograd

Autograd 为所有 Tensor
上的操作提供了自动微分机制。这是一个动态运行的机制，也就是说你的代码运行的时候，反向传播过程就已经被定义了，且每迭代都会动态变化。如果把 Tensor
的属性 .requires_grad 设置为 True，就会追踪它的所有操作。完成计算后，可以通过 .backward() 来自动完成所有梯度计算。

首先，创建一个 Tensor，设置 requires_grad=True，跟踪整个计算过程。

    
    
    x = torch.ones(2, 2, requires_grad=True)
    print(x)
    

输出为：

    
    
    tensor([[ 1.,  1.],
            [ 1.,  1.]])
    

在 Tensor 上进行运算操作：

    
    
    y = x + 1
    z = y * y * 2
    out = z.mean()
    print(z, out)
    

输出为：

    
    
    tensor([[ 8.,  8.],
            [ 8.,  8.]]) tensor(8.)
    

接下来进行反向传播，因为 out 是标量，所以 out.backward() 等价于 out.backward(torch.tensor(1))。

    
    
    out.backward()
    print(x.grad)
    

输出为：

    
    
    tensor([[ 2.,  2.],
            [ 2.,  2.]])
    

我们可以对求导结果做个简单验证。因为：

$out=\frac14\sum_i z_i=\frac14\sum_i2(x_i+1)^2{dx_i}=x_i+1=1+1=2$

### 4.5 总结

本文主要介绍了深度学习框架 PyTorch。通过介绍目前最流行的几个深度学习框架，以及与 TensorFlow 的比较，阐述了选择 PyTorch
的原因：上手简单和动态图调试。然后重点介绍了 PyTorch 的基础语法知识，包括张量 Tensor 和自动求导机制 Autograd。

> **参考资料**
>
>   * 《PyTorch 还是 TensorFlow？这有一份新手深度学习框架选择指南》
>

>
> 访问链接：<https://zhuanlan.zhihu.com/p/28636490>
>
>   * 《等什么，赶快抱紧 PyTorch 的大腿！》
>

>
> 访问链接：<https://zhuanlan.zhihu.com/p/26670032>

