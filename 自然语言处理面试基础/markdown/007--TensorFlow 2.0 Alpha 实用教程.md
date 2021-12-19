在这篇文章中我们将介绍一下 **TensorFlow 2.0 Alpha** 的内容，在课程准备的途中赶上了 TensorFlow 的 2019
开发人员峰会，TensorFlow 团队推出了 Alpha 版的 TensorFlow 2.0，我们这门课前一半的代码目前还是 1.x
版本的，在课程结束后会逐渐补充 2.0 版本的代码，在那之前我们先来看一下 TensorFlow 2.0 Alpha 的简易教程，这样如果大家有兴趣直接用
2.0 实现的话，这个教程会对大家有所帮助。

**本文结构：**

  1. TF 2.0 的优点
  2. Keras 和 TF 2.0 的比较
  3. TF 1.x 和 2.0 的比较
  4. 如何将旧代码升级：自动更新
  5. 如何将旧代码升级：深度学习常用技术 1.x 到 2.0 的转换

* * *

### 1\. TF 2.0 的优点

今年 3 月 6-7 日，在 TensorFlow 的 2019 峰会上正式推出了 2.0 Alpha 版本。

TensorFlow 可以说是最受欢迎的深度学习框架了，那么 **为什么要有 TensorFlow 2.0 呢？**

《Hands-On Machine Learning with Scikit-Learn and TensorFlow》这本书大家应该都知道，作者
Aurélien Géron 发过一条 twitter，画出了机器学习论文中涉及的深度学习工具使用趋势：

