从这篇开始，我们将重点放在 **分布式** 上，且只会着眼于 TensorFlow 框架，我们会探讨分布式训练的必要性、分布式的原理是什么以及将单机环境
DIN 训练代码适配为分布式环境训练代码。

其他机器学习训练框架，比如 PyTorch 工业中应用得还不是很广泛，学术界中使用比较多。

### 为什么需要分布式

使用分布式只有一个原因——数据太多/模型太大。

**数据多**

数据量太多时，单机环境力不从心，尤其是现在数字化极速发展，几乎每个人每天都会在手机、平板和 PC
等终端产生大量的数据。当单机环境实在难以在合理的时间范围内训出一个高质量模型时，就要考虑采用分布式的方式去训练模型了。

还有一个很重要的原因就在于——试错。一个高质量模型的诞生，不是凭空出现的，而是要经过若干次的离线实验（调参）得到的，而海量数据的存在会导致在单机环境做离线实验的时间成本过高，我想不会有多少组织或者个人能够接受模型要训十天半个月才能上线。因此，当数据量过大时，需要分布式训练的最根本原因就是降低试错成本、时间成本，提高建模效率。

读者可能会有疑问，海量数据导致模型更新频率低的问题不是通过 Online Learning 解决了吗？其实 Online Learning
只是一种模型训练方式，我们可以在单机环境去做 Online
Learning，也可以在分布式环境去做，本身它与分布式并不是互斥的关系，相反，两者的结合会更好的解决模型更新慢的问题。

**模型大**

深度模型的结构越来越复杂，参数越来越多，创造一个几十 G 上百 G
的模型不再是难事，可是服务器的内存是有限的，当模型的容量已经大到单台服务器容纳不下时，也必须要考虑分布式训练。

### 数据并行和模型并行

对应着上述两个问题（数据多、模型大）就诞生了两种分布式训练策略：

  * 数据并行
  * 模型并行

#### **数据并行**

假设有 N 台服务器，那么数据会被分成 N 份，每台服务器只训练其中的 1 份数据，可见理论上训练速度可以提升 N 倍。数据并行的总体框架图（这里展示的是
Parameter Server 架构，当然还有其他架构，比如 All Reduce 等等）如下所示：

