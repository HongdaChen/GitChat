今天我们将用 LSTM 来预测时间序列数据。代码包括两个版本，第一个是 TensorFlow 1，第二个是 Keras 的，二者有很多地方是差不多的，在
TensorFlow 中给出了很详细的解释，在 Keras 中会主要解释区别之处。

**时间序列**
是指由依赖于时间顺序的离散值或连续值所组成的序列。对这类数据来说时间是一个很重要的特征，我们在生活中处处可见这样的数据，例如气温、心电图、物价、每天的销售额，每天客服中心接到的电话数量、访问量、股价、油价、GDP
等等，时间可以用年、季、月、日、小时、分钟、秒等时间单位来度量。

因为应用非常广，预测时间序列的方法也有很多很多，大家感兴趣可以[看这里](https://en.wikipedia.org/wiki/Time_series)。

比较著名的统计模型有 Moving Average（MA），Autoregressive Integrated Moving
Average（ARIMA），Holt Winter’s Exponential
Smoothing（HWES）等等，这些方法一般都需要分析数据的趋势、季节性、周期性，保持数据的稳定性等条件。

![](https://images.gitbook.cn/52e54a90-38eb-11ea-8868-77fd80e627b4)

（图片来自：[algorithmia.com](https://algorithmia.com/blog/introduction-to-time-
series)）

但在现实生活中，很多时间序列问题的数据构成机制要么非常复杂，要么未知。如果用神经网络来处理这类问题就不是必须要分析出数据的季节性周期性等规律，只是用训练数据和优化算法训练模型使实际值和预测值等误差达到最小即可。

LSTM 在预测序列数据问题中有着很好的表现，它能从上下文中学习到时间的依赖性，能够比较好地处理有序的数据。

接下来我们通过一个 **预测股票价格** 的例子来看 LSTM 如何用于时间序列问题。

如果在股市如果我们可以正确地预测股票价格，就可以决定什么时候买入，什么时候卖出，进而获得利润。这个时候就需要用到时间序列模型，或者应用机器学习模型根据历史数据来预测未来的价格。

股票价格是很难被准确地预测的，因为它的时间序列数据中并没有固定可循的模式，不过我们还是可以应用机器学习技术来对数据进行建模，预测数据的趋势，看价格在未来是上升还是下降。

> 我们所用等数据来自 [kaggle](https://www.kaggle.com/dgawlik/nyse)： 是纽约证券交易所 [S&P
> 500](https://en.wikipedia.org/wiki/List_of_S%26P_500_companies) 数据集。

**训练 LSTM 预测时间序列分为以下几步：**

  1. 导入需要的包，建立会话
  2. 导入数据集，可视化
  3. 数据预处理 
    * 标准化
    * 划分训练集测试集
    * 选择一支股票，去掉不需要的列
    * 对训练集数据做 shuffle
  4. 建立 LSTM 模型
    * 定义模型参数
    * 定义函数获得 batch 数据
    * 定义 x、y 的 placeholder
    * 建立 LSTM 模型，得到输出值，定义损失和优化算法
  5. 运行图，训练 LSTM
  6. 可视化结果

### TensorFlow

#### **1\. 首先导入需要的包**

    
    
    import numpy as np
    import pandas as pd
    import math
    import sklearn
    import sklearn.preprocessing
    import datetime
    import os
    import matplotlib.pyplot as plt
    import tensorflow as tf
    
    
    
    # 将 default graph 重新初始化，保证内存中没有其他的 Graph，相当于清空所有的张量
    tf.reset_default_graph()
    
    # 为 graph 建立会话 session
    sess = tf.Session()
    

* * *

#### **2\. 导入所有股票价格数据，进行可视化**

    
    
    df = pd.read_csv("../input/prices-split-adjusted.csv", index_col = 0)
    

导入数据后，先看一下数据长什么样子，这个数据集有 501 家公司从 2016-01-05 到 2016-12-30 的股票价格，symbol
是公司的代号，volume 是成交量，每一条数据是某一天某个公司的 open、close、low、high 价格和当天交易量 volume。

    
    
    df.info()
    df.tail()
    df.describe()
    print('\nnumber of different stocks: ', len(list(set(df.symbol))))        # 不同股票的个数
    print(list(set(df.symbol))[:10])
    

输出结果：

    
    
    <class 'pandas.core.frame.DataFrame'>
    Index: 851264 entries, 2016-01-05 to 2016-12-30
    Data columns (total 6 columns):
    symbol    851264 non-null object
    open      851264 non-null float64
    close     851264 non-null float64
    low       851264 non-null float64
    high      851264 non-null float64
    volume    851264 non-null float64
    dtypes: float64(5), object(1)
    memory usage: 45.5+ MB
    
                symbol  open    close   low high    volume
    date                        
    2016-12-30    ZBH 103.309998  103.199997  102.849998  103.930000  973800.0
    2016-12-30    ZION    43.070000   43.040001   42.689999   43.310001   1938100.0
    2016-12-30    ZTS 53.639999   53.529999   53.270000   53.740002   1701200.0
    2016-12-30    AIV 44.730000   45.450001   44.410000   45.590000   1380900.0
    2016-12-30    FTV 54.200001   53.630001   53.389999   54.480000   705100.0
    
            open    close   low high    volume
    count    851264.000000   851264.000000   851264.000000   851264.000000   8.512640e+05
    mean    64.993618   65.011913   64.336541   65.639748   5.415113e+06
    std    75.203893   75.201216   74.459518   75.906861   1.249468e+07
    min    1.660000    1.590000    1.500000    1.810000    0.000000e+00
    25%    31.270000   31.292776   30.940001   31.620001   1.221500e+06
    50%    48.459999   48.480000   47.970001   48.959999   2.476250e+06
    75%    75.120003   75.139999   74.400002   75.849998   5.222500e+06
    max    1584.439941 1578.130005 1549.939941 1600.930054 8.596434e+08
    
    number of different stocks:  501
    ['AMAT', 'BDX', 'VZ', 'MSI', 'TJX', 'PSX', 'COST', 'WM', 'WAT', 'UPS']
    

我们可以将某个公司例如 'EQIX' 的五个数据进行可视化，看一下随时间变化的趋势：

    
    
    plt.figure(figsize=(15, 5));
    plt.subplot(1,2,1);
    plt.plot(df[df.symbol == 'EQIX'].open.values, color='red', label='open')
    plt.plot(df[df.symbol == 'EQIX'].close.values, color='green', label='close')
    plt.plot(df[df.symbol == 'EQIX'].low.values, color='blue', label='low')
    plt.plot(df[df.symbol == 'EQIX'].high.values, color='yellow', label='high')
    plt.title('stock price')
    plt.xlabel('time [days]')
    plt.ylabel('price')
    plt.legend(loc='best')
    
    plt.subplot(1,2,2);
    plt.plot(df[df.symbol == 'EQIX'].volume.values, color='black', label='volume')
    plt.title('stock volume')
    plt.xlabel('time [days]')
    plt.ylabel('volume')
    plt.legend(loc='best');
    

![](https://images.gitbook.cn/e7080ad0-38ed-11ea-92fb-2922fbf27559)

* * *

#### **3\. 数据预处理**

**3.1 标准化**

定义函数，将数据集的 open high low close 四列数据进行 MinMaxScaler 标准化。

    
    
    def normalize_data(df):
        min_max_scaler = sklearn.preprocessing.MinMaxScaler()
        df['open'] = min_max_scaler.fit_transform(df.open.values.reshape(-1,1))
        df['high'] = min_max_scaler.fit_transform(df.high.values.reshape(-1,1))
        df['low'] = min_max_scaler.fit_transform(df.low.values.reshape(-1,1))
        df['close'] = min_max_scaler.fit_transform(df['close'].values.reshape(-1,1))
        return df
    

**MinMaxScaler** 是一种标准化方法，用来将特征的数值缩放到指定的最小值到最大值之间，`fit_transform`
将填充数据并对数据进行此转换。

它的计算公式为：

    
    
    X_std = (X - X.min(axis=0)) / (X.max(axis=0) - X.min(axis=0))
    X_scaled = X_std * (max - min) + min
    

常用的是将数据缩放到 0~1 之间。通过标准化处理，可以使各个特征具有相同的尺度，在训练神经网络时能够加速权重参数的收敛。

* * *

**3.2 将数据集分为训练集测试集**

在下面的代码中我们首先用长为 `seq_len` 的窗口截取原数据，构造新的数据集 X 和 Y，再按设定的训练集测试集比例进行划分。

**如何构造 X 和 Y**

**对于一个典型的 LSTM 来说，如果我们的 input 序列是 [a,b,c,d,e]，那么它的 target 是 [b,c,d,e,f]，**
例如我们有数据序列 [a,b,c,d,e,f,g,h,i]，用长度为 6 的滑动窗口截取得到四个序列：

    
    
    [
        [a,b,c,d,e,f],
        [b,c,d,e,f,g],
        [c,d,e,f,g,h],
        [d,e,f,g,h,i],
    ]
    

那么 input X 就是：

    
    
    [
        [a,b,c,d,e],
        [b,c,d,e,f],
        [c,d,e,f,g],
        [d,e,f,g,h],
    ]
    

LSTM 要预测的 target Y 就是：

    
    
    [
        [b,c,d,e,f],
        [c,d,e,f,g],
        [d,e,f,g,h],
        [e,f,g,h,i],
    ]
    

在 many-to-one 的例子中，就只要最后的一个结果，即 f，g，h，i。

下面是完整代码，其中设定数据集划分的比例为 **80% 的训练集，20% 的测试集** :

    
    
    train_set_size_percentage = 80 
    
    def load_data(stock, seq_len):
        # 将数据转化为 numpy array
        data_raw = stock.as_matrix()     
        data = []
    
        # 构建数据 data: 用长为 seq_len 的窗口，从头截取 data_raw 中的数据，每一段为 data 中的一条样本
        for index in range( len(data_raw) - seq_len ): 
            data.append( data_raw[index: index + seq_len] )
    
        data = np.array(data);
    
        # 根据设置的比例计算 训练集，测试集 的大小    
        train_set_size = int( np.round( train_set_size_percentage / 100 * data.shape[0] ) );
    
        # data 中从开始到 train_set_size 这些是 训练集，剩下那些是测试集    
        x_train = data[:train_set_size, :-1, :]
        y_train = data[:train_set_size, -1, :]
    
        x_test = data[train_set_size:, :-1, :]
        y_test = data[train_set_size:, -1, :]
    
        return [x_train, y_train, x_test, y_test]
    

* * *

**3.3 选择一支股票进行预测，去掉 symbol、volume 两列**

volume 是当天交易量，我们这里只预测价格，可以先将它去掉，选定 symbol 后也将这一列去掉，这支股票也可以换成其它公司的。

    
    
    # 只留下 ['open', 'close', 'low', 'high'] 四列
    df_stock = df[df.symbol == 'EQIX'].copy()
    df_stock.drop(['symbol'], 1, inplace = True)
    df_stock.drop(['volume'], 1, inplace = True)
    
    cols = list(df_stock.columns.values)
    print('df_stock.columns.values = ', cols)  
    

输出结果：

    
    
    df_stock.columns.values =  ['open', 'close', 'low', 'high']
    

删掉两列，剩下的数据有四列都是价格：

![](https://images.gitbook.cn/878dcad0-38ee-11ea-a09a-339e8e51a043)

然后就可以对选定数据的四列进行 **min-max normalization** ：

    
    
    df_stock_norm = df_stock.copy()
    df_stock_norm = normalize_data(df_stock_norm)
    

标准化之后的数据是这样的：

![](https://images.gitbook.cn/9b6634c0-38ee-11ea-a508-65c0fa00bba7)

接着设定窗口序列长度为 20， **生成 训练集，测试集的 x 和 y** ，可以看一下各自有多少数据。

    
    
    seq_len = 20 
    x_train, y_train, x_test, y_test = load_data(df_stock_norm, seq_len)
    
    print('x_train.shape = ',x_train.shape)
    print('y_train.shape = ', y_train.shape)
    print('x_test.shape = ', x_test.shape)
    print('y_test.shape = ',y_test.shape)
    

输出结果：

    
    
    x_train.shape =  (1394, 19, 4)
    y_train.shape =  (1394, 4)
    x_test.shape =  (348, 19, 4)
    y_test.shape =  (348, 4)
    

在划分之前的数据集中我们取前 30 行看一下：

![](https://images.gitbook.cn/adbb3300-38ee-11ea-97b4-9def676d9f8e)

用 `load_data` 获得 训练集，验证集，测试集后，举例来说，`seq_len = 20`，所以 `x_train[0]` 为 前 19 个数据：

![](https://images.gitbook.cn/bb9ecd60-38ee-11ea-92fb-2922fbf27559)

`y_train[0]` 就是第 20 个数据：

    
    
    array([ 0.07836361,  0.07994109,  0.08194543,  0.07363532])
    

再往后看，`x_train[1]` 就是第 2 到 20 个数据：

![](https://images.gitbook.cn/d8b93610-38ee-11ea-a897-97fbe818b058)

`y_train[1]` 就是第 21 个数据：

    
    
    array([ 0.07591279,  0.08222871,  0.07998228,  0.07180312])
    

就是每次都向后移动一位，获得 x 和 y。

将预处理后的这支股票的数据 `df_stock_norm` 再次可视化：

    
    
    plt.figure(figsize=(15, 5));
    plt.plot(df_stock_norm.open.values, color='red', label='open')
    plt.plot(df_stock_norm.close.values, color='green', label='low')
    plt.plot(df_stock_norm.low.values, color='blue', label='low')
    plt.plot(df_stock_norm.high.values, color='yellow', label='high')
    plt.title('stock')
    plt.xlabel('time [days]')
    plt.ylabel('normalized price/volume')
    plt.legend(loc='best')
    plt.show()
    

![](https://images.gitbook.cn/ee401350-38ee-11ea-a09a-339e8e51a043)

* * *

**3.4 对训练集数据做 shuffle**

    
    
    perm_array  = np.arange(x_train.shape[0])
    np.random.shuffle(perm_array)
    

* * *

#### **4\. 建立 LSTM 模型**

**4.1 定义`get_next_batch` 来获取每一批的训练数据**

由 start 和 end 截取 shuffle 后的数据 `perm_array`， 来得到每一批数据。

    
    
    index_in_epoch = 0;
    
    def get_next_batch(batch_size):
        global index_in_epoch, x_train, perm_array   
    
        start = index_in_epoch
    
        # 每次获取完一批后，index 向后更新一次
        index_in_epoch += batch_size    
    
        # 如果 end 超过了训练集数量，就重新 shuffle 数据，从头开始取 batch
        if index_in_epoch > x_train.shape[0]:  
            np.random.shuffle(perm_array) 
            start = 0 
            index_in_epoch = batch_size
    
        end = index_in_epoch
    
        return x_train[perm_array[start:end]], y_train[perm_array[start:end]]
    

* * *

**4.2 定义模型参数**

下面参数代表含义为：

  * 根据窗口的设定，每个训练数据是长度等于 `seq_len - 1` 的序列，所以展开 LSTM n_steps ＝ `seq_len - 1` 个时间步，
  * 每个 input 有 4 个特征，即在每个时刻有 `n_inputs＝ 4` 个值，open high low close，
  * `num_units` 为 LSTM cell 中 h 和 c 的维度，LSTM 的结构在上一篇文章中有详细的介绍，
  * 每个预测结果也是包含 `n_outputs ＝ 4` 个值。

关于这几个超参数，在下文还有更直观的图解。

    
    
    n_steps = seq_len - 1 
    n_inputs = 4 
    num_units = 200     
    n_outputs = 4                
    
    learning_rate = 0.001
    batch_size = 50
    n_epochs = 100 
    train_set_size = x_train.shape[0]
    test_set_size = x_test.shape[0]
    

**4.3 定义 x y 的 placeholder**

    
    
    X = tf.placeholder(tf.float32, [None, n_steps, n_inputs])
    y = tf.placeholder(tf.float32, [None, n_outputs])
    

* * *

**4.4 建立 LSTM**

首先建立一个基本 LSTM 单元，然后构造多步 LSTM：

    
    
    cell = tf.contrib.rnn.BasicLSTMCell(num_units = num_units, activation = tf.nn.elu)
    rnn_outputs, states = tf.nn.dynamic_rnn(cell, X, dtype = tf.float32)
    

**1\. 下面是一个基本的 LSTM cell:**

    
    
    tf.contrib.rnn.BasicLSTMCell(num_units = num_units, activation = tf.nn.elu) 
    

其中 `num_units` 指的是 LSTM cell 中当前状态 h 和长期状态 c 的维度.

![](https://images.gitbook.cn/056b03a0-38ef-11ea-b7a3-cd73d417f46b)

**2\. 要构建一个多步 LSTM 用`dynamic_rnn`：**

    
    
    tf.nn.dynamic_rnn(cell, inputs)
    

其中 inputs 接收一个 tensors 的列表，这个列表的长度就是时间步的个数，每个 tensor 对应着一个时刻的输入 x，它的形状为
`[batch_size, input_size]`，其中 `batch_size` 为每一批样本数，`input_size`
为每个输入的特征个数，`dynamic_rnn` 的输出是一个 tensors 列表，每个 tensor 的形状为 `[batch_size,
num_units]`，同样这个列表的长度就是时间步的个数。

![](https://images.gitbook.cn/34c5f830-38ef-11ea-87ea-632bd89cb3c5)

**3\. 得到输出 outputs**

    
    
    stacked_rnn_outputs = tf.reshape(rnn_outputs, [-1, num_units]) 
    stacked_outputs = tf.layers.dense(stacked_rnn_outputs, n_outputs)
    outputs = tf.reshape(stacked_outputs, [-1, n_steps, n_outputs])
    outputs = outputs[:,n_steps-1,:] 
    

  * 将上一步 LSTM 的输出 `rnn_outputs` 的维度从 `[batch_size, n_steps, num_units]` 变换到 `[batch_size * n_steps, num_units]`；
  * 然后用一个全连接层，它的输出大小和预测输出值维度相等 ＝ `n_outputs`，这样输出的维度为 `[batch_size * n_steps, n_outputs]`；
  * 接着再把这个 tensor 变为 `[batch_size, n_steps, n_outputs]`；
  * 最后只保留 outputs 的最后一个时间步 `n_steps - 1` 时的输出。

![](https://images.gitbook.cn/4f055470-38ef-11ea-a897-97fbe818b058)

**4\. 定义 Loss 和 optimizer**

这里损失函数为 MSE，优化算法为 AdamOptimizer：

    
    
    loss = tf.reduce_mean(tf.square(outputs - y))  
    optimizer = tf.train.AdamOptimizer(learning_rate=learning_rate) 
    training_op = optimizer.minimize(loss)
    

* * *

#### **5\. 运行图**

初始化变量，在每次迭代中，将训练集批次数据喂到 `training_op`，用 AdamOptimizer 训练模型使 MSE
达到最优，并输出训练集和测试集的 MSE，保存训练集，测试集的预测值进行可视化。

    
    
    train_loss = []
    test_loss = []
    
    with tf.Session() as sess: 
    
        sess.run(tf.global_variables_initializer())
    
        for epoch in range(n_epochs):
    
            num_batches = int(train_set_size / batch_size) + 1
    
            for i in range(num_batches):
                # 获得每一批的 x 和 y 
                x_batch, y_batch = get_next_batch(batch_size) 
                # 训练模型时，将 训练集 x 和 y 喂给网络，然后用优化算法训练
                sess.run(training_op, feed_dict={X: x_batch, y: y_batch})   
    
            # 计算 训练集 和 测试集的 MSE
            mse_train = loss.eval(feed_dict={X: x_train, y: y_train})     
            mse_test = loss.eval(feed_dict={X: x_test, y: y_test}) 
    
            train_loss.append(mse_train)
            test_loss.append(mse_test)
    
            if epoch % 10 == 0:
                print('Epoch: {}, MSE train: {:.6}, MSE test: {:.6}'.format(epoch+1, mse_train, mse_test))
    
        # 当输入数据集分别为 训练集，测试集 时，得到相应的预测值
        y_train_pred = sess.run(outputs, feed_dict={X: x_train})    
        y_test_pred = sess.run(outputs, feed_dict={X: x_test})
    

运行结果：

![](https://images.gitbook.cn/c632ab10-38ef-11ea-97b4-9def676d9f8e)

* * *

#### **6\. 可视化结果**

    
    
    # ft：代表预测值 y 的第几列，0 = open, 1 = close, 2 = highest, 3 = lowest 
    ft = 0  
    
    plt.figure(figsize=(15, 5));
    
    plt.subplot(1,2,1);
    
    # 画出训练集的实际值和预测值
    plt.plot(np.arange(y_train.shape[0]), y_train[:,ft], color='blue', label='train target')
    plt.plot(np.arange(y_train_pred.shape[0]),y_train_pred[:,ft], color='red',
             label='train prediction')
    
    # 画出测试集的实际值和预测值
    plt.plot(np.arange(y_train.shape[0], y_train.shape[0] + y_test.shape[0]), y_test[:,ft],     
             color='gray', label='test target')
    plt.plot(np.arange(y_train_pred.shape[0], y_train_pred.shape[0]+y_test_pred.shape[0]),
             y_test_pred[:,ft], color='green', label='test prediction')
    
    plt.title('past and future stock prices')
    plt.xlabel('time [days]')
    plt.ylabel('normalized price')
    plt.legend(loc='best');
    
    plt.subplot(1,2,2);
    
    # 只看测试集的实际值和预测值
    plt.plot(np.arange(y_train.shape[0],
                       y_train.shape[0] + y_test.shape[0]),         
             y_test[:,ft], color='grey', label='test target')
    plt.plot(np.arange(y_train_pred.shape[0],
                       y_train_pred.shape[0] + y_test_pred.shape[0]),
             y_test_pred[:,ft], color='green', label='test prediction')
    
    plt.title('future stock prices')
    plt.xlabel('time [days]')
    plt.ylabel('normalized price')
    plt.legend(loc='best');
    

![](https://images.gitbook.cn/03764ed0-390b-11ea-ae5b-e5a1c8132838)

* * *

### Keras

下面我们用 Keras 再做一下，它们的很多步骤都是差不多的，就是在模型部分 Keras 会简化很多。

    
    
    import numpy as np
    import matplotlib.pyplot as plt
    import pandas as pd
    import math, time
    import sklearn
    import sklearn.preprocessing
    from pandas import datetime
    import itertools
    import datetime
    from operator import itemgetter
    from sklearn.metrics import mean_squared_error
    from math import sqrt
    from keras.models import Sequential
    from keras.layers.core import Dense, Dropout, Activation
    from keras.layers.recurrent import LSTM
    from keras.models import load_model
    import keras
    import h5py
    import requests
    import os
    
    test_set_size_percentage = 10 
    
    #display parent directory and working directory
    print(os.path.dirname(os.getcwd())+':', os.listdir(os.path.dirname(os.getcwd())));
    print(os.getcwd()+':', os.listdir(os.getcwd()));
    

**导入数据，通过 symbol 查看有多少家公司的股票：**

    
    
    df = pd.read_csv("../input/prices-split-adjusted.csv", index_col = 0)
    
    print('\nnumber of different stocks: ', len(list(set(df.symbol))))
    print(list(set(df.symbol))[:10])
    
    
    
    df.head()
    
    df.shape
    
    df.isnull().sum()
    

> number of different stocks: 501 ['ADSK', 'ECL', 'EXPD', 'DISCA', 'UNM',
> 'COG', 'ZTS', 'TRV', 'DVA', 'MAA']
>
> (851264, 6)
>
> symbol 0 open 0 close 0 low 0 high 0 volume 0 dtype: int64

![](https://images.gitbook.cn/1c943580-390b-11ea-8868-77fd80e627b4)

**定义标准化的函数，这个和第一版代码一样的，对 open、high、low、close 四个价格应用 MinMaxScaler：**

    
    
    def normalize_data(df):
        min_max_scaler = sklearn.preprocessing.MinMaxScaler()
        df['open'] = min_max_scaler.fit_transform(df.open.values.reshape(-1,1))
        df['high'] = min_max_scaler.fit_transform(df.high.values.reshape(-1,1))
        df['low'] = min_max_scaler.fit_transform(df.low.values.reshape(-1,1))
        df['close'] = min_max_scaler.fit_transform(df['close'].values.reshape(-1,1))
        return df
    

**划分数据集这里和前一版稍有不同，因为在后面 model 那里会指定验证集的比例，所以这里只将数据集分成训练集和测试集：**

    
    
    def load_data_keras(stock, seq_len):
        data_raw = stock.as_matrix() 
        data = []
    
        for index in range(len(data_raw) - seq_len): 
            data.append(data_raw[index: index + seq_len])
    
        data = np.array(data);
    
        test_set_size = int(np.round(test_set_size_percentage/100*data.shape[0]));
        train_set_size = data.shape[0] - test_set_size;
    
        x_train = data[:train_set_size,:-1,:]
        y_train = data[:train_set_size,-1,:]
    
        x_test = data[train_set_size:,:-1,:]
        y_test = data[train_set_size:,-1,:]
    
        return [x_train, y_train, x_test, y_test]
    
    
    
    # 选择 EQIX 这一支
    df_stock = df[df.symbol == 'EQIX'].copy()
    df_stock.head()
    

![](https://images.gitbook.cn/e3870f80-38ef-11ea-a09a-339e8e51a043)

**可以看一下它的价格走势：**

    
    
    global closing_stock
    global opening_stock
    
    f, axs = plt.subplots(2,2,figsize=(8,8))
    plt.subplot(212)
    
    company = df[df['symbol']=='EQIX']
    company = company.open.values.astype('float32')
    company = company.reshape(-1, 1)
    opening_stock = company
    
    plt.grid(True)
    plt.xlabel('Time')
    plt.ylabel('EQIX' + " open stock prices")
    plt.title('prices Vs Time')
    plt.plot(company , 'g')
    
    plt.subplot(211)
    company_close = df[df['symbol']=='EQIX']
    company_close = company_close.close.values.astype('float32')
    company_close = company_close.reshape(-1, 1)
    closing_stock = company_close
    
    plt.xlabel('Time')
    plt.ylabel('EQIX' + " close stock prices")
    plt.title('prices Vs Time')
    plt.grid(True)
    plt.plot(company_close , 'b')
    plt.show()
    

![](https://images.gitbook.cn/50344210-38f0-11ea-ae5b-e5a1c8132838)

**这部分是一样的，删掉 symbol 和 volume 两列，只保留四个价格：**

    
    
    df_stock.drop(['symbol'],1,inplace=True)
    df_stock.drop(['volume'],1,inplace=True)
    cols = list(df_stock.columns.values)
    print('df_stock.columns.values = ', cols)
    df_stock.head()
    

![](https://images.gitbook.cn/a65adaa0-38f0-11ea-b7a3-cd73d417f46b)

**得到训练集和测试集：**

    
    
    df_stock_norm = df_stock.copy()
    df_stock_norm = normalize_data(df_stock_norm)
    
    seq_len = 20 
    x_train_keras, y_train_keras, x_test_keras, y_test_keras = load_data_keras(df_stock_norm, seq_len)
    

**模型这里就相对简单了很多：**

输入模型的参数为 `[4,seq_len-1,4]`，第一个 4 是指 x 有 4 列，`seq_len-1` 是每一批即 `x_i` 的长度，每个 `x
i` 有 19 条数据，第二个 4 是 y 有 4 列。

    
    
    def build_model(layers):
        d = 0.3
        model = Sequential()
    
        model.add(LSTM(256, input_shape=(layers[1], layers[0]), return_sequences=True))
        model.add(Dropout(d))
    
        model.add(LSTM(256, input_shape=(layers[1], layers[0]), return_sequences=False))
        model.add(Dropout(d))
    
        model.add(Dense(4,kernel_initializer="uniform",activation='linear'))
    
        start = time.time()
        model.compile(loss='mse',optimizer='adam', metrics=['accuracy'])
        print("Compilation Time : ", time.time() - start)
        return model
    
    model = build_model([4,seq_len-1,4])
    

**训练模型：**

    
    
    model.fit(x_train_keras,y_train_keras,batch_size=50,epochs=100,validation_split=0.1,verbose=1)
    
    
    
    def model_score(model, X_train, y_train, X_test, y_test):
        trainScore = model.evaluate(X_train, y_train, verbose=0)
        print('Train Score: %.5f MSE (%.2f RMSE)' % (trainScore[0], math.sqrt(trainScore[0])))
    
        testScore = model.evaluate(X_test, y_test, verbose=0)
        print('Test Score: %.5f MSE (%.2f RMSE)' % (testScore[0], math.sqrt(testScore[0])))
        return trainScore[0], testScore[0]
    
    
    model_score(model, x_train_keras, y_train_keras, x_test_keras, y_test_keras)
    

> Train Score: 0.00013 MSE (0.01 RMSE) Test Score: 0.00068 MSE (0.03 RMSE)

* * *

通过时间序列的例子，我们知道了 LSTM 的预测是这样的，如果 input 序列是 [a,b,c,d,e]，那么它的 target 是
[b,c,d,e,f]，所以要用滑动窗口构造出序列数据。

而且图解了 LSTM 中 `n_steps， n_inputs， num_units， n_outputs` 等参数的含义，还有 outputs 的构造，对
LSTMcell 有了更深入地了解。

下一篇将介绍语言模型，将对序列的预测应用到语言领域。

* * *

时间序列的问题类型有很多种，这里加一个彩蛋，这是我之前写过的根据问题的输入输出模式划分，六种时间序列问题所对应的 LSTM 模型结构的 Keras 实现。

![](https://images.gitbook.cn/c3f39b10-38f0-11ea-a508-65c0fa00bba7)

#### **1\. Univariate**

![](https://images.gitbook.cn/d24d9df0-38f0-11ea-a09a-339e8e51a043)

Univariate 是指：

  * input 为多个时间步
  * output 为一个时间的问题

**数例：**

    
    
    训练集：
    X,            y
    10, 20, 30        40
    20, 30, 40        50
    30, 40, 50        60
    …
    
    
    预测输入：
    X，
    70, 80, 90
    

**模型的 Keras 代码：**

    
    
    # define model【Vanilla LSTM】
    
    model = Sequential()
    model.add( LSTM(50,  activation='relu',  input_shape = (n_steps, n_features)) )
    model.add( Dense(1) )
    model.compile(optimizer='adam', loss='mse')
    
    n_steps = 3
    n_features = 1
    

其中：

  * `n_steps` 为输入的 X 每次考虑几个 **时间步**
  * `n_features` 为每个时间步的 **序列数**

这个是最基本的模型结构，我们后面几种模型会和这个进行比较。

* * *

#### **2\. Multiple Input**

![](https://images.gitbook.cn/052c6210-38f1-11ea-8ae1-1b3ffbbc095a)

Multiple Input 是指：

  * input 为多个序列
  * output 为一个序列的问题

**数例：**

    
    
    训练集：
    X，       y
    [[10 15]
     [20 25]
     [30 35]] 65
    [[20 25]
     [30 35]
     [40 45]] 85
    [[30 35]
     [40 45]
     [50 55]] 105
    [[40 45]
     [50 55]
     [60 65]] 125
    …
    
    
    预测输入：
    X，
    80,     85
    90,     95
    100,     105
    

即数据样式为：

    
    
    in_seq1： [10, 20, 30, 40, 50, 60, 70, 80, 90]
    in_seq2： [15, 25, 35, 45, 55, 65, 75, 85, 95]
    
    out_seq： [in_seq1[i]+in_seq2[i] for i in range(len(in_seq1))]
    

**模型的 Keras 代码：**

    
    
    # define model【Vanilla LSTM】
    model = Sequential()
    model.add(LSTM(50, activation='relu', input_shape=(n_steps, n_features)))
    model.add(Dense(1))
    model.compile(optimizer='adam', loss='mse')
    
    n_steps = 3
    # 此例中 n features = 2，因为输入有两个并行序列
    n_features = X.shape[2]    
    

其中：

  * `n_steps` 为输入的 X 每次考虑几个时间步
  * `n_features` 此例中 = 2，因为输入有 **两个并行序列**

**和 Univariate 相比：**

模型的结构代码是一样的，只是在 `n_features = X.shape[2]`，而不是 1。

* * *

#### **3\. Multiple Parallel**

![](https://images.gitbook.cn/249f87d0-38f1-11ea-8868-77fd80e627b4)

Multiple Parallel 是指：

  * input 为多个序列
  * output 也是多个序列的问题

**数例：**

    
    
    训练集：
    X,            y
    [[10 15 25]
     [20 25 45]
     [30 35 65]] [40 45 85]
    [[20 25 45]
     [30 35 65]
     [40 45 85]] [ 50  55 105]
    [[ 30  35  65]
     [ 40  45  85]
     [ 50  55 105]] [ 60  65 125]
    [[ 40  45  85]
     [ 50  55 105]
     [ 60  65 125]] [ 70  75 145]
    …
    
    
    预测输入：
    X，
    70, 75, 145
    80, 85, 165
    90, 95, 185
    

**模型的 Keras 代码：**

    
    
    # define model【Vanilla LSTM】
    model = Sequential()
    model.add(LSTM(100, activation='relu', return_sequences=True, input_shape=(n_steps, n_features)))
    model.add(Dense(n_features))
    model.compile(optimizer='adam', loss='mse')
    
    n_steps = 3
    # 此例中 n features = 3，因为输入有3个并行序列
    n_features = X.shape[2]       
    

其中：

  * `n_steps` 为输入的 X 每次考虑几个时间步
  * `n_features` 此例中 = 3，因为输入有 3 个并行序列

**和 Univariate 相比：**

模型结构的定义中，多了一个 `return_sequences=True`，即返回的是序列，输出为 `Dense(n_features)`，而不是 1。

* * *

#### **4\. Multi-Step**

![](https://images.gitbook.cn/452a1100-38f1-11ea-a09a-339e8e51a043)

Multi-Step 是指：

  * input 为多个时间步
  * output 也是 **多个时间步** 的问题

**数例：**

    
    
    训练集：
    X,            y
    [10 20 30] [40 50]
    [20 30 40] [50 60]
    [30 40 50] [60 70]
    [40 50 60] [70 80]
    …
    
    
    预测输入：
    X，
    [70, 80, 90]
    

**模型的 Keras 代码：**

    
    
    # define model【Vanilla LSTM】
    model = Sequential()
    model.add(LSTM(100, activation='relu', return_sequences=True, input_shape=(n_steps_in, n_features)))
    model.add(LSTM(100, activation='relu'))
    model.add(Dense(n_steps_out))
    model.compile(optimizer='adam', loss='mse')
    
    n_steps_in, n_steps_out = 3, 2
    n_features = 1     
    

其中：

  * `n_steps_in` 为输入的 X 每次考虑几个时间步
  * `n_steps_out` 为输出的 y 每次考虑几个时间步
  * `n_features` 为输入有几个序列

**和 Univariate 相比：**

模型结构的定义中，多了一个 `return_sequences=True`，即返回的是序列，而且 `input_shape=(n_steps_in,
n_features)` 中有代表输入时间步数的 `n_steps_in`，输出为 `Dense(n_steps_out)`，代表输出的 y
每次考虑几个时间步。

**当然这个问题还可以用 Encoder-Decoder 结构实现：**

    
    
    # define model【Encoder-Decoder Model】
    model = Sequential()
    model.add(LSTM(100, activation='relu', input_shape=(n_steps_in, n_features)))
    model.add(RepeatVector(n_steps_out))
    model.add(LSTM(100, activation='relu', return_sequences=True))
    model.add(TimeDistributed(Dense(1)))
    model.compile(optimizer='adam', loss='mse')
    

* * *

#### **5\. Multivariate Multi-Step**

![](https://images.gitbook.cn/69e9b860-38f1-11ea-8ae1-1b3ffbbc095a)

Multivariate Multi-Step 是指：

  * input 为多个序列
  * output 为多个时间步的问题

**数例：**

    
    
    训练集：
    X,            y
    [[10 15]
     [20 25]
     [30 35]] [65 
              85]
    [[20 25]
     [30 35]
     [40 45]] [ 85
               105]
    [[30 35]
     [40 45]
     [50 55]] [105 
             125]
    …
    
    
    预测输入：
    X，
    [40 45]
     [50 55]
     [60 65]
    

**模型的 Keras 代码：**

    
    
    # define model
    model = Sequential()
    model.add(LSTM(100, activation='relu', return_sequences=True, input_shape=(n_steps_in, n_features)))
    model.add(LSTM(100, activation='relu'))
    model.add(Dense(n_steps_out))
    model.compile(optimizer='adam', loss='mse')
    
    n_steps_in, n_steps_out = 3, 2
    # 此例中 n features = 2，因为输入有2个并行序列  
    n_features = X.shape[2]        
    

其中：

  * `n_steps_in` 为输入的 X 每次考虑几个时间步
  * `n_steps_out` 为输出的 y 每次考虑几个时间步
  * `n_features` 为输入有几个序列，此例中 = 2，因为输入有 2 个并行序列 

**和 Univariate 相比：**

模型结构的定义中，多了一个 `return_sequences=True`，即返回的是序列，而且 `input_shape=(n_steps_in,
n_features)` 中有代表输入时间步数的 `n_steps_in`，输出为 `Dense(n_steps_out)`，代表输出的 y
每次考虑几个时间步，另外 `n_features = X.shape[2]`，而不是 1，相当于是 Multivariate 和 Multi-Step
的结构组合起来。

* * *

#### **6\. Multiple Parallel Input & Multi-Step Output**

![](https://images.gitbook.cn/a4c57820-38f1-11ea-a508-65c0fa00bba7)

Multiple Parallel Input & Multi-Step Output 是指：

  * input 为多个序列
  * output 也是多个序列 & 多个时间步的问题

**数例：**

    
    
    训练集：
    X,            y
    [[10 15 25]
     [20 25 45]
     [30 35 65]] [[ 40  45  85]
               [ 50  55 105]]
    [[20 25 45]
     [30 35 65]
     [40 45 85]] [[ 50  55 105]
               [ 60  65 125]]
    [[ 30  35  65]
     [ 40  45  85]
     [ 50  55 105]] [[ 60  65 125]
                  [ 70  75 145]]
    …
    
    
    预测输入：
    X，
    [[ 40  45  85]
     [ 50  55 105]
     [ 60  65 125]]
    

**模型的 Keras 代码：**

    
    
    # define model【Encoder-Decoder model】
    model = Sequential()
    model.add(LSTM(200, activation='relu', input_shape=(n_steps_in, n_features)))
    model.add(RepeatVector(n_steps_out))
    model.add(LSTM(200, activation='relu', return_sequences=True))
    model.add(TimeDistributed(Dense(n_features)))
    model.compile(optimizer='adam', loss='mse')
    
    n_steps_in, n_steps_out = 3, 2
    # 此例中 n features = 3，因为输入有3个并行序列   
    n_features = X.shape[2]       
    

其中：

  * `n_steps_in` 为输入的 X 每次考虑几个时间步
  * `n_steps_out` 为输出的 y 每次考虑几个时间步
  * `n_features` 为输入有几个序列

**这里我们和 Multi-Step 的 Encoder-Decoder 相比：**

二者的模型结构，只是在最后的输出层参数不同，`TimeDistributed(Dense(n_features))` 而不是 `Dense(1)`。

* * *

无论是时间序列，还是股票预测，包括本专栏中的其他一些实战课题，所涵盖的内容远远不止文章中的这些，都是差不多作为独立的学科存在的，所以非常鼓励大家如果有兴趣就更深入地探索一下。

* * *

参考文献：

  * Giancarlo Zaccone, Md. Rezaul Karim, 《Deep Learning with TensorFlow - Second Edition》 [NY Stock Price Prediction RNN LSTM GRU](https://www.kaggle.com/raoulma/ny-stock-price-prediction-rnn-lstm-gru)

