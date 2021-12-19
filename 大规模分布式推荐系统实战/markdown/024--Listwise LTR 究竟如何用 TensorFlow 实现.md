到现在为止，相信大家对于 TensorFlow 的使用已经有了一定的熟练度，它对于特征处理、模型搭建和对外提供服务等等方面都做得特别好，也特别的灵活。上一节
DIN 模型的实现完整地展示了从数据到 Serving 的整个过程，这也是实际应用中的工作流程。

**第 21 篇** 曾经介绍过 Listwise LTR，详细的讲解过如何将它转变为概率模型$^1$，但是因为彼时并没有介绍 TensorFlow
相关用法，所有就没有实现它，这一篇，我们就来看看如何使用 TensorFlow 来实现 Listwise LTR 模型。

建模的流程都是相似的，如下：

  * 数据构造：Listwise 的数据构造与普通的模型数据构造差别很大，一条训练实例对应的是若干条数据。TensorFlow 的 SequenceExample 专门用来存储此类数据。
  * 模型结构：我们采用上一章节介绍的 DIN 模型来作为 LTR 的基础模型。LTR 只是一种思想，可以用任意模型来实现这种思想，模型结构本质上变化特别的微小，最大的差异在于输入数据的维度多了一维（因为一条训练实例是多条数据）。
  * 损失函数：Listwise LTR 的损失函数是需要特别注意的地方，虽然理论很简单（先计算真实值和预测值的 top one probability 分布，然后再根据两个分布计算交叉熵），但是实际编码过沉重很容易就会计算出错误的损失值，并且很难发现这个隐藏颇深的 Bug，要特别的小心，本章节的后面会详细的说明这个问题。
  * 指标：LTR 的指标一般都是 NDCG，但是在实现的时候还是有很多小细节需要注意，不然会算出一个虚假的指标出来，同样地，这个问题我们会在后面特别强调。
  * Serving：Serving 基本上与上一节并无差异。

本篇依然以代码为主，毕竟 talk is cheap。

本篇中很多概念沿用第 21 篇的内容，所以一定要先掌握第 21 篇的内容，再来看本篇的内容。

软件环境：

  * Spark：2.4.0
  * Java：1.8.0
  * Maven：3.6.3
  * TensorFlow：1.15
  * Docker：18.09.6

### 训练数据

既然是 Listwise，学习 List 中样本的顺序关系，那么就要先定义 List 怎么生成。

一般情况下，以 PV_ID 或者 SESSION_ID 为粒度对原始进行聚合操作得到一个 List。

PV_ID 是页面访问 ID，用户每刷新一次，PV_ID 就会变化，所以可以认为一个 PV_ID 下的数据具有可比性。

SESSION_ID 是会话 ID，一般是打开 APP 到关闭 APP 期间，SESSION_ID 保持不变。

究竟是以 PV_ID 还是 SESSION_ID 为粒度，要根据不同的数据来定，如果使用 PV_ID 为粒度聚合得到的 List 中 relevance
大部分都只有 1 个值，则可以考虑采用 SESSION_ID 聚合。

假设原始数据中有如下字段。

名称 | 格式 | 示例 | 备注  
---|---|---|---  
PV_ID | 字符串 | "pv123" | 非特征，用于聚合数据  
user_id | 字符串 | "uid012" | 用户 ID  
age | 整型 | 18 | 异常值: 999  
gender | 字符串 | "0" | 取值 "0", "1", "未知"  
item_id | 字符串 | "item012" | 物品 ID  
clicked_items_15d | 字符串列表 | ["item012", "item345"] | 15 天内点击的物品  
relevance | 整型 | 0 | 0: 曝光; 1: 点击; 2: 加车; 3: 下单  
  
如果数据集如下所示：

