上一篇，我们主要介绍了 CNN 的基本概念和模型结构。本文将带领大家使用 PyTorch 一步步搭建 CNN 模型，进行数字图片识别。本案例中，我们选用的是
MNIST 数据集。

总的来说，我们构建分类器将按照以下步骤来做：

  * 使用 torchvision 加载 MNIST 数据集；
  * 定义一个卷积神经网络 CNN；
  * 定义损失函数；
  * 使用训练样本，训练网络；
  * 在测试样本上进行测试。

### MNIST 简介

[MNIST](http://yann.lecun.com/exdb/mnist/) 是深度学习领域中经典的手写图片数据集，这些图片采集自不同人手写的从 0
到 9 的数字，由 6 万张训练图片和 1 万张测试图片构成，每张图片都是 28*28 大小（单通道）。示例图片如下图所示：

![enter image description
here](https://images.gitbook.cn/b3fa7b80-b6a0-11e8-be77-6d9c9e98f294)

MNIST 数据集由以下四个部分组成：

  * 训练图片： `train-images-idx3-ubyte.gz`
  * 训练图片标签：`train-labels-idx1-ubyte.gz`
  * 测试图片：`t10k-images-idx3-ubyte.gz`
  * 测试图片标签：`t10k-labels-idx1-ubyte.gz`

MNIST 数据集采用 ubyte 格式存储，便于压缩和节省空间。

### 导入数据集

首先介绍一下 torchvision。torchvision 是一个专门进行图形处理的库，可加载比较常见的数据库，例如
Imagenet、CIFAR10、MNIST 等等。图片的数据转换采用 `torchvision.datasets` 和
`torch.utils.data.DataLoader`。torchvision 避免了重复写数据集加载代码，让数据集的加载更加简单。

一般情况下， torchvision 需独立安装，安装 PyTorch 之后再安装 torchvision 即可。

    
    
    import torch
    import torchvision
    import torchvision.transforms as transforms
    import torch.nn as nn
    import torch.nn.functional as F
    import torch.optim as optim
    import matplotlib.pyplot as plt
    import numpy as np
    
    transform = transforms.Compose(
        [transforms.ToTensor()])
    
    # 训练集
    trainset = torchvision.datasets.MNIST(root='./data',     # 选择数据的根目录
                                          train=True,
                                          download=False,    # 不从网络上download图片
                                          transform=transform)
    trainloader = torch.utils.data.DataLoader(trainset, batch_size=4,
                                             shuffle=True, num_workers=2)
    # 测试集
    testset = torchvision.datasets.MNIST(root='./data',     # 选择数据的根目录
                                         train=False,
                                         download=False,    # 不从网络上download图片
                                         transform=transform)
    testloader = torch.utils.data.DataLoader(testset, batch_size=4,
                                            shuffle=False, num_workers=2)
    

上述代码用来导入 MNIST 数据集。我们可以设置 `download=True`，即在线下载数据集。我已经提前下载完成，所以这里的 download
设置为 False，将从本地导入数据集。设置 `batch_size=4`，`shuffle=True` 表示每次 epoch
都重新打乱训练样本，`num_workers=2` 表示使用两个子进程加载数据。

下面程序展示了 Mini-batch 训练样本图片并标注正确标签的过程。

    
    
    def imshow(img):
        npimg = img.numpy()
        plt.imshow(np.transpose(npimg, (1, 2, 0)))
    
    # 选择一个 batch 的图片
    dataiter = iter(trainloader)
    images, labels = dataiter.next()
    
    # 显示图片
    imshow(torchvision.utils.make_grid(images))
    plt.show()
    # 打印 labels
    print(' '.join('%11s' % labels[j].numpy() for j in range(4)))
    

![enter image description
here](https://images.gitbook.cn/849bece0-b581-11e8-b571-d9b5354f1ca8)

### 定义卷积神经网络

我们选择使用 `LeNet-5` 网络，其网络结构如下所示：

![enter image description
here](https://images.gitbook.cn/018db440-b582-11e8-b620-f5e17eb8ec57)

典型的 LeNet-5 结构包含卷积层、池化层、全连接层，顺序一般是 CONV Layer -> POOL Layer -> CONV Layer ->
POOL Layer -> FC Layer -> FC Layer -> Output Layer。

    
    
    class Net(nn.Module):
        def __init__(self):
            super(Net, self).__init__()
            self.conv1 = nn.Conv2d(1, 6, 5)        # 1个输入图片通道，6个输出通道，5x5 卷积核
            self.pool = nn.MaxPool2d(2, 2)         # max pooling，2x2
            self.conv2 = nn.Conv2d(6, 16, 5)       # 6个输入图片通道，16个输出通道，5x5 卷积核
            self.fc1 = nn.Linear(16 * 4 * 4, 120)  # 拉伸成一维向量，全连接层
            self.fc2 = nn.Linear(120, 84)          # 全连接层 
            self.fc3 = nn.Linear(84, 10)           # 全连接层，输出层 softmax，10个数字
    
        def forward(self, x):
            x = self.pool(F.relu(self.conv1(x)))
            x = self.pool(F.relu(self.conv2(x)))
            x = x.view(-1, 16 * 4 * 4)    # 拉伸成一维向量
            x = F.relu(self.fc1(x))
            x = F.relu(self.fc2(x))
            x = self.fc3(x)
            return x
    

以上代码是构建 CNN 的核心部分。我们发现 PyTroch 构建卷积神经网络模型的过程非常简单，只需要简单的几行语句。在类 Net
的初始化函数中，直接搭建卷积层、池化层和全连接层。其中，`nn.Conv2d(1, 6, 5)` 里的 1 代表输入图片的维度，因为是灰度图片，所以维度为
1；6 表示第一层滤波器组的个数，即第一层的输出维度；5 表示滤波器的尺寸为 `5*5`。`nn.MaxPool2d(2, 2)` 表示池化层采用 Max
Pooling，尺寸为 `2*2`。`nn.Linear(16 * 4 * 4, 120)` 表示全连接层。下面解释一下 `16*4*4` 是怎样得来的。

MNIST 图片大小为 `28*28`，经过第一层卷积层和池化层后，尺寸为：

$$\frac{28-5}{1}+1=24$$

$$\frac{24}{2}=12$$

经过第二层卷积层和池化层后，尺寸为：

$$\frac{12-5}{1}+1=8$$

$$\frac{8}{2}=4$$

由于该层滤波器组个数为 16，则拉伸一维数组的维度为 `16*4*4`。

函数 `forward(self, x)` 定义了 CNN 的正向传播过程。

接下来我们可以建立一个 Net 对象，并打印出来，看看其网络结构。

    
    
    net = Net()
    print(net)  
    

打印出的网络结构如下：

    
    
    Net(
    
      (conv1): Conv2d(1, 6, kernel_size=(5, 5), stride=(1, 1))
    
      (pool): MaxPool2d(kernel_size=2, stride=2, padding=0, dilation=1, ceil_mode=False)
    
      (conv2): Conv2d(6, 16, kernel_size=(5, 5), stride=(1, 1))
    
      (fc1): Linear(in_features=400, out_features=120, bias=True)
    
      (fc2): Linear(in_features=120, out_features=84, bias=True)
    
      (fc3): Linear(in_features=84, out_features=10, bias=True)
    
    )
    

非常直观，可以完整清晰地查看我们构建的网络模型结构。

### 定义损失函数

该项目是一个分类问题，所以损失函数使用交叉熵，PyTorch 中用 CrossEntropyLoss 表示交叉熵。如果是回归问题，损失函数一般使用均方差
MSE ，即 `nn.MSELoss`。

要构建一个优化器 Optimizer，必须给它一个可进行迭代优化的、包含了所有参数的列表，下面代码中 `net.parameters()`
表示优化的参数。然后，可以指定程序优化的选项，例如学习速率，本例中设置学习率 lr=0.0001。

    
    
    criterion = nn.CrossEntropyLoss()
    optimizer = optim.Adam(net.parameters(), lr=0.0001)
    

本项目使用了 Adam 梯度优化算法。关于 Adam，之前的课程中已做过详细介绍。除了 Adam，还可以使用 SGD、Momentum 等其它梯度优化算法。

### 训练网络

接下来就是最有趣的地方了。我们只需循环遍历数据迭代器，放入网络的输入层并优化即可。

    
    
    num_epoches = 5    # 设置 epoch 数目
    cost = []     # 损失函数累加
    
    for epoch in range(num_epoches):    
    
        running_loss = 0.0
        for i, data in enumerate(trainloader, 0):
            # 输入样本和标签
            inputs, labels = data
    
            # 每次训练梯度清零
            optimizer.zero_grad()
    
            # 正向传播、反向传播和优化过程
            outputs = net(inputs)
            loss = criterion(outputs, labels)
            loss.backward()
            optimizer.step()
    
            # 打印训练情况
            running_loss += loss.item()
            if i % 2000 == 1999:    # 每隔2000 mini-batches，打印一次
                print('[%d, %5d] loss: %.3f' % 
                     (epoch + 1, i + 1, running_loss / 2000))
                cost.append(running_loss / 2000)
                running_loss = 0.0
    

上述代码中需要注意的是，每次迭代训练时都要先把所有梯度清零，即执行
`optimizer.zero_grad()`。否则，梯度会累加，造成训练错误和失效。PyTorch 中的 `.backward()`
可自动完成所有梯度计算。

打印的结果如下：

    
    
    [1,  2000] loss: 1.038
    [1,  4000] loss: 0.364
    [1,  6000] loss: 0.261
    [1,  8000] loss: 0.225
    [1, 10000] loss: 0.182
    [1, 12000] loss: 0.170
    [1, 14000] loss: 0.146
    [2,  2000] loss: 0.122
    [2,  4000] loss: 0.118
    [2,  6000] loss: 0.102
    [2,  8000] loss: 0.108
    [2, 10000] loss: 0.103
    [2, 12000] loss: 0.092
    [2, 14000] loss: 0.085
    [3,  2000] loss: 0.089
    [3,  4000] loss: 0.082
    [3,  6000] loss: 0.078
    [3,  8000] loss: 0.068
    [3, 10000] loss: 0.059
    [3, 12000] loss: 0.064
    [3, 14000] loss: 0.067
    [4,  2000] loss: 0.058
    [4,  4000] loss: 0.063
    [4,  6000] loss: 0.055
    [4,  8000] loss: 0.059
    [4, 10000] loss: 0.057
    [4, 12000] loss: 0.055
    [4, 14000] loss: 0.052
    [5,  2000] loss: 0.044
    [5,  4000] loss: 0.046
    [5,  6000] loss: 0.053
    [5,  8000] loss: 0.044
    [5, 10000] loss: 0.048
    [5, 12000] loss: 0.046
    [5, 14000] loss: 0.049
    

将所有 Loss 趋势绘制成图，代码及图片如下所示：

    
    
    plt.plot(cost)
    plt.ylabel('cost')
    plt.show()
    

![enter image description
here](https://images.gitbook.cn/9faf7620-b58e-11e8-b3b2-8f3a5efca007)

显然，随着迭代训练，Loss 逐渐减小。

### 测试数据

让我们来看一下网络模型在整个测试数据集上的训练效果。

    
    
    correct = 0
    total = 0
    with torch.no_grad():
        for data in testloader:
            images, labels = data
            outputs = net(images)
            _, predicted = torch.max(outputs.data, 1)
            total += labels.size(0)
            correct += (predicted == labels).sum().item()
    
    print('Accuracy of the network on the 10000 test images: %.3f %%' % 
         (100 * correct / total))
    

执行结果如下：

> Accuracy of the network on the 10000 test images: 98.900 %

结果显示模型在测试集上的准确率达到了 98.900 %。说明我们训练的卷积神经网络性能还是不错的。

包含本文完整代码的 `.ipynb` 文件，我已经放在了 GitHub 上，大家可自行下载：

> [第16课：项目实战：利用 PyTorch 构建 CNN
> 模型](https://github.com/RedstoneWill/gitchat_dl/tree/master/16%20chapter)

