今天我们来学习一下 Keras。

**Keras** 是一个用 Python 编写的高级神经网络 API，能够用 TensorFlow 或者 Theano 作为后端运行。是由
Francois Chollet 在 Google 工作时开发的，用于构建深度学习模型。

Keras 是为了实现快速实验和快速原型设计的，让开发者能够用最短的时间把想法转换为实验结果。用 Keras 可以非常 **快速地上手深度学习**
，搭建各种神经网络，现在在工业界和学术界都被广泛地采用。

我们这门课程的主体是学习用 TensorFlow 来解决问题，但是在一些比较复杂问题的实战应用上，我们会采用 Keras 来实现，这样可以让项目快速落地。

**本文结构：**

  1. 用 Keras 解决深度学习问题的一般流程
  2. 用 Keras 识别 MNIST 数据集
  3. 模型的改进
  4. Keras 中常用模块简介

* * *

> ### 1\. 用 Keras 解决深度学习问题的一般流程

关于安装，我们可以从官网找到各种方法，这里就不赘述：https://keras.io/zh/#_2

接下来我们用一个例子来看如何用 Keras 搭建神经网络并解决问题。

**首先，用 Keras 解决深度学习问题的一般流程可以分为 8 个步骤，如下图所示：**

![wYc6jv](https://images.gitbook.cn/wYc6jv.jpg)

* * *

> ### 2\. 用 Keras 识别 MNIST 数据集

我们通过一个例子，即用 Keras 识别 MNIST 数据集，来看这几步是如何实现的：

**首先导入所需要的包：**

    
    
    from __future__ import print_function
    import numpy as np
    from keras.datasets import mnist
    from keras.models import Sequential
    from keras.layers.core import Dense, Activation
    from keras.optimizers import SGD
    from keras.utils import np_utils
    
    np.random.seed(1671) 
    

**设置网络的超参数：**

    
    
    n_epoch = 20        # 网络最小化损失函数所需的迭代次数
    batch_size = 128        # 模型每批处理的图像数
    verbose = 1
    n_classes = 10       # 分类结果的类别个数
    optimizer = SGD() 
    n_hidden = 128
    validation_split=0.2     # 训练集的 20% 作为验证集
    

* * *

### 1\. 分析数据集

在 Keras 里有一些常见的[公开数据集](https://keras.io/zh/datasets/)：图像分类，电影评论，MNIST，Fashion-
MNIST，Boston 房价等。

这里我们将以 MNIST：Mixed National Institute of Standards and Technology 为例，

这个数据集是由 Yaan LeCun，Corinna Cortes，Christopher Burges 等创建的。训练样本 60,000 个，测试样本
10,000 个，每个样本为一个手写数字的图片，已经标准化为 28*28 像素。

**加载数据集：**

    
    
    # 获得训练集和测试集
    (X_train, y_train), (X_test, y_test) = mnist.load_data()
    
    print(X_train.shape[0], 'train samples')
    print(X_test.shape[0], 'test samples')
    
    # 结果：
    # 60000 train samples
    # 10000 test samples
    

* * *

### 2\. 数据预处理

这一步我们主要做这样四件事：

**Reshaping：** 将训练集和测试集的每个图片 X 的形状从 28x28 变为 784。 **Data type：** 数据类型要变成
float32。 **Normalize：** 标准化到 [0, 1] 之间

    
    
    RESHAPED = 784
    X_train = X_train.reshape(60000, RESHAPED)
    X_test = X_test.reshape(10000, RESHAPED)
    
    X_train = X_train.astype('float32')        
    X_test = X_test.astype('float32')
    
    X_train /= 255
    X_test /= 255
    

**One Hot Encoding：** 我们要预测的目标有 0～9 十个类别，是个多分类问题，要用 One Hot
方式将原来的标签数字转换为二进制向量，就是变成长度为 10 的向量，数字是几，就在第几个位置上面为 1， 其他位置为 0.

    
    
    Y_train = np_utils.to_categorical(y_train, n_classes)
    Y_test = np_utils.to_categorical(y_test, n_classes)
    

* * *

### 3\. 建立模型

在 Keras 中有两类主要的模型：[Sequential
顺序模型](https://keras.io/zh/models/sequential/)和使用函数式 API 的 Model 类模型。

这里我们 **用 Sequential 来建立神经网络结构** ，相当于是把多个网络层进行线性堆叠。 我们可以将各个层的列表传递给
Sequential，不过更常用的是通过 `.add()` 方法将各个网络层添加到模型中，直观又简单。

用 Sequential 建立成 model 后，就可以对模型进行编译训练，评价和预测：

compile：配置训练模型 fit：迭代训练模型 evaluate：测试模型，返回误差和评估标准的值
predict：训练好的模型，输入样本后可以生成预测的数据

    
    
    model = Sequential()
    
    model.add(Dense(n_hidden, input_shape=(RESHAPED,)))
    model.add(Activation('relu'))
    model.add(Dense(n_hidden))
    model.add(Activation('relu'))
    
    model.add(Dense(n_classes))
    model.add(Activation('softmax'))
    model.summary()
    

在这里我们建立了有两个隐藏层的网络，

因为 MNIST 每张图片的像素维度是 28 x 28 = 784，所以输入层的维度也就有 784 个神经元，对应着图片的每个像素。 每个隐藏层有
`n_hidden` = 128 个神经元。 后面有一个全连接层有 10 个神经元，用来计算 10 个类别的概率。

隐藏层的激活函数用 ReLU，输出层的激活函数用 Softmax，用来将任意实数值的k维向量压缩成[0, 1]区间的实数值k维向量，即计算概率。

在 Keras 中可以直接调用很多流行的[激活函数](https://keras.io/zh/activations/).

* * *

### 4\. 编译模型：

在训练模型之前，需要通过 compile 配置模型，这里需要定义三个参数：

**损失函数 loss：** 训练过程中优化器的目标就是使损失达到最小。损失函数也有很多种：https://keras.io/zh/losses/，常见的有
categorical_crossentropy 和 mse。

**优化器 optimizer：** 在训练模型时用于更新权重的算法。Keras
中的优化器列表：https://keras.io/zh/optimizers/，关于优化器的选择可以看这篇文章：

**评估标准 metrics：**
用于对训练好的模型进行评价。评估函数列表：https://keras.io/zh/metrics/，我们还可以自定义评价函数。

    
    
    model.compile(loss='categorical_crossentropy',
                  optimizer=optimizer,
                  metrics=['accuracy'])
    

* * *

### 5\. 训练模型：

之后用 fit 训练模型，`batch_size` 是每一批训练数据的数量，`epochs` 模型训练的代数，`verbose` 用于调试，

    
    
    history = model.fit(X_train, Y_train,
                        batch_size=batch_size, epochs=n_epoch,
                        verbose=verbose, validation_split=validation_split)
    

* * *

### 6\. 评价模型：

训练模型后，用测试集评估模型：

    
    
    score = model.evaluate(X_test, Y_test, verbose=verbose)
    
    print("\nTest score:", score[0])
    print('Test accuracy:', score[1])
    
    # 结果：
    # Test score: 0.18604595039635896
    # Test accuracy: 0.9462
    

还可以用 matplotlib 对结果进行可视化，来观察随着 epoch 的增加，模型在 train 和 test
数据集上面的准确率和损失的变化，也可以看出来当 epoch 为什么数值时就不用再训练了。

    
    
    import matplotlib.pyplot as plt
    
    # 列出 history 的数据
    print(history.history.keys())
    
    # 总结 history 计算 accuracy
    plt.plot(history.history['acc'])
    plt.plot(history.history['val_acc'])
    plt.title('model accuracy')
    plt.ylabel('accuracy')
    plt.xlabel('epoch')
    plt.legend(['train', 'test'], loc='upper left')
    plt.show()
    
    # 总结 history 计算 loss
    plt.plot(history.history['loss'])
    plt.plot(history.history['val_loss'])
    plt.title('model loss')
    plt.ylabel('loss')
    plt.xlabel('epoch')
    plt.legend(['train', 'test'], loc='upper left')
    plt.show()
    

![ZFjKVV](https://images.gitbook.cn/ZFjKVV.jpg)

* * *

### 7\. 模型预测

模型训练后，并且评估模型质量不错，就可以用来预测新的数据了。﻿

    
    
    predictions = model.predict(X)
    

### 8\. 保存模型

我们还可以将训练好的模型保存起来，可以保存模型结构，模型的权重，训练配置，如损失函数，优化器等，还有优化器的状态，下次使用时可以直接加载。﻿

    
    
    # 保存模型
    model.save('model.h5')
    jsonModel = model.to_jason()
    model.save_weights('modelWeight.h5')
    
    # 加载模型的权重
    modelWt = model.load_weights('modelweight.h5')
    

* * *

> ### 3\. 模型的改进

以上就是用 Keras 建立一个神经网络的流程，可以看出用 Keras
建立模型非常和符合直觉，顺序建模一气呵成，特别适合入门深度学习，并且快速建立模型得到结果。

接下来我们根据流程图，来看如何在每个环节对模型进行改进。

### 1\. dropout

我们可以在 建立模型 环节，加入 dropout 来改进模型，这是一种正则化的方法，简单来说就是随机地丢掉一些在隐藏层中传播的值。

首先，需要从 `keras.layers.core` 导入 Dropout， 将 `DROPOUT` 设置为 0.3， 迭代次数仍为 20
的话，准确率并没有增加，所以将 epoch 的数目增加到 250 次：

    
    
    from keras.layers.core import Dense, Dropout, Activation
    
    n_epoch = 250
    DROPOUT = 0.3
    

然后在模型的每个 Dense 层后面加了一个 Dropout：

    
    
    model = Sequential()
    model.add(Dense(n_hidden, input_shape=(RESHAPED,)))
    model.add(Activation('relu'))
    
    model.add(Dropout(DROPOUT))
    
    model.add(Dense(n_hidden))
    model.add(Activation('relu'))
    
    model.add(Dropout(DROPOUT))
    
    model.add(Dense(n_classes))
    model.add(Activation('softmax'))
    model.summary()
    

加入 dropout 模型前后的结构对比：

![LGIK7B](https://images.gitbook.cn/LGIK7B.jpg)

这时结果变为：

![wQkrR7](https://images.gitbook.cn/wQkrR7.jpg)

* * *

### 2\. 用正则化来减小过拟合

在 建立模型 这一部分，还可以加入正则化项来减少过拟合，例如：

    
    
    from keras import regularizers 
    model.add(Dense(64, input_dim=64, kernel_regularizer=regularizers.l2(0.01)))
    

* * *

### 3\. 选择 optimizer

在 编译模型 环节，我们还可以选择其他优化器，例如尝试 RMSprop，或者 Adam。

    
    
    from keras.optimizers import Adam
    optimizer = Adam()
    

![qyqq7z](https://images.gitbook.cn/qyqq7z.jpg)

可以看出 Adam 和前面同样的模型相比，准确率提高，而且它只需要迭代 20 代就可以得到这个结果。

此外可以增加隐藏层的神经元个数，不过因为参数增多了，模型训练的时间也会变长。﻿

还可以调节 batch size 这个参数找到一个最优点。

* * *

> ### 4\. Keras 中常用模块简介

以上，我们已经对 Keras
建立神经网络模型的基本过程，还有优化的技巧有了基本的认识，下面简单介绍一下在实战代码中经常用到的网络层和模块，我们在后面实战中遇到时会做详细讲解。

#### 1\. 核心网络层

在 Keras 的[核心网络层中](https://keras.io/zh/layers/core/﻿)，下面这些在几乎所有模型结构中都会用到：﻿

    
    
    from keras.layers import Dense, Activation, Dropout, Input, Masking
    from keras.layers import Reshape, Lambda, RepeatVector, Permute
    

`Dense`：全连接层

`Activation`：将指定的激活函数应用于 output
上，可以用：`softmax，elu，selu，softplus，softsign，relu，tanh，sigmoid，hard_sigmoid 和
linear`实例化。

`Dropout`：用指定的丢失率对 input 进行 Dropout 正则化。

`Masking`：当缺乏某几个时间步的数据时，用覆盖值覆盖序列。

`Reshape`：将 input 转换为指定的形状。

`Lambda`：将给定的函数包装为层，input 通过这个自定义的函数产生输出，这样用户就可以将自定义的函数添加到结构中。

`RepeatVector`：将 input 重复 n 次，这样如果输入是形状的 2D 的
tensor（#samples，＃feature）那么输出将是3D张量的形状（#samples，n，＃feature）。

`Permute`：按照指定的模式重新排序 input 的维度。

* * *

#### 2\. Embedding 层

    
    
    from keras.layers.embeddings import Embedding
    

`Embedding`：输入形状为 `(batch_size, sequence_length)` 的 2D 张量，。 输出形状为
`(batch_size, sequence_length, output_dim)` 的 3D 张量。

* * *

#### 3\. 融合层 Merge

Merge 层主要是对输入的若干个张量执行指定的运算， 里面经常用的有 Concatenate, Dot, Multiply：

    
    
    from keras.layers import Concatenate, Dot, Multiply
    

`Multiply`：计算输入张量的逐元素级乘积

`Concatenate`：沿指定的轴串联输入的张量

`Dot`：计算两个张量之间的点积

* * *

#### 4\. Normalization 层

    
    
    from keras.layers import BatchNormalization
    

`BatchNormalization`：批量标准化，标准化每批次的前一层的 output，使得该层的输有接近 0 的平均值和接近 1 的标准偏差。

* * *

#### 5\. 层封装器 wrappers

    
    
    from keras.layers import Bidirectional, TimeDistributed
    

`Bidirectional`：这是 RNN 的双向封装器，对序列进行前向和后向计算。

`TimeDistributed`：将一个层应用于 input 的每个时间片。

* * *

#### 6\. 卷积层和循环层

CNN，RNN 的层有很多可以选，例如我们可以选择常见的 Conv1D，GRU, LSTM

    
    
    from keras.layers import Conv1D
    from keras.layers import GRU,  LSTM
    

* * *

#### 7\. 数据预处理

这里有关于序列预处理，文本预处理，图像预处理的，我们在做序列模型的任务时，会用到 `pad_sequences`

    
    
    from keras.preprocessing import sequence
    from keras.preprocessing.sequence import pad_sequences
    

`pad_sequences`：用于将多个序列截断或补齐为相同的长度。

* * *

#### 8\. Callback

回调函数是一个函数的合集，用来查看训练模型的内在状态和统计：https://keras.io/zh/callbacks/

    
    
    from keras.callbacks import LambdaCallback
    from keras.callbacks import ModelCheckpoint
    

`LambdaCallback`：用于创建简单的 callback。 `ModelCheckpoint`：在每个训练期之后保存模型。

* * *

#### 9\. 初始化

初始化器的选择也有很多种：https://keras.io/zh/initializers/

    
    
    from keras.initializers import glorot_uniform
    

`glorot_uniform`：Glorot 均匀分布初始化器，也称为 Xavier 均匀分布初始化器。

* * *

通过今天的内容，相信大家已经对 Keras 有了初步印象，我们可以很容易地按照 8
步流程建立一个神经网络，第四部分的内容会在后面的实战案例中用到，那时再逐步深入。