![poSe14](https://images.gitbook.cn/poSe14.jpg)

**从图中可以看到：**

  * Caffe, Theano，Torch 的趋势下降，
  * MXNet, CNTK ，Chainer 依然很低，
  * TensorFlow 和 Keras 上升很快， 因为大多数 Keras 用户使用 TensorFlow 作为后端，所以 TensorFlow 的曲线可能会更高。
  * 不过，这同时 PyTorch 也在上升，PyTorch 的语言也很简洁，搭建模型方便。

**而且 TensorFlow 1.x 存在一些不便之处：**

首先，它的代码比较麻烦，有 placeholder，要定义 variable 和 op，还要放在 session 里面运行。 其次，不容易
debug，当有很多错误操作时，调试起来会很棘手。 而且 graph 的模式很难描述，这一点通过后来的 Eager 模式改善了。

**为了让 TensorFlow 更方便易用，2.0 中做了很多改进：**

  1. Eager Execution 成为核心功能，不需要启用 Eager 模式，是默认的，不过 2.0 可以同时支持 eager execution 和 graph mode，所以具有二者的优点。
  2. TF 2.0 更加面向对象，主要依赖 Keras API，代码风格也是 Keras 风格，共享变量也更容易。
  3. 弃用 collections，代码更清晰。
  4. 删除杂乱无章的 API，很多重复的 API 或者不推荐使用的 API 弃用，使用 tf.Keras 层，很多函数如 optimizer，loss，metrics 被统合到 Keras 中；tf.contrib 的各种项目也已经被合并到 Keras 等核心 API 中，或者移动到单独的项目中，还有一些被删除。

2.0 版本的代码和 Keras，PyTorch 一样简洁，这样新手也可以很方便地直接用 TensorFlow 入门深度学习了。
下面让我们通过代码来具体看一下。

* * *

### 2\. Keras 和 2.0 的对比

TensorFlow 几年前就采用了 Keras 作为它的高级 API 之一，然后不断将 Keras 进行了优化，现在称为 **TF Keras。**
我们前面有一篇文章也已经介绍了 Keras 的用法，知道 Keras
真的是非常简单易学，它里面有很多常用的神经网络层，损失函数，优化器等模型算法，可以很轻松地就完成模型的设计和改进。

不过 TF Keras 比较适合小型问题，但在实际生产中问题规模都很大，所以 TensorFlow 设计了 Estimator
估算器，估算器可以从零开始构建到大规模分布，在数百台机器上有较强的容错能力。 在 TensorFlow 2.0 中， **估算器的所有功能也都被带入 TF
Keras** ，这样就可以一次性从原型到分布式训练，再到生产服务，简单有效。

**首先让我们看一下 Keras 和 2.0 的对比** : TF 2.0 的代码风格已经变为和 Keras 几乎一样的了，

![zuf9DO](https://images.gitbook.cn/zuf9DO.jpg)

**再看一下 TF 1.13 中 Keras 模型的定义和 TF 2.0 的对比：**

![ZjLTLi](https://images.gitbook.cn/ZjLTLi.jpg)

它们模型定义的代码是一样的，但其实 TF 2.0 中做了很多事情:

在 1.13 中，构建了一个基于图的模型，这个模型会在 hood 下运行。 在 2.0 中，相同的模型不需要进行任何修改就会在 Eager
模式下运行，这样可以更充分地利用 Eager。

**Eager 便于原型设计和调试** ，在 TF Keras 中有一个适用于 Eager 的函数 `run_eagerly`，它可以帮助追踪模型，使模型在
Eager 模式下一步一步地明确地运行，可以使用 Python 控制流和 Eager 模式进行调试，来优化模型的性能和规模：

![ofUVYq](https://images.gitbook.cn/ofUVYq.jpg)

* * *

### 3\. TF 1.x 和 2.0 的比较

**再来看一下 2.0 和 Eager 模式下的旧版本代码有什么不同：**

![HEEcI4](https://images.gitbook.cn/HEEcI4.jpg)

  1. 不用再麻烦地使用 `tf.placeholders` 和 `feed_dict` 了，在定义函数时，`input` 可以直接作为参数，也不用 `fetches` ，函数的返回值就是我们想要的结果。
  2. `tf.get_variable` 变成 `tf.Variable`。
  3. `tf.contrib` 不再用了，`regularizers` 可以直接从 `tf.keras` 中调用，所以 `tf.losses.get_regularization_loss` 这样的函数也不会再使用。
  4. `tf.Session.run` 不再使用，直接就运行函数就可以。

![NP2ttm](https://images.gitbook.cn/NP2ttm.jpg)

**在建立模型时，**

  1. `tf.layers` 所包含的层函数，是 Keras 风格的，是依赖 `tf.variable_scope` 来定义和重用变量的。 在 TF Keras 中叠加层之前要用到 `tf.keras.Sequential`，新的模型可以自己追踪变量和损失， 而且由 `tf.layers` 到 `tf.keras.layers` 是一对一映射的，层函数只是名称有大写。
  2. 在模型运行时 `training` 参数自动传递给模型的每个层
  3. 旧模型的参数 `x`消失了，因为对象层将模型的构建和调用分开了 另外激活函数不用再通过 `tf.nn` 调用。

**尤其是循环神经网络模型：**

![CUy5OK](https://images.gitbook.cn/CUy5OK.jpg)

在 1.x 中，有好几个不同版本的 LSTM 和 GRU，还需要提前知道设备，以便可以使用 cuDNN 内核获得最佳性能。 在 2.0 中，
**只有一个版本的 LSTM 和一个版本的 GRU 层** ，它们在运行时会自己选择可用的设备，就不用我们自己做了，如果有可用的
GPU，这段代码就会自己运行 cuDNN 内核，如果没有可用的 GPU，就会回到 CPU 操作。

**模型的编译：**

在 1.x 版本中，定义模型后，要单独定义 `loss，accuracy，optimizer`，

    
    
    loss = tf.reduce_mean(input_tensor=losses)
    accuracy = tf.reduce_mean(input_tensor=tf.cast( tf.equal( tf.argmax(input=logits_out, axis=1), tf.cast(y_output, tf.int64) ), tf.float32 ))
    optimizer = tf.train.RMSPropOptimizer(learning_rate)
    

在 2.0 中，可以直接都放在 `model.compile` 中。

    
    
    model.compile(optimizer='adam',
                  loss='sparse_categorical_crossentropy',
                  metrics=['accuracy'])
    

优化器合并在一个集合中，它们可以在 Eager 模式下进出，可以在一台机器上或者多台机器上分布式运行，可以检索和设置常规的超参数。
损失函数也都合并到一个集合中了，其中包含许多常用的内置函数，还有一个界面可以自己定制。 度量的集合包含了所有以前的 TF 的度量标准和 Keras
度量标准，还可以轻松自定义。

**模型的训练：**

1.x 中要用 `sess.run` 来运行，`train_step` 是让 `optimizer` 最小化损失函数 loss：

    
    
    train_step = optimizer.minimize(loss)
    …
    sess.run(train_step, feed_dict=train_dict)
    train_dict = {x_data: x_train_batch, y_output: y_train_batch, dropout_keep_prob:0.5}
    

2.0 中最简单的是用 `model.fit`:

    
    
    model.fit(train_data, epochs=NUM_EPOCHS)
    

**模型的评估：**

1.x 中要写好几行代码来打印损失和准确率，

    
    
    temp_train_loss, temp_train_acc = sess.run([loss, accuracy], feed_dict=train_dict)
    train_loss.append(temp_train_loss)
    train_accuracy.append(temp_train_acc)
    

2.0 中可以直接用 `model.evaluate`:

    
    
    loss, acc = model.evaluate(test_data)
    

之前我们还经常会有 epoch 的循环训练模型， 在 2.0 中也有更简单的实现方法： 直接用
`tf.keras.model.train_on_batch` 就可以对批次数据进行训练， 还有测试集的
`tf.keras.model.test_on_batch` 和评估 `tf.keras.Model.evaluate`。

    
    
    metrics_names = model.metrics_names
    for epoch in range(NUM_EPOCHS):
      #Reset the metric accumulators
      model.reset_metrics()
    for image_batch, label_batch in train_data:
        result = model.train_on_batch(image_batch, label_batch)
        print("train: ",
              "{}: {:.3f}".format(metrics_names[0], result[0]),
              "{}: {:.3f}".format(metrics_names[1], result[1]))
    
      for image_batch, label_batch in test_data:
        result = model.test_on_batch(image_batch, label_batch, 
                                     # return accumulated metrics
                                     reset_metrics=False)
      print("\neval: ",
            "{}: {:.3f}".format(metrics_names[0], result[0]),
            "{}: {:.3f}".format(metrics_names[1], result[1]))
    

**另外，TensorBoard 也可以轻松地和 TF Keras 一起使用** ，只需要一行代码将 TensorBoard 的 callback
加入到模型：

![8IswPO](https://images.gitbook.cn/8IswPO.jpg)

在 TensorBoard 中可以看到准确率和损失的变化，还有模型的概念图，这个 callback
还包括了模型的完整分析，可以更好地了解模型的性能，更容易找到最小化瓶颈的方法。

![2ub5c5](https://images.gitbook.cn/2ub5c5.jpg)

**此外，还有一个`tf.distribute.Strategy API` 用来分布训练工作流**，在 TF
Keras，只需要几行代码就可以添加或切换分发策略，这些 API 易于使用，通用性也不错，适用于许多不同的分发体系结构和设备：

![m3mL2K](https://images.gitbook.cn/m3mL2K.jpg)

**模型训练好后，需要将其保存以便用于生产，或者手机端，或是其他编程语言** ，现在用 TF Keras 可以直接导出模型到
`saved_model`，这样可以直接用于 TF Serving，TF Lite 等等，当然也可以重新加载模型继续训练。

![aeOQcl](https://images.gitbook.cn/aeOQcl.jpg)

![DcWq7h](https://images.gitbook.cn/DcWq7h.jpg)

* * *

上面从建立模型到可视化的各个步骤对两个版本进行了比较，那么手里的代码要如何转换为 2.0 版本呢。

### 4\. 如何将旧代码升级：自动更新

我们可以选择不进行转换，旧代码也可以通过下面的设定而继续正常运行，不过就无法使用 2.0 所具有的一些特性了，比如代码简单，容易维护等。

    
    
    import tensorflow.compat.v1 as tf
    tf.disable_v2_behavior()
    

官网提供了一个脚本用来自动将旧代码进行转换， https://www.tensorflow.org/alpha/guide/upgrade

**首先，安装 2.0 Alpha 版：**

    
    
    !pip install tensorflow==2.0.0-alpha0
    

**然后执行下面这段代码：**

    
    
    !tf_upgrade_v2 --infile old.py --outfile new.py
    

会打印出都做了哪些更改，并且输出一个报告 `report.txt`:

![rHgbVr](https://images.gitbook.cn/rHgbVr.jpg)

不过转换后的代码也并不算是 2.0 版本的风格，它只是通过 `tf.compat.v1` 来使用 placeholders, sessions,
collections 和其他 1.x 风格的函数，所以我们还要进一步转换。

* * *

### 5\. 如何将旧代码升级：深度学习常用技术 1.x 到 2.0 的转换

在前面我们用一篇文章介绍了深度学习中的一些主要概念，今天我们将 **用 TensorFlow 2.0 Alpha 将这些概念加以运用，并且和
TensorFlow 1.x 的代码进行对比。**

首先为了保证运行在 TensorFlow 2.0 上面，要提前安装：

    
    
    !pip install tensorflow==2.0.0-alpha0
    

查看 TensorFlow 版本：

    
    
    import tensorflow as tf
    print("TensorFlow:{}".format(tf.__version__))
    

**1\. 初始化**

![LNtJ4n](https://images.gitbook.cn/LNtJ4n.jpg)

在 2.0 版本中API的命名更规范了，我们可以用 `[name for name in dir(keras.initializers) if not
name.startswith("_")]` 来查看都有哪些初始化算法。

**2\. 激活函数**

![TFVjA6](https://images.gitbook.cn/TFVjA6.jpg)

![9JLnxF](https://images.gitbook.cn/9JLnxF.jpg)

同样，我们可以用 `[m for m in dir(keras.activations) if not m.startswith("_")]`
来查看都有哪些激活函数。

2.0 中在模型中应用激活函数也变得非常简单： 模型 model 的定义更简洁， loss，optimizer，metrics 的定义只需要在
`model.compile` 里面指定， 要打印训练集，验证集的损失和准确率只需要用 `model.fit` 完成

**3\. Batch Normalization**

![CnGIY6](https://images.gitbook.cn/CnGIY6.jpg)

BN 的应用只需要在每一个层后面加上 `keras.layers.BatchNormalization()`， 有时在激活函数前面应用，效果会比较好，
而且在 BN 前面的层不需要 bias，因此可以设置 `use_bias=False`，来减少参数的冗余。

**4\. Gradient Clipping**

![92myKs](https://images.gitbook.cn/92myKs.jpg)

在 1.x 中 gradient clipping 需要几行代码完成，在 2.0 只需要在 `optimizer` 加上参数 `clipvalue` 或者
`clipnorm` 都可以。

**5\. Learning rate**

![MwMgoR](https://images.gitbook.cn/MwMgoR.jpg)

例如我们要采用的是 `Exponential Scheduling：lr = lr0 * 0.1**(epoch / s)`

1.x 中的学习率是在 `training_op` 中设置，定义了三个参数后，再调用 `tf.train.exponential_decay`
应用指数下降策略。 2.0 中要调用 `keras.callbacks.LearningRateScheduler`，将
`exponential_decay_fn` 指数下降策略的函数传递进去。

**6\. L1， L2 regularization**

![TG25tV](https://images.gitbook.cn/TG25tV.jpg)

1.x 中用 `tf.contrib.layers.l1_regularizer(scale)` 给模型加入正则项， 2 中相应的是用
`keras.regularizers.l1(0.01)` 来实现。

**7\. Dropout**

![YkKLbM](https://images.gitbook.cn/YkKLbM.jpg)

Dropout 的代码也变得更简洁， 2.0 中只需要在需要加的层后面用 `keras.layers.Dropout()`。

* * *

在上面我们比较了 Keras 和 TF 2.0，1.x 和 2.0，还从宏观步骤到具体技术列出了 1.x 的旧代码要如何对应到 TF 2.0，这样我们可以在
1 的基础上将代码转化为 2 的风格，也可以对 2 有更直观的理解，可以比较顺畅地用 TF 2.0 来解决实践问题。