![](https://images.gitbook.cn/c510ead0-a1a8-11eb-8d7d-61d3e9c8e4b1)

每台工作服务器各自使用各自的训练数据进行训练，彼此之间互不干扰，训完一批数据，产生梯度后，再将梯度推送到参数服务器上，由参数服务器对参数进行更新。当下一批次的数据到来时，工作服务器继续从参数服务器上拉取最新更新后的参数进行训练。

Map-Reduce 思想，由工作服务器并行训练产生梯度（map），参数服务器对多个工作服务器传来的梯度进行聚合（reduce）。

#### **模型并行**

模型并行在实际应用中不太常见，笔者在工作中也很少遇到使用模型并行的情况。模型并行的使用场景是，当模型已经大到单台服务器内存容不下时，会将模型的不同部分放置在不同的服务器上，如下图所示：

![](https://images.gitbook.cn/db20ce30-a1a8-11eb-9901-85e993a6256e)

这样一个超大的模型就被拆分到了不同的服务器上，但是这又带来了另外一个问题，模型并行的网络开销可能比数据并行时大得多，要知道数据并行时，传递的是梯度，而模型并行时传递的可是输入数据，后者往往比前者大的多。不仅如此，模型并行反向传播时没有办法做到并行化（想想为什么），也是一个很大的问题。因此我们主要讲解
**数据并行** 的分布式训练，尤其是数据并行中的 **参数服务器架构** 。

### 参数服务器架构

一般会有多种架构可以支持数据并行，Parameter Server，All Reduce，Ring All Reduce 等等，虽然 Ring All
Reduce 使用的也比较广泛，但是我们还是着眼于 Parameter Server（参数服务器，简写为 PS）。

推荐系统中，模型特征特别的稀疏，Parameter Server 架构比较适合这种场景，而 [Ring] All Reduce
等架构需要每台训练节点都存储全量的模型，当模型很大时，就显得力不从心，对于带宽的要求也比较的高，特别适用于单机多卡环境。

现在我们使用的参数服务器架构，是由李沐在 2013 年提出的[^1]，主要结构$^1$ 如下所示：

![](https://images.gitbook.cn/ecc6eed0-a1a8-11eb-9901-85e993a6256e)

可以看到，这种架构将节点分成了两种类型——server 和 worker。

  * server 存储着模型参数，可以简单理解为分布式键值数据库，一般 server 是多台，以防单点故障。worker 就负责训练模型。
  * worker 读取训练数据、从 server 拉取参数、计算 LOSS 和梯度、并将梯度发送给 server。
  * server 接收 worker 发送的梯度、更新和保存参数，可以简单理解为分布式键值数据库，一般 server 是多台，以防单点故障。

在 PS 架构下，训练过程如下：

  1. worker 读取数据，从 server 拉取参数。
  2. 有了数据和参数，worker 计算 LOSS，得到梯度，再将梯度回传给 server。
  3. server 拿到梯度后，对参数进行更新，完成一轮训练。

上述参数更新过程如下图$^1$ 所示：

![](https://images.gitbook.cn/de4a5350-a1a9-11eb-a6d9-e9904e63f59e)

在这里，要注意一个很重要的点：

当对数据进行分片时，每个 worker 只能看到部分数据，训练时，worker 拉取的模型参数 **并不是** 全量模型参数，而只是该 worker
能看到的训练数据对应的 参数，这就大大减少了网络开销，特别是当特征非常稀疏时，这种网络开销减少的更为明显。而且也不用担心模型参数太大时，worker
能不能容得下的问题，因为每个 worker 只会保留很少一部分参数。如下图$^1$ 所示：

![](https://images.gitbook.cn/faf1aee0-a1a9-11eb-9901-85e993a6256e)

上图描述的是 **单个 worker 上的参数占全量参数的比例与 worker 总数** 的关系，可见当 worker 数量在 100 时，每个
worker 只保有大概 1/10 的参数，这对于内存和带宽已经非常友好了。

举个最简单的例子，线性模型 M，一共有两个特征 $a_1$和 $a_2$，$label$ 与特征的关系如下：

$$y = a_1x_1 +a_2x_2+b$$

其中 $y$ 是 $label$，$x$ 是特征，$b$ 是 $bias$，采用 PS 架构训练进行分布式训练，有 2 个 server、2 个
worker，参数 $a_1$ 存储在 $server_1$ 上，$a_2$ 存储在 $server_2$ 上。

当 worker​ 各自训练时，不可避免地会出现 $worker_1$ 和 $worker_2$ 均向 $server_1$ 发送 $a_1$ 的梯度以及向
$server_2$ 发送 $a_2$ 的梯度的情况，这就涉及到了 server 在接收到同一个特征的多个梯度时如何对参数进行的问题。

#### **同步更新**

当开启同步更新时，对于同一个特征，server 接收到 worker 的梯度，会一直等待累计到 n （n 是 worker
数）个梯度后才会对参数进行更新，如果其中有 worker 比较慢，那么就一直等着，完美的木桶理论的代表，训练速度被最慢的 worker
拖累，所以如果集群中节点的性能差异很大时，是不适合用同步更新的。

当然 Tensorflow 1 中，可以不用等待 n 个梯度，而是可以指定为＜n 的数，这样就可以只等部分梯度到位后就可以更新参数了。Tensorflow
2 中，目前（2.4 版本）PS 架构还不支持同步更新。

#### **异步更新**

与同步更新相反，开启异步更新时，对于同一个特征，server 只要收到任意 worker 发送来的梯度就进行更新，不等其他 worker
该特征的梯度。这样带来的好处就是训练速度大大提升，不再受木桶理论的约束。坏处就是训练的模型有可能不够稳定，也会影响模型的收敛。

在实际应用中，究竟是采用同步还是异步，最好是通过实验验证一下，可能同步好，可能异步好，可能没区别。

接下来就到了本篇最核心的部分啦——将第 23 节的单机训练代码移植到分布式环境（PS 架构，异步训练）。

### 单机代码移植

软件环境：

  * Spark 2.4.0
  * Tensorflow 1.15
  * Python 3.6.0

完整代码详见 [GitHub](https://github.com/recsys-4-everyone/dist-
recsys/tree/main/chapter%2031)。

我们知道单机训练时，代码的总体组成结构如下：

  * 生成数据 —— Spark
    * data_generator.py
  * 生成模型 —— Tensorflow
    * 读取数据：data.py
    * 搭建网络：estimator.py
    * 导出模型：export.py
    * 程序入口：main.py

按照上述结构，我们看看有哪些部分需要修改。

#### **生成数据**

这部分代码不用修改，要注意的就是为了达到分布式训练的效果，数据最好保存成多个文件。

#### **生成模型**

**1\. 读取数据**

既然是数据并行，那么就需要让每台工作节点看到不同的数据，加快训练速度，比如总共 100 条数据，10 个工作节点，平均每个节点可以分 10
条数据，训练速度理论上可以提高 10 倍，对应的代码就要稍作修改。

    
    
    # -*- coding: utf-8 -*-
    
    import tensorflow as tf
    from tensorflow.compat.v1 import data
    from tensorflow.compat.v1.data import experimental
    import os
    
    
    def get_example_fmt():
        example_fmt = dict()
    
        example_fmt['label'] = tf.FixedLenFeature([], tf.int64)
        example_fmt['user_id'] = tf.FixedLenFeature([], tf.string)
        example_fmt['age'] = tf.FixedLenFeature([], tf.int64)
        example_fmt['gender'] = tf.FixedLenFeature([], tf.string)
        example_fmt['item_id'] = tf.FixedLenFeature([], tf.string)
        # 此特征长度不固定
        example_fmt['clicked_items_15d'] = tf.VarLenFeature(tf.string)
    
        return example_fmt
    
    
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
    
    
    def parse_fn(example):
        example_fmt = get_example_fmt()
        parsed = tf.parse_single_example(example, example_fmt)
        # VarLenFeature 解析的特征是 Sparse 的，需要转成 Dense 便于操作
        parsed['clicked_items_15d'] = tf.sparse.to_dense(parsed['clicked_items_15d'], '0')
        label = parsed['label']
        parsed.pop('label')
        features = parsed
        return features, label
    
    
    def input_fn(mode, pattern, epochs=1, batch_size=512, num_workers=None, worker_index=None):
        cpus = os.cpu_count()
        padded_shapes, padding_values = padded_shapes_and_padding_values()
        files = tf.data.Dataset.list_files(pattern)
        if num_workers and num_workers > 0 and worker_index > -1: #1
            files = files.shard(num_workers, worker_index)
        data_set = files.apply(experimental.parallel_interleave(tf.data.TFRecordDataset, cycle_length=cpus, sloppy=True))
        data_set = data_set.apply(experimental.ignore_errors())
        data_set = data_set.map(map_func=parse_fn, num_parallel_calls=cpus)
        if mode == 'train':
            data_set = data_set.shuffle(buffer_size=10000)
            data_set = data_set.repeat(epochs)
        data_set = data_set.padded_batch(batch_size, padded_shapes=padded_shapes, padding_values=padding_values)
        data_set = data_set.prefetch(buffer_size=experimental.AUTOTUNE)
        return data_set
    

相比于单机环境，分布式环境我们需要告诉 Tensorflow 一共有多少个工作节点(num_workers)，以及当前工作节点的编号(0 到
num_workers - 1)，shard 函数会自动将数据均分。

注释 #1 处也是唯一需要修改的地方。

**2\. 搭建网络**

建模的代码可以不做任何修改，但是有时候需要考虑一下：

> 如果物品 embedding 矩阵过大，单个 ps 存放不下或者负载过重时，应该怎么办？

这时候需要对这个变量进行分片，将其均匀的分在多个 ps 上，达到负载均衡，实现起来很简单，代码的注释 #1 处可以解决这个问题。

    
    
    # -*- coding: utf-8 -*-
    import math
    import tensorflow as tf
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
            # shape: B (batch size)
            user_embedding = fc.input_layer(features, [user_id, age, gender])
    
        with tf.name_scope('item'):
            item_buckets = 100
            item_id = features['item_id']
            item_id = tf.reshape(item_id, [-1, 1])
            list_size = tf.shape(item_id)[0]
            item_id = tf.string_to_hash_bucket_fast(item_id, num_buckets=item_buckets)
            # if matrix is huge, it can be distributed
            # item_matrix = tf.get_variable(name='item_matrix',
            #                               shape=(100, 16),
            #                               initializer=tf.initializers.glorot_uniform())
            if mode != tf.estimator.ModeKeys.PREDICT:
                ps_num = len(params['tf_config']['cluster']['ps'])
                item_matrix = tf.get_variable(name='item_matrix',
                                              shape=(100, 16),
                                              initializer=tf.initializers.glorot_uniform(),
                                              partitioner=tf.fixed_size_partitioner(num_shards=ps_num)) #1
            else:
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
            clicked_mask = tf.cast(tf.not_equal(clicked_items, '0'), tf.bool)
            clicked_items = tf.string_to_hash_bucket_fast(clicked_items, num_buckets=item_buckets)
            # shape: B * T * E
            clicked_embedding = tf.nn.embedding_lookup(item_matrix,
                                                       clicked_items,
                                                       name='clicked_embedding')
    
        if mode == tf.estimator.ModeKeys.PREDICT:
            user_embedding = tf.tile(user_embedding, [list_size, 1])
            clicked_embedding = tf.tile(clicked_embedding, [list_size, 1, 1])
            clicked_mask = tf.tile(clicked_mask, [list_size, 1])
    
        # shape: B * E
        clicked_attention = attention(clicked_embedding,
                                      item_embedding,
                                      clicked_mask,
                                      [16, 8],
                                      'clicked_attention')
    
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
        padding = tf.ones_like(scores) * (-2 ** 32 + 1)
        scores = tf.where(history_masks, scores, padding)
        scores = tf.nn.softmax(scores)
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
    

**3\. 导出模型**

代码不需做任何修改。

**4\. 程序入口**

需要修改，Tensorflow 需要知道分布式集群拓扑结构——ps 的机器是哪些，worker 的机器是哪些等等。

分布式环境中，节点被分成了 4 个部分：

  1. ps：存储和更新模型参数
  2. worker：工作节点，读取数据，训练模型
  3. chief：特殊的 worker 节点，它除了需要做一般 worker 的工作之外，还需要做保存模型 checkpoints 等额外工作
  4. evaluator：只负责做评估工作，定时拉取最新的模型，使用验证集做评估

    
    
    # -*- coding: utf-8 -*-
    
    import os
    import json
    import args
    import argparse
    from data import input_fn
    from estimator import model_fn
    from tensorflow.compat.v1 import app
    from tensorflow.compat.v1 import logging
    from tensorflow.compat.v1 import estimator
    from tensorflow.compat.v1 import ConfigProto
    from tensorflow.compat.v1.distribute import experimental
    
    parser = argparse.ArgumentParser()
    args.add_arguments(parser)
    flags, un_parsed = parser.parse_known_args()
    
    
    def _tf_config(_flags):
        tf_config = dict()
        ps = ['localhost:2220']
        chief = ['localhost:2221']
        worker = ['localhost:2222']
        evaluator = ['localhost:2223']
    
        cluster = {
            'ps': ps,
            'chief': chief,
            'worker': worker,
            'evaluator': evaluator
        }
    
        task = {
            'type': _flags.type,
            'index': _flags.index
        }
    
        tf_config['cluster'] = cluster
        tf_config['task'] = task
    
        if _flags.type == 'chief':
            _flags.__dict__['worker_index'] = 0
        elif _flags.type == 'worker':
            _flags.__dict__['worker_index'] = 1
    
        _flags.__dict__['num_workers'] = len(worker) + len(chief)
    
        _flags.__dict__['device_filters'] = ["/job:ps", f"/job:{_flags.type}/task:{_flags.index}"]
    
        return tf_config
    
    
    def main(_):
        cpu = os.cpu_count()
        tf_config = _tf_config(flags) #1
        # 分布式需要 TF_CONFIG 环境变量
        os.environ['TF_CONFIG'] = json.dumps(tf_config) #2
        session_config = ConfigProto(
            device_count={'CPU': cpu},
            inter_op_parallelism_threads=cpu // 2,
            intra_op_parallelism_threads=cpu // 2,
            device_filters=flags.device_filters,
            allow_soft_placement=True)
        strategy = experimental.ParameterServerStrategy()
        run_config = estimator.RunConfig(**{
            'save_summary_steps': 100,
            'save_checkpoints_steps': 1000,
            'keep_checkpoint_max': 10,
            'log_step_count_steps': 100,
            'train_distribute': strategy,
            'eval_distribute': strategy,
        }).replace(session_config=session_config)
    
        model = estimator.Estimator(
            model_fn=model_fn,
            model_dir='/home/axing/din/checkpoints/din', #实际应用中是分布式文件系统
            config=run_config,
            params={
                'tf_config': tf_config,
                'decay_rate': 0.9,
                'decay_steps': 10000,
                'learning_rate': 0.1
            }
        )
    
        train_spec = estimator.TrainSpec(
            input_fn=lambda: input_fn(mode='train',
                                      num_workers=flags.num_workers,
                                      worker_index=flags.worker_index,
                                      pattern='/home/axing/din/dataset/*'), #3
            max_steps=1000  #4
        )
    
        # 这里就假设验证集和训练集地址一样了，实际应用中是肯定不一样的。
        eval_spec = estimator.EvalSpec(
            input_fn=lambda: input_fn(mode='eval', pattern='/home/axing/din/dataset/*'),
            steps=100,  # 每次验证 100 个 batch size 的数据
            throttle_secs=60  # 每隔至少 60 秒验证一次
        )
        estimator.train_and_evaluate(model, train_spec, eval_spec)
    
    
    if __name__ == '__main__':
        logging.set_verbosity(logging.INFO)
        app.run(main=main)
    

注释说明如下：

  * #1、#2 处生成分布式网络拓扑，告知每个节点的角色，并写入系统环境变量 TF_CONFIG
  * #3 处将 num_workers 和 worker_index 传入 input_fn，实现数据分片
  * #4 处告知 Tensorflow 最多训练 max_steps 个 batch size

程序启动命令：

    
    
        # nohup python main.py --type=ps --index=0 > ps.log 2>&1 &
        # nohup python main.py --type=chief --index=0 > chief.log 2>&1 &
        # nohup python main.py --type=worker --index=0 > worker.log 2>&1 &
        # nohup python main.py --type=evaluator --index=0 > evaluator.log 2>&1 &
    

每个节点的角色，各自的 index 都是从 0 开始，上述运行在本地起了 4 个进程模拟分布式环境，分别占用了 2220、2221、2222、2223 这
4 个端口。

后续的导出和 Serving 不需做任何的修改。至此，单机训练代码迁移到分布式环境就完全结束了。

这里是为了演示方便，所以是本地起多个进程来模拟分布式环境，实际应用中，我们的模型存储地址（model_dir）必须是分布式文件系统中的地址这样所有的节点都能够访问的到，不然是肯定会训练失败的。同理，训练数据也必须存放在分布式文件系统中。

### 小结

随着业务量的增长，数据量也会水涨船高，当某个时间节点忽然发现线下调参或者线上模型训练的时间已经超出可以接受的范围时，分布式训练几乎是唯一的选择——缩短训练时间，不影响模型质量。

在推荐系统中，因为特征的稀疏性，PS 架构一般来说是我们的首选，当然也可以尝试其他架构（All Reduce 等）。PS
架构的参数更新有同步和异步两种，读者们可以根据自己的需要在实际应用中选择适合自己的同步方式。

关于分布式训练，这里推荐一篇出自 FaceBook
的一篇很优秀的[论文](https://arxiv.org/pdf/1706.02677.pdf)$^2$，里面详细讨论了如何做分布式训练的模型加速以及调参，非常值得一读。

从单机代码移植的步骤中可以看出，移植还是比较简单的，需要修改的地方也比较少，但是我们同样能看到，现有的分布式实现方式存在巨大的缺点——太朴素了。

居然要手动执行每个节点，还有整个程序特别的脆弱，任意一个节点失败，它都没有办法“复活”，作为一名精益求精的工程师来说，实在不能接受这么原始的分布式训练方式，有没有更加自动化的框架或者工具帮助我们更好的完成分布式训练呢？

这么迫切的需求，当然有！

下一篇，我们手把手，一步一步地手动搭建一套基于 K8S 的分布式训练平台——希望能对读者们在实际应用中带去些许启发和帮助。

  * 注释 1：[参数服务器原始论文](https://www.cs.cmu.edu/~muli/file/parameter_server_osdi14.pdf)
  * 注释 2：[Accurate, Large Minibatch SGD: Training ImageNet in 1 Hour](https://arxiv.org/pdf/1706.02677.pdf)

