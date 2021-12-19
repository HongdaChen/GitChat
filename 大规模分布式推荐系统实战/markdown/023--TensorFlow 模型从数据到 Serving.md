上一篇熟悉了 TensorFlow 建模流程以及对应的 API
使用方式，并在此基础上实现了一个玩具模型。但是想要真正的在实际应用中生成一个效果不错的模型并将其上线，还有很多很多工作要做。这正是本节将要介绍的重点。

首先会简单介绍两个常用的模型：Wide and Deep$^1$ 和 DIN$^2$，然后就详细讲述一个完整的数据生成到模型上线的 Pipeline。

  * Spark 生成训练/测试数据
  * TensorFlow 建模、训练
  * 模型导出
  * TensorFlow Serving 自动加载模型、模型预测

话不多说，我们先从网络结构说起。

本篇重点在代码实现，只会简单介绍一下模型网络结构，想要深入了解模型背后理论的读者建议熟读模型对应的论文。

软件环境：

  * Spark：2.4.0
  * Java：1.8.0
  * Maven：3.6.3
  * TensorFlow：1.15
  * Docker：18.09.6

### 网络结构

#### **Wide and Deep**

Wide and Deep 是谷歌在 2015
年发表的一篇具有深远影响的论文。论文表示深度模型的泛化性特别强、线性模型的记忆性特别强，何不取二者之长，整合为一个模型，同时发挥两者的优点，于是就诞生了下面这张图里的网络结构。