PV_ID | user_id | age | gender | item_id | clicked_items_15d | relevance  
---|---|---|---|---|---|---  
"pv123" | "uid012" | 18 | "0" | "item012" | [] | 1  
"pv123" | "uid012" | 18 | "0" | "item345" | ["item012"] | 0  
"pv456" | "uid345" | 25 | "1" | "item456" | [] | 2  
"pv456" | "uid345" | 25 | "1" | "item567" | [item456"] | 1  
"pv456" | "uid345" | 25 | "1" | "item678" | [item456", "item567"] | 0  
  
由数据集可见，用户 uid012 在同一个页面中浏览了 item012 和 item345 两个物品并对 item012 发生了点击。同理，用户
uid345 在同一个页面浏览了 item456、item567 和 item678 三个物品并对 item456 进行了加车，对 item567
进行了点击。注意到特征 clicked _items_ 15d 实时发生变化。

对数据进行聚合时：

  1. 用户画像特征在同一个 PV_ID 肯定是一样的，所以存储为单值；
  2. 物品特征在同一个 PV_ID 下聚合为一个数组/列表，所以存储为变长数组；
  3. 用户行为特征基本格式为数组，聚合后为数组的数组，所以存储为二位数组；
  4. relevance/label 在同一个 PV_ID 聚合为一个数组，存储为变长数组。

聚合后，上述特征和 relevance 组成了 Listwise LTR 一条训练实例，TensorFlow 提供了 SequenceExample
专门用来存储这种类型的数据。

训练数据的生成代码如下所示，依然使用 spark-tensorflow-connector 生成 TFRecord 格式的数据。

    
    
    # 这里因为要将数据保存在本地，所以 master 指定为 local，同时指定 jars
    # 启动命令: spark-submit --master local --jars spark-tensorflow-connector_2.11-1.15.0 data.py
    from pyspark.sql.types import *
    from pyspark.sql import SparkSession
    
    spark = SparkSession.builder.appName('ltr_train_data').getOrCreate()
    
    # 保存在本地，可以换成 HDFS、S3 等分布式存储路径
    path = "file:///home/axing/ltr/dataset"
    
    def get_training_data(records):
        """
        records: 同一个 pv_id 下的数据集合
        """
        relevances = [record[6] for record in records]
        # 如果此 pv_id 下全是曝光数据，直接返回
        if not any(relevances):
            return []
        # 用户基本特征在一个 pv_id 下是相同的，存储为单值
        user_id = records[0][1]
        age = records[0][2]
        gender = records[0][3]
    
        # 物品在同一个 pv_id 下聚合为列表/数组
        item_id = [record[4] for record in records]
    
        # 用户行为在同一个 pv_id 下聚合为数组的数组
        clicked_items_15d = [record[5] for record in records]
    
        row = [relevances]
        row.append(user_id)
        row.append(age)
        row.append(gender)
        row.append(item_id)
        row.append(clicked_items_15d)
    
        return row
    
    
    # 指定各特征类型
    feature_names = [
                     StructField("relevance", ArrayType(LongType())),
                     StructField("user_id", StringType()),
                     StructField("age", LongType()),
                     StructField("gender", StringType()),
                     StructField("item_id", ArrayType(StringType())),
                     StructField("clicked_items_15d", ArrayType(ArrayType(StringType())))
    ]
    
    schema = StructType(feature_names)
    test_rows = [
        ["pv123", "uid012", 18, "0", "item012", [], 1],
        ["pv123", "uid012", 18, "0", "item345", ["item012"], 0],
        ["pv456", "uid345", 25, "1", "item456", [], 2],
        ["pv456", "uid345", 25, "1", "item567", ["item456"], 1],
        ["pv456", "uid345", 25, "1", "item678", ["item456", "item567"], 1]
    ]
    rdd = spark.sparkContext.parallelize(test_rows)
    rdd = rdd.keyBy(lambda row: row[0]).groupByKey().mapValues(list)
    rdd = rdd.map(lambda pv_id_and_records: get_training_data(pv_id_and_records[1]))
    df = spark.createDataFrame(rdd, schema)
    
    # 存储为 tfrecord 文件格式，文件内部的数据格式为 SequenceExample
    df.write.format("tfrecords").option("recordType", "SequenceExample").save(path, mode="overwrite")
    
    df = spark.read.format("tfrecords").option("recordType", "SequenceExample").load(path)
    df.show()
    
    # 打印 dataframe 结构
    df.printSchema()
    # root
    #  |-- item_id: array (nullable = true)
    #  |    |-- element: string (containsNull = true)
    #  |-- age: long (nullable = true)
    #  |-- gender: string (nullable = true)
    #  |-- clicked_items_15d: array (nullable = true)
    #  |    |-- element: array (containsNull = true)
    #  |    |    |-- element: string (containsNull = true)
    #  |-- relevance: array (nullable = true)
    #  |    |-- element: long (containsNull = true)
    #  |-- user_id: string (nullable = true)
    

### 基于 DIN 的 Listwise LTR

建模步骤与上一章节没太大差别，从读数据开始。

#### **读数据**

**第一步** ：定义每个特征的格式和类型。

    
    
    def get_example_fmt():
        context_fmt = {}
        sequence_fmt = {}
    
        # 注意每个特征的类型哦
        context_fmt['user_id'] = tf.FixedLenFeature([], tf.string)
        context_fmt['age'] = tf.FixedLenFeature([], tf.int64)
        context_fmt['gender'] = tf.FixedLenFeature([], tf.string)
    
        context_fmt['item_id'] = tf.VarLenFeature(tf.string)
        context_fmt['relevance'] = tf.VarLenFeature(tf.int64)
    
        # 此特征放在了 sequence_fmt 中，表明了它是一个数组的数组
        sequence_fmt['clicked_items_15d'] = tf.VarLenFeature(tf.string)
    
        return context_fmt, sequence_fmt
    

**第二步** ：定义解析函数。

    
    
    def parse_fn(example):
        context_fmt, sequence_fmt = get_example_fmt()
        context, sequence = tf.parse_single_sequence_example(example, context_fmt, sequence_fmt)
    
        parsed = context
        parsed.update(sequence)
        parsed['item_id'] = tf.sparse.to_dense(parsed['item_id'], '0')
        parsed['clicked_items_15d'] = tf.sparse.to_dense(parsed['clicked_items_15d'], '0')
        relevance = parsed.pop('relevance')
        relevance = tf.sparse.to_dense(relevance, -2 ** 32) # 1
        features = parsed
        return features, relevance
    

**第三步** ：定义读数据函数，先定义 pad 再完成数据读取函数。

    
    
    # pad 返回的数据格式与形状必须与 parse_fn 的返回值完全一致。
    def padded_shapes_and_padding_values():
        context_fmt, sequence_fmt = get_example_fmt()
    
        padded_shapes = {}
        padding_values = {}
    
        padded_shapes['user_id'] = []
        padded_shapes['age'] = []
        padded_shapes['gender'] = []
        padded_shapes['item_id'] = [None]
        padded_shapes['clicked_items_15d'] = [None, None]
    
        padding_values['user_id'] = '0'
        padding_values['age'] = tf.constant(0, dtype=tf.int64)
        padding_values['gender'] = '0'
        padding_values['item_id'] = '0'
        padding_values['clicked_items_15d'] = '0'
    
        padded_shapes = (padded_shapes, [None])
        padding_values = (padding_values, -2 ** 32)  # 2
        return padded_shapes, padding_values
    

这里需要特别注意 relevance 的 默认值和 pad 值均为 $-2^{32}$ 而不是
0.0，在搭建网络结构实现损失函数时，再详细说明为什么要这么设置。

    
    
    def input_fn(mode, pattern, epochs=1, batch_size=512, num_parallel_calls=1):
        """
        与上一章节完全一致，故略。
        """
    

#### **搭结构**

依旧需要实现 TensorFlow estimator API 指定的 model_fn 接口。

写代码之前，先要清楚训练时的数据和线上预测时的数据，两者的形状（shape）差别比较大。

训练时，clicked_items_15d 特征的 shape 为 B * L * T (Batch size * List size * Time
steps)。

预测时，clicked_items_15d 特征的 shape 为 1 * T，这一点一定清楚，写代码时做各种维度的 expand 和 tile
就不会感到迷惑了。

预测时，所有跟用户有关的特征的 List size 均没有，因为请求模型的调用者为了减少网络开销，只会将用户特征的第一维度置为 1（即 B =
1），而物品有关的特征，因为物品个数为 list size，所以第一维度为 list size（即 B = list size）。

    
    
    # -*- coding: utf-8 -*-
    import math
    import tensorflow as tf
    from ndcg import metrics_impl
    from tensorflow import feature_column as fc
    
    
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
    
    
    def model_fn(features, labels, mode, params):
        init_learning_rate = params['learning_rate']
        decay_steps = params['decay_steps']
        decay_rate = params['decay_rate']
    
        with tf.name_scope('user'):
            # shape: B (batch size) * concatenated embedding size
            user_embedding = fc.input_layer(features, [user_id, age, gender])
    
        with tf.name_scope('item'):
            item_buckets = 100
            # shape:
            # 训练:B * L (List size)
            # 线上: L, reshape to (1, L)
            item_id = features['item_id']
            if mode == tf.estimator.ModeKeys.PREDICT:
                item_id = tf.reshape(item_id, [1, -1])
            list_size = tf.shape(item_id)[1]
            item_id = tf.string_to_hash_bucket_fast(item_id, num_buckets=item_buckets)
            item_matrix = tf.get_variable(name='item_matrix',
                                          shape=(100, 16),
                                          initializer=tf.initializers.glorot_uniform())
    
            # shape: B * L * E (Embedding size)
            item_embedding = tf.nn.embedding_lookup(item_matrix,
                                                    item_id,
                                                    name='item_embedding')
    
        with tf.name_scope('history'):
            # shape: 训练: B * L * T (sequence length), 线上预测: 1 * T
            clicked_items = features['clicked_items_15d']
            # shape: 训练: B * L * T (sequence length), 线上预测: 1 * T
            clicked_mask = tf.cast(tf.not_equal(clicked_items, '0'), tf.bool)
            clicked_items = tf.string_to_hash_bucket_fast(clicked_items, num_buckets=item_buckets)
            # shape: 训练: B * L * T * E, 线上预测: 1 * T * E
            clicked_embedding = tf.nn.embedding_lookup(item_matrix,
                                                       clicked_items,
                                                       name='clicked_embedding')
    
        # if mode == tf.estimator.ModeKeys.PREDICT:
        # 复制 user_embedding 为 B * L * concatenated embedding size
        user_embedding = tf.expand_dims(user_embedding, 1)
        user_embedding = tf.tile(user_embedding, [1, list_size, 1])
        if mode == tf.estimator.ModeKeys.PREDICT:
            # 1 * T * E
            clicked_embedding = tf.expand_dims(clicked_embedding, 1)
            # 1 * L * T * E
            clicked_embedding = tf.tile(clicked_embedding, [1, list_size, 1, 1])
            # 1 * T
            clicked_mask = tf.expand_dims(clicked_mask, 1)
            # 1 * L * T
            clicked_mask = tf.tile(clicked_mask, [1, list_size, 1])
    
        # shape: B * L * E
        clicked_attention = attention(clicked_embedding,
                                      item_embedding,
                                      clicked_mask,
                                      [16, 8],
                                      'clicked_attention')
    
        fc_inputs = tf.concat([user_embedding, item_embedding, clicked_attention], axis=-1, name='fc_inputs')
    
        with tf.name_scope('predictions'):
            # B * L * 1
            logits = fc_layers(mode, net=fc_inputs, hidden_units=[64, 16, 1], dropout=0.3)
            logits = tf.squeeze(logits, axis=-1)  # B * L
    
            if mode == tf.estimator.ModeKeys.PREDICT:
                predictions = tf.nn.softmax(logits, name='predictions')  # B * L (线上预测时 B = 1)
                predictions = {
                    'probability': predictions
                }
                export_outputs = {
                    'predictions': tf.estimator.export.PredictOutput(predictions)
                }
                return tf.estimator.EstimatorSpec(mode, predictions=predictions,
                                                  export_outputs=export_outputs)
            else:
                # B * L
                relevance = tf.cast(labels, tf.float32)  
                soft_max = tf.nn.softmax(relevance, axis=-1)  # 1
                mask = tf.cast(relevance >= 0.0, tf.bool)
                loss = _masked_softmax_cross_entropy(logits=logits, labels=soft_max, mask=mask)  # 2
    
                if mode == tf.estimator.ModeKeys.EVAL:
                    weights = tf.cast(mask, tf.float32)
    
                    metrics = {}
                    metrics.update(_ndcg(relevance, logits, weights=weights, name='ndcg'))
    
                    for metric_name, op in metrics.items():
                        tf.summary.scalar(metric_name, op[1])
                    return tf.estimator.EstimatorSpec(mode, loss=loss,
                                                      eval_metric_ops=metrics)
                else:
                    global_step = tf.train.get_global_step()
                    learning_rate = exponential_decay(global_step,
                                                      init_learning_rate,
                                                      decay_steps,
                                                      decay_rate)
                    optimizer = tf.train.AdagradOptimizer(learning_rate=learning_rate)
                    tf.summary.scalar('learning_rate', learning_rate)
                    train_op = optimizer.minimize(loss=loss, global_step=global_step)
                    return tf.estimator.EstimatorSpec(mode, loss=loss, train_op=train_op)
    
    
    def attention(history, target, history_mask, hidden_units, name='attention_out'):
        """
        :param history: 训练: B * L * T * E, 线上预测: 1 * T * E
        :param target: 训练: B * L * E, 线上预测: 1 * L * E
        :param history_mask: 训练: B * L * T, 线上预测: 1 * T
        :param hidden_units: hidden units
        :param name: name
        :return: weighted sum tensor: 训练: B * L * E, 线上预测: 1 * L * E
        """
        # list_size: get L
        list_size = tf.shape(history)[1]
        # keys length: get T
        keys_length = tf.shape(history)[2]
        target_emb_size = target.get_shape()[-1]
        target_emb_tmp = tf.tile(target, [1, 1, keys_length])
        target = tf.reshape(target_emb_tmp, shape=[-1, list_size, keys_length, target_emb_size])
        net = tf.concat([history, history - target, target, history * target], axis=-1)
        for units in hidden_units:
            net = tf.layers.dense(net, units=units, activation=tf.nn.relu)
        # net: B * L * T * hidden_units[-1]
        # attention_weight: B * L * T * 1
        attention_weight = tf.layers.dense(net, units=1, activation=None)
        scores = tf.transpose(attention_weight, [0, 1, 3, 2])  # B * L * 1 * T
        history_mask = tf.expand_dims(history_mask, axis=2)  # B * L * T --> B * L * 1 * T
        padding = tf.zeros_like(scores)  # B * L * 1 * T
        # mask 为 true 时使用 score, false 时使用 0
        scores = tf.where(history_mask, scores, padding)  # B * L * 1 * T
        outputs = tf.matmul(scores, history)  # [B * L * 1 * T] * [B * L * T * E] --> [B * L * 1 * E]
        outputs = tf.squeeze(outputs, axis=2, name=name)  # 去掉维度为 1 的 axis, B * L * E
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
    
    
    def _ndcg(relevance, predictions, ks=(1, 4, 8, 20, None), weights=None, name='ndcg'):
        ndcgs = {}
        for k in ks:
            metric = metrics_impl.NDCGMetric('ndcg', topn=k, gain_fn=lambda label: tf.pow(2.0, label) - 1,
                                             rank_discount_fn=lambda rank: tf.math.log(2.) / tf.math.log1p(rank))
    
            with tf.name_scope(metric.name):
                per_list_ndcg, per_list_weights = metric.compute(relevance, predictions, weights)
    
            ndcgs.update({'{}_{}'.format(name, k): tf.metrics.mean(per_list_ndcg, per_list_weights)})
    
        return ndcgs
    
    
    def _masked_softmax_cross_entropy(logits, labels, mask):
        """Softmax cross-entropy loss with masking."""
        padding = tf.ones_like(logits) * -2 ** 32
        logits = tf.where(mask, logits, padding)
        return tf.reduce_mean(tf.nn.softmax_cross_entropy_with_logits(logits=logits, labels=labels))
    

这里主要提一下注释 #1 和 #2。

**第 21 篇** 提到的损失函数：真实概率分布和预测概率分布计算分布差。

注释 #1 就是计算真实概率分布，#2 是计算损失函数，logits 是经过 softmax 之后得到的就是预测概率分布。

这里的 relevance 是相关性，形状为 B * L，每个元素的值为 0、1、2、3。

假设有两条训练数据：

  1. 第一条的 list size 为 2，relevance 为 [0, 1]，将 relevance 经过 softmax 得到 [0.269, 0.731]。
  2. 第二条的 list size 为 3，relevance 为 [0, 1, 2]，将 relevance 经过 softmax 得到 [0.090, 0.245, 0.665]。

但是，实际情况是：因为有 pad 操作，会将上面两条数据的形状进行 pad，得到形状为 2 * 3 的数据。如下所示：

$$\begin{align} & [0, 1, PAD] \&[0, 1, 2]\end{align}$$

TensorFlow 在计算 softmax 时，会按行计算，显然 PAD 的值不应该影响真实概率分布，也就是说上述 PAD 后的数据经过 softmax
之后，得到的概率应该如下所示：

$$\begin{align} & [0.269, 0.731, 0.0] \&[0.090, 0.245, 0.665]\end{align}$$

现在，你知道为什么要在读取数据时将 pad 值设置为 $-2^{32}$ 了吗？还有函数 _masked_softmax_cross_entropy
中对预测 logits 进行了 pad，均设置为 $-2^{32}$，这个值经过 softmax 之后会变为 0.0，不会对真实概率分布造成任何影响。

同理，在计算 NDCG 时，将所有被 pad 的 relevance 权重设置为 0，从而在计算指标时不会产生任何的干扰。

pad 的默认值设为 $-2^{32}$ 和 计算离线指标 NDCG 时将 pad 的 relevance 权重设为
0，是两个看上去微不足道但是特别重要的地方，不仅会影响模型的质量，而且可能会对离线指标产生重要影响。希望读者们在实现 listwise LTR
时一定要注意。

本篇完整代码详见 [GIT](https://github.com/recsys-4-everyone/dist-
recsys/tree/main/chapter%2024)。

#### **模型训练、导出 和 Serving**

模型的训练、导出 和 Serving 部分的代码与上一章节完全没有差别，再次就不赘述了。

### 小结

本篇详细讲解了 listwise LTR 如何用 TensorFlow
来实现，特殊的数据格式以及训练方式让它稍微有点难以理解，对于它的实现也并不是很多，希望这篇文章能为你带来一定的启发。

TensorFlow 提供了 SequenceExample 存储 listwise 模型的数据，为我们带来了很多的便利，在数据读取时一定要关注
relevance 的默认填充值，稍不注意就会影响模型的质量。同理，在计算 NDCG 时，也要特别注意对于 pad 的 relevance，需要将其权重置为
0，让它不会对 NDCG 的计算产生影响。

通过几篇的内容，大家一定注意到了深度模型有很多的超参（模型没法学习，只能通过人工设置的参数）需要调，但是训练数据动辄上亿条，训练一次时间成本太高，这么多的超参我们该如何去调呢，这也就是为什么算法工程师会经常被戏谑地称为“调参侠”的原因。

下一篇，我们就来看看深度学习中的超参有哪些，各自的优先级怎样，如何去调节它们。

下篇见。

注释 1：[Learning to Rank: From Pairwise Approach to Listwise
Approach](https://www.microsoft.com/en-us/research/wp-
content/uploads/2016/02/tr-2007-40.pdf)