![](https://images.gitbook.cn/1985b7c0-662c-11eb-a841-9166c06258f2)

左半边是 Deep 模型，右半边是 Wide 模型，一般 Wide 模型的输入是部分特征的两两交叉，便于模型的记忆以及引入一定程度的非线性。

Wide and Deep 模型作为谷歌官方的作品，被集成到 TensorFlow API 中，实现该模型也就变得特别的简单，所以本节就不再赘述了。

TensorFlow 官方对于 Wide and Deep 的实现——DNNLinearCombinedClassifier。

#### **Deep Interest Network**

DIN 是 2018 年阿里发表的一篇点击率预估的论文，极具创意，且工程性极强。后续阿里在此基础上又发表了基于 RNN 的 DIEN$^3$ 和 基于
Transformer 的 BST$^4$，但是影响力远不如 DIN，在实际应用中因为结构稍显复杂使得调参难度变大。目前地鼠商城很多场景都使用 DIN
提供服务，效果也特别的好。

在认识 DIN 网络结构之前，先说以一个小概念——目标物品（ **Target Item** ）。

![](https://images.gitbook.cn/378465f0-662c-11eb-b925-810e10ceddc3)

如上图所示，假设用户此刻停留在此页，此刻之前一段时间内用户的足迹中有物品 **[U, V, W, X, Y, Z]** 。那么此刻展示在用户眼前的物品
**[A, B, C, D]** 被称为目标物品，用户的足迹被称为历史行为序列。DIN
正是学习历史行为序列与目标物品之间的关系，从而通过历史行为序列来计算用户对目标的点击概率，比如用户的历史行为序列中都是手机，目标物品中有手机和鞋子，那么模型很可能将手机排在鞋子前面。

现在可以来看一下 DIN 的网络结构了，如下图所示。

![](https://images.gitbook.cn/43d50440-662c-11eb-9c08-ab1c72a43d60)

可以看到 DIN 与一般深度模型最大的不同点在于：并不是将用户行为序列直接输入模型，而是先利用 attention
机制学习到用户的历史行为与当前物品的关系，通过 **权重** 来表征这种关系，然后将行为序列中的各个物品 embedding 乘以各自的 **权重**
后求和，也就是加权求和，将得到的新的 embedding 与其他特征连接（concat）起来一并送入模型进行训练。

DIN 是将业务与算法结合特别好的模型之一，它直接对 **用户当前行为受历史行为的影响有多大**
进行建模，不仅很符合人的直觉，而且也非常的科学，值得我们仔细研究，不断学习。

> 强烈推荐多读几遍 DIN 的论文，文中有很多特别巧的处理，几乎每次阅读都可能会有新的发现。

弄清楚 Wide and Deep 和 DIN 结构之后，接下来将重心转向工程实现，这几乎是令每个算法工程师最激动的时刻了，话不多说，先看看如何为
TensorFlow 生成 T 级，甚至 P 级的训练/验证数据。

### TFRecord

TFRecord 是官方推荐的专门用于存储 TensorFlow 数据的文件格式，不仅可以存储文本文件，还可以存储视频、语音、图片等数据。

tf.Example 是 TFRecord 文件中存储的具体数据，本质上来说就是一个 `{feature_name: feature_value}`
的映射。feature_name 是字符串类型，feature_value 是 tf.train.Feature 类型，其中可以存储各种数据类型。

对 TFRecord 和 tf.Example
感兴趣的读者可以查看[官方文档](https://tensorflow.google.cn/tutorials/load_data/tfrecord)，特别的详细，这里就不再赘述啦。

一般深度学习训练数据的大小动辄 T 级，很多教程里只给出了单机生成 TFRecord 的方法，这怎么可能满足地鼠商城这种国际超级大电商呀，于是
TensorFlow 官方贴心的提供了使用 Spark 生成 TFRecord 的工具，从此再也不怕原始数据体积过大了，处理海量数据正是 Spark
的强项。

这个工具的名字叫 [spark-tensorflow-
connector](https://github.com/tensorflow/ecosystem/tree/master/spark/spark-
tensorflow-connector)。

#### **spark-tensorflow-connector**

该工具源码是用 scala 编写的，因此想要使用它，需要先将所有代码打成 jar 形式。动手能力强的读者请按照官方 GIT
自行打包，我在这里提供了一个直接可用的 jar，地址在[这里](https://github.com/recsys-4-everyone/dist-
recsys/blob/main/chapter%2023/din/spark/spark-tensorflow-
connector_2.11-1.15.0.jar)。

既然工具已经有了，我们来看看怎么利用它。

#### **训练数据**

有此利器，首先就来生成 DIN 模型的训练集，特征如下所示，涵盖了用户、上下文、商品和历史行为特征：

名称 | 格式 | 示例 | 备注  
---|---|---|---  
user_id | 字符串 | "uid012" | 用户 ID  
age | 整型 | 18 | 年龄  
gender | 字符串 | "0" | 取值 "0", "1", "未知"  
device | 字符串 | "huawei p40pro max" | 终端设备型号  
item_id | 字符串 | "item012" | 物品 ID  
clicked_items_15d | 字符串列表 | ["item012", "item345"] | 15 天内点击的物品  
  
想要将这样的数据集存储成 TFRecord，Spark 代码应该如何编写呢？

    
    
    # 这里因为要将数据保存在本地，所以 master 指定为 local, 同时指定 jars. 
    # 启动命令: spark-submit data.py --master local --jars spark-tensorflow-connector_2.11-1.15.0
    from pyspark.sql.types import *
    from pyspark.sql import SparkSession
    
    spark = SparkSession.builder.appName('din_train_data').getOrCreate()
    
    # 保存在本地，可以换成 HDFS、S3 等分布式存储路径
    path = "file:///tmp/din_train_data"
    
    # 指定各特征类型
    feature_names = [
                     StructField("label", LongType()),
                     StructField("user_id", StringType()),
                     StructField("age", IntegerType()),
                     StructField("gender", StringType()),
                     StructField("item_id", StringType()),
                     StructField("clicked_items_15d", ArrayType(StringType(), True))]
    
    schema = StructType(feature_names)
    test_rows = [
        [1, "user_id1", 22, "0", "item_id1", ["item_id2", "item_id3", "item_id4"]],
        [0, "user_id2", 33, "1", "item_id5", ["item_id6", "item_id7"]]
    ]
    rdd = spark.sparkContext.parallelize(test_rows)
    df = spark.createDataFrame(rdd, schema)
    
    # 存储为 tfrecord 文件格式，文件内部的数据格式为 Example
    df.write.format("tfrecords").option("recordType", "Example").save(path, mode="overwrite")
    
    df = spark.read.format("tfrecords").option("recordType", "Example").load(path)
    df.show()
    
    # 打印 dataframe 结构
    df.printSchema()
    # root
    #  |-- item_id: string (nullable = true)
    #  |-- age: long (nullable = true)
    #  |-- gender: string (nullable = true)
    #  |-- clicked_items_15d: array (nullable = true)
    #  |    |-- element: string (containsNull = true)
    #  |-- label: long (nullable = true)
    #  |-- user_id: string (nullable = true)
    

同理，根据相同的逻辑，可以生成验证集。

有了数据集了，现在可以搭建模型结构了，这里即将介绍数据的读取、模型的搭建、模型的导出以及 TensorFlow Serving 加载模型实现实时预测。

### DIN 模型

使用 TensorFlow estimator API 进行模型的搭建和导出。

#### **读数据**

读数据分三步走。

**第一步** ：定义每个特征的格式和类型，与生成 TFRecord 时的格式和类型要一一对应。

FixLenFeature、VarLenFeature 等 TF API，读者可以查阅官方文档，解释的特别详细。

    
    
    def get_example_fmt():
        example_fmt = {}
    
        example_fmt['label'] = tf.FixedLenFeature([], tf.int64)
        example_fmt['user_id'] = tf.FixedLenFeature([], tf.string)
        example_fmt['age'] = tf.FixedLenFeature([], tf.int64)
        example_fmt['gender'] = tf.FixedLenFeature([], tf.string)
        example_fmt['item_id'] = tf.FixedLenFeature([], tf.string)
        # 此特征长度不固定
        example_fmt['clicked_items_15d'] = tf.VarLenFeature(tf.string)
    
        return example_fmt
    

**第二步** ：定义解析函数，输入为一条数据，按照第一步定义的特征信息进行数据的解析，并将一些特征的稀疏表示转成稠密表示。

    
    
    def parse_fn(example):
        example_fmt = get_example_fmt()
        parsed = tf.parse_single_example(example, example_fmt)
        # VarLenFeature 解析的特征是 Sparse 的，需要转成 Dense 便于操作
        parsed['clicked_items_15d'] = tf.sparse.to_dense(parsed['clicked_items_15d'], '0')
        label = parsed.pop('label')
        features = parsed
        return features, label
    

**第三步** ：定义读数据函数，输入为若干 TFRecord 文件，每个文件中的每一条数据经过第二步的解析函数，完成整个训练数据的解析。

首先介绍一下 pad（补齐）。因为 TensorFlow 是按批读数据的，一个批次内的数据可能形状会不一致，但是训练时要求形状一致，所以需要进行 pad
操作。比如 clicked_items_15d 这个特征，每条数据中它的长度可能都不一样，就需要将每条数据中该特征 pad 到当前批次中最长的长度。

举个例子，假设数据中有 item_id 和 clicked_items_15d 两个特征，一个批次两条数据，如下所示：

    
    
    [item_id1, [click_id1, click_id2]]
    
    [item_id2, [click_id3, click_id4, click_id5]]
    

则 TensorFlow 需要将此数据 pad 成如下形式：

    
    
    [item_id1, [click_id1, click_id2， '0']] 
    
    [item_id2, [click_id3, click_id4, click_id5]]
    
    
    
    # pad 返回的数据格式与形状必须与 parse_fn 的返回值完全一致。
    def padded_shapes_and_padding_values():
        example_fmt = get_example_fmt()
    
        padded_shapes = {}
        padding_values = {}
    
        for f_name, f_fmt in example_fmt.items():
            if 'label' == f_name:
                continue
            if isinstance(f_fmt, tf.FixedLenFeature):
                padded_shapes[f_name] = []
            elif isinstance(f_fmt, tf.VarLenFeature):
                padded_shapes[f_name] = [None]
            else:
                raise NotImplementedError('feature {} feature type error.'.format(f_name))
    
            if f_fmt.dtype == tf.string:
                value = '0'
            elif f_fmt.dtype == tf.int64:
                value = 0
            elif f_fmt.dtype == tf.float32:
                value = 0.0
            else:
                raise NotImplementedError('feature {} data type error.'.format(f_name))
    
            padding_values[f_name] = tf.constant(value, dtype=f_fmt.dtype)
    
        padded_shapes = (padded_shapes, [])
        padding_values = (padding_values, tf.constant(0, tf.int64))
        return padded_shapes, padding_values
    

pad 完成后，就可以读数据了。

    
    
    def input_fn(mode, pattern, epochs=1, batch_size=512, num_parallel_calls=1):
        padded_shapes, padding_values = padded_shapes_and_padding_values()
        files = tf.data.Dataset.list_files(pattern)
        data_set = files.apply(
            tf.data.experimental.parallel_interleave(
                tf.data.TFRecordDataset,
                cycle_length=8,
                sloppy=True
            )
        )  # 1
        data_set = data_set.apply(tf.data.experimental.ignore_errors())
        data_set = data_set.map(map_func=parse_fn,
                                num_parallel_calls=num_parallel_calls)  # 2
    
        if mode == 'train':
            data_set = data_set.shuffle(buffer_size=10000)  # 3.1
            data_set = data_set.repeat(epochs)  # 3.2
        data_set = data_set.padded_batch(batch_size,
                                         padded_shapes=padded_shapes,
                                         padding_values=padding_values)
    
        data_set = data_set.prefetch(buffer_size=1)  # 4
        return data_set
    

这里需要注意、特别影响数据读取速度的地方：

1\. parallel_interleave 并行读取文件，其中的 sloppy 建议设为 True，表示对元素的顺序没有要求，可以提高读取性能。

2\. map 函数的 num_parallel_calls 强烈建议设置为当前机器可用 CPU 核数，显著提高读取性能。

3\. shuffle 和 repeat 的顺序也需要注意，一般 shuffle 在前，repeat 在后，这样可以保证一个 epoch
结束后所有数据都能够被模型看到，如果 shuffle 在后，repeat 在前，有些数据可能很多 epochs 后都没有被看到。

比如，数据为 $[1,2,3]$，repeat 设为 2，先 shuffle 后 repeat 可能会得到这样的数据：$[1,3,2,2,3,1]$。如果先
repeat 后 shuffle，则可能得到这样的数据：$[1,2,1,2,3,3]$。

这里还要注意，shuffle 函数中的 buffer size 对内存的影响特别大，因为它要把数据缓存在内存中进行 shuffle，不能设置的太大了。

4\. prefetch 也是对性能有显著提升的，TensorFlow 会提前拉取数据，这样会节省训练时等待数据的时间，建议放在数据流的最后。

#### **搭结构**

Tensorflow estimator API 需要实现具有如下签名的函数，用来定义模型结构。

    
    
    def model_fn(features, labels, mode, params)
    """
    Args:
      features: 传入的特征，即 parse_fn 返回值的第一项
      labels: 传入的 label，即 parse_fn 返回值的第二项
      mode: 用来标识 训练/验证/推理 三个阶段
          1. 训练时其值为 train
          2. 验证时其值为 eval
          3. 导出模型线上服务时其值为 infer
      params: 传入的一些超参和配置，比如 learning rate 等
    """
    

也就是说 TensorFlow 已经定义好接口规范，需要我们来实现。具体实现如下：

    
    
    # -*- coding: utf-8 -*-
    import math
    import tensorflow as tf
    from tensorflow import feature_column as fc
    
    ##### feature columns #####
    def embedding_size(category_num):
        return int(2 ** math.ceil(math.log2(category_num ** 0.25)))
    
    
    # user_id
    user_id = fc.categorical_column_with_hash_bucket("user_id",
                                                     hash_bucket_size=10000)
    user_id = fc.embedding_column(user_id,
                                  dimension=embedding_size(10000))
    
    # age
    raw_age = fc.numeric_column('age',
                                default_value=0,
                                dtype=tf.int64)
    
    boundaries = [18, 25, 36, 45, 55, 65, 80]
    bucketized_age = fc.bucketized_column(source_column=raw_age,
                                          boundaries=boundaries)
    
    age = fc.embedding_column(bucketized_age,
                              dimension=embedding_size(len(boundaries) + 1))
    
    # gender
    gender = fc.categorical_column_with_hash_bucket('gender',
                                                    hash_bucket_size=100)
    gender = fc.embedding_column(gender,
                                 dimension=embedding_size(100))
    ##### feature columns end #####
    
    def model_fn(features, labels, mode, params):
        init_learning_rate = params['learning_rate']
        decay_steps = params['decay_steps']
        decay_rate = params['decay_rate']
    
        with tf.name_scope('user'):
            # shape: B (batch size)
            user_embedding = fc.input_layer(features, [user_id, age, gender])
    
        # 需要生成 item_id 对应的 embedding matrix 方便 embedding 共享
        with tf.name_scope('item'):
            item_bucktes = 100
            item_id = features['item_id']
            item_id = tf.reshape(item_id, [-1, 1])
            list_size = tf.shape(item_id)[0] # 1
            item_id = tf.string_to_hash_bucket_fast(item_id, num_buckets=item_bucktes)
            item_matrix = tf.get_variable(name='item_matrix',
                                          shape=(100, 16),
                                          initializer=tf.initializers.glorot_uniform())
    
            item_embedding = tf.nn.embedding_lookup(item_matrix,
                                                    item_id,
                                                    name='item_embedding')
            item_embedding = tf.squeeze(item_embedding, axis=1)
    
        with tf.name_scope('history'):
            # shape: B * T (sequence length)
            clicked_items = features['clicked_items_15d']
            # mask 用来标记 clicked_item 中哪些物品 id 不是 pad 上去的
            clicked_mask = tf.cast(tf.not_equal(clicked_items, '0'), tf.bool)
            clicked_items = tf.string_to_hash_bucket_fast(clicked_items, num_buckets=item_bucktes)
            # shape: B * T * E
            # clicked embedding 与 item embedding 共享 item matrix.
            clicked_embedding = tf.nn.embedding_lookup(item_matrix,
                                                       clicked_items,
                                                       name='clicked_embedding')
    
        if mode == tf.estimator.ModeKeys.PREDICT: # 2
            user_embedding = tf.tile(user_embedding, [list_size, 1])
            clicked_embedding = tf.tile(clicked_embedding, [list_size, 1, 1])
            clicked_mask = tf.tile(clicked_mask, [list_size, 1])
    
        # shape: B * E
        clicked_attention = attention(clicked_embedding,
                                      item_embedding,
                                      clicked_mask,
                                      [16, 8],
                                      'clicked_attention')
    
        # 对照着 DIN 的网络结构图，fc_inputs 即为全连接层的输入
        fc_inputs = tf.concat([user_embedding, item_embedding, clicked_attention], axis=-1, name='fc_inputs')
    
        with tf.name_scope('predictions'):
            logits = fc_layers(mode, net=fc_inputs, hidden_units=[64, 16, 1], dropout=0.3)
            predictions = tf.sigmoid(logits, name='predictions')
    
            if mode != tf.estimator.ModeKeys.PREDICT:
                labels = tf.reshape(labels, [-1, 1])
                loss = tf.losses.sigmoid_cross_entropy(labels, logits)
                if mode == tf.estimator.ModeKeys.EVAL:
                    metrics = {
                        'auc': tf.metrics.auc(labels=labels,
                                              predictions=predictions,
                                              num_thresholds=500)
                    }
                    for metric_name, op in metrics.items():
                        tf.summary.scalar(metric_name, op[1])
                    return tf.estimator.EstimatorSpec(mode, loss=loss,
                                                      eval_metric_ops=metrics)
                else:
                    global_step = tf.train.get_global_step()
                    learning_rate = exponential_decay(global_step, init_learning_rate, decay_steps, decay_rate)
                    optimizer = tf.train.AdagradOptimizer(learning_rate=learning_rate)
                    tf.summary.scalar('learning_rate', learning_rate)
                    train_op = optimizer.minimize(loss=loss, global_step=global_step)
                    return tf.estimator.EstimatorSpec(mode, loss=loss, train_op=train_op)
            else:
                predictions = {
                    'probability': tf.reshape(predictions, [1, -1])
                }
                export_outputs = {
                    'predictions': tf.estimator.export.PredictOutput(predictions)
                }
                return tf.estimator.EstimatorSpec(mode, predictions=predictions,
                                                  export_outputs=export_outputs)
    
    
    # attention 机制，history 为用户历史行为序列，target 为候选物品
    def attention(history, target, history_mask, hidden_units, name='attention_out'):
        keys_length = tf.shape(history)[1]
        target_emb_size = target.get_shape()[-1]
        target_emb_tmp = tf.tile(target, [1, keys_length])
        target = tf.reshape(target_emb_tmp, shape=[-1, keys_length, target_emb_size])
        net = tf.concat([history, history - target, target, history * target], axis=-1)
        for units in hidden_units:
            net = tf.layers.dense(net, units=units, activation=tf.nn.relu)
        attention_weight = tf.layers.dense(net, units=1, activation=None)
        scores = tf.transpose(attention_weight, [0, 2, 1])
        history_masks = tf.expand_dims(history_mask, axis=1)
        padding = tf.zeros_like(scores)
        scores = tf.where(history_masks, scores, padding)
        outputs = tf.matmul(scores, history)
        outputs = tf.reduce_sum(outputs, 1, name=name)
        return outputs
    
    
    def fc_layers(mode, net, hidden_units, dropout=0.0, activation=None, name='fc_layers'):
        layers = len(hidden_units)
        for i in range(layers - 1):
            num = hidden_units[i]
            net = tf.layers.dense(net, units=num, activation=tf.nn.relu,
                                  kernel_initializer=tf.initializers.he_uniform(),
                                  name=name + '_hidden_{}'.format(str(num) + '_' + str(i)))
            net = tf.layers.dropout(inputs=net, rate=dropout, training=mode == tf.estimator.ModeKeys.TRAIN)
        num = hidden_units[layers - 1]
        net = tf.layers.dense(net, units=num, activation=activation,
                              kernel_initializer=tf.initializers.glorot_uniform(),
                              name=name + '_hidden_{}'.format(str(num)))
        return net
    
    
    def exponential_decay(global_step, learning_rate=0.1, decay_steps=10000, decay_rate=0.9):
        return tf.train.exponential_decay(learning_rate=learning_rate,
                                          global_step=global_step,
                                          decay_steps=decay_steps,
                                          decay_rate=decay_rate,
                                          staircase=False)
    

几个注释需要做些说明：

  1. list_size 训练时表示 batch size，预测时表示物品个数。
  2. 为这么这里需要 tile 呢，这是因为在预测时，一般物品个数为 N，但是用户特征一般只有一条（因为预测时肯定是当前用户的特征，一般只有一条）。为了提高性能，节约不必要的网络开销（后面我们会看到为什么会有网络开销）。模型服务时 用户特征只传一条给模型，物品特征还是 N 条，所以需要在模型中将用户特征复制 N 份，这样后面的 concat 操作才不会报形状不一致的错。

#### **模型训练**

到这里就是要让模型跑起来，代码逻辑也很简单，就不再赘述了。

    
    
    # -*- coding: utf-8 -*-
    
    import os
    from data import input_fn
    from estimator import model_fn
    import tensorflow as tf
    
    
    def main(_):
        cpu = os.cpu_count()
        session_config = tf.ConfigProto(
            device_count={'GPU': 0,
                          'CPU': cpu},
            inter_op_parallelism_threads=cpu // 2,
            intra_op_parallelism_threads=cpu // 2,
            device_filters=[],
            allow_soft_placement=True)
    
        run_config = tf.estimator.RunConfig(**{
            'save_summary_steps': 100,
            'save_checkpoints_steps': 1000,
            'keep_checkpoint_max': 10,
            'log_step_count_steps': 100
        }).replace(session_config=session_config)
    
        model = tf.estimator.Estimator(
            model_fn=model_fn,
            model_dir='/home/axing/din/checkpoints/din',
            config=run_config,
            params={
                'decay_rate': 0.9,
                'decay_steps': 10000,
                'learning_rate': 0.1
            }
        )
    
        train_spec = tf.estimator.TrainSpec(
            input_fn=lambda: input_fn(mode='train', pattern='/home/axing/din/dataset/*'))
    
        # 这里就假设验证集和训练集地址一样了，实际应用中是肯定不一样的。
        eval_spec = tf.estimator.EvalSpec(
            input_fn=lambda: input_fn(mode='eval', pattern='/home/axing/din/dataset/*'),
            steps=100, # 每次验证 100 个 batch size 的数据
            throttle_secs=60 # 每隔至少 60 秒验证一次
        )
        tf.estimator.train_and_evaluate(model, train_spec, eval_spec)
    
    
    if __name__ == '__main__':
        # run: python main.py
        tf.logging.set_verbosity(tf.logging.INFO)
        tf.app.run(main=main)
    

#### **模型导出**

模型导出，供线上服务使用。

    
    
    # -*- coding: utf-8 -*-
    import tensorflow as tf
    from estimator import model_fn
    
    
    def export_model():
        def serving_input_receiver_fn():
            receiver_tensors = \
                {'user_id': tf.placeholder(dtype=tf.string,
                                           shape=(None, None),
                                           name='user_id'),
                 'age': tf.placeholder(dtype=tf.int64,
                                       shape=(None, None),
                                       name='age'),
                 'gender': tf.placeholder(dtype=tf.string,
                                          shape=(None, None),
                                          name='gender'),
                 'item_id': tf.placeholder(dtype=tf.string,
                                           shape=(None, None),
                                           name='item_id'),
                 'clicked_items_15d': tf.placeholder(dtype=tf.string,
                                                     shape=(None, None),
                                                     name='clicked_items_15d')
                 }
    
            return tf.estimator.export.build_raw_serving_input_receiver_fn(receiver_tensors)
    
        model = tf.estimator.Estimator(
            model_fn=model_fn,
            model_dir='/home/axing/checkpoints/din', # 与
            params={
                'decay_rate': 0.9,
                'decay_steps': 50000,
                'learning_rate': 0.1
            }
        )
    
        model.export_savedmodel('/home/axing/savers/din', serving_input_receiver_fn())
    
    def main(_):
        export_model()
    
    if __name__ == '__main__':
        tf.logging.set_verbosity(tf.logging.INFO)     
        tf.app.run(main=main)
    

#### **TensorFlow Serving**

为了让模型提供线上预测，使用 TensorFlow Serving 来完成这个功能。

    
    
    docker 的安装就不在这里演示了。
    

首先保证机器上有 tensorflow serving 镜像。

    
    
    docker pull tensorflow/serving
    

镜像拉取成功之后，启动容器加载模型。

    
    
    docker run -d -p 8501:8501 --mount type=bind,source=/home/axing/savers/din,target=/models/din -e MODEL_NAME=din -t tensorflow/serving
    

  1. source 是导出模型的地址
  2. target 是映射的容器地址

使用 `docker logs -f container_id` 观察服务是否成功，当看到类似 `Exporting HTTP/REST API
at:localhost:8501 ...` 日志时，表明模型服务成功。

服务启动成功后，即可对其进行访问。

    
    
    curl -X POST http://localhost:8501/v1/models/din:predict \
      -d '{
      "signature_name": "serving_default",
      "instances":[
         {
            "user_id":["a"],
            "item_id":["item01", "item02", "item02"],
            "age": [22],
            "gender": ["1"],
            "clicked_items_15d": ["item01", "item02"]
         }]}'
    

是不是可以看见 3 个预测值？对，就是对应 3 个 item_id 的预测值。现在可以理解为什么我们在建模时需要加 tile
对用户特征进行复制了吗？如果不在模型中复制，就需要在传数据时将用户数据复制，这样增大了不必要的网络开销（不管是 HTTP 调用还是 RPC
调用，都是通过网络传递数据）。

既然有了服务，那么后台的工程师们就编写相应的应用程序来请求模型进行预测了，实现了模型的对外服务。

### 小结

本章节的内容有点多，主要介绍了 DIN 这么一个很经典的模型，从 **数据生成** 到 **模型搭建** 再到 **模型导出** 以及最后的
**模型服务** ，基本上囊括了 TensorFlow 模型开发的完整流程，对于接触不多的读者来说需要慢慢消化。

一般的训练数据是 TFRecord 格式的，灵活好用，对于海量训练数据，采用 spark-tensorflow-connector 工具来生成
TFRecord 文件。

建模基于 estimator 而不是 keras，目前 estimator 在搭建网络结构时比较灵活，尤其在一些复杂的模型搭建时对于 feature
column 的支持也是特别的好。当然 TensorFlow 在 2.0 版本后主推
keras，相信它会越来越好用。不过这些都不重要，理解了模型的原理，搭建结构只不过是一步翻译的工作。

模型训练完毕后，导出模型并通过 TensorFlow Serving 加载并对外提供服务，这一步几乎在 1
分钟之内就可以完成，可见这是有多方便。当然实际应用中单个 Docker 容器是没法应对生产任务的，还需要一个 TensorFlow Serving
集群，这部分的内容就超出了专栏的范围了。

好了，到此就差不多要结束本章节的内容了。希望聪明的读者们举一反三，继续沿用上述流程去实现其他的模型。

下一篇，将目光放在第 21 篇介绍过的 Learning to Rank 上，看看如何用 TensorFlow 实现 ListNet。

咱们下篇见！

  * 注释 1：[Wide and Deep](https://arxiv.org/abs/1606.07792)
  * 注释 2：[Deep Interest Network for Click-Through Rate Prediction](https://arxiv.org/abs/1706.06978)
  * 注释 3：[Deep Interest Evolution Network for Click-Through Rate Prediction](https://arxiv.org/pdf/1809.03672.pdf)
  * 注释 4：[Behavior Sequence Transformer for E-commerce Recommendation in Alibaba](https://arxiv.org/pdf/1905.06874v1.pdf)

