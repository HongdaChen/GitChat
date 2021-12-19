在后面的课程中我们将主要使用 TensorFlow 来实现各种模型的应用，所以在本节我们先来看一下 TensorFlow 的基础知识点。

TensorFlow 是一个深度学习库，由 Google 开源，可以对定义在 Tensor(张量)上的函数自动求导。

Tensor(张量)意味着 N 维数组，Flow(流)意味着基于数据流图的计算，TensorFlow 即为张量从图的一端流动到另一端。

它的一大亮点是支持异构设备分布式计算，它能够在各个平台上自动运行模型，从手机、单个CPU / GPU到成百上千GPU卡组成的分布式系统。支持 CNN、RNN
等主流算法，是目前在图像和自然语言处理中最流行的深度学习库。

接下来我们将对 TensorFlow 的一些核心概念展开学习。

### **本文结构：**

  * TensorFlow 的代码结构
  * 1\. 图 Graph
    * 1.1 图的定义
    * 1.2 建立图
    * 1.3 运行图：会话 session
      * 1.3.1 fetches
  * 2\. Tensor 和 Operation
    * 2.1 Tensor
      * 2.1.1 tensor 的 shape
    * 2.2. Operations
    * 2.3. Placeholder
      * 2.3.1. Feed
    * 2.4. Variable
      * 2.4.1 初始化
      * 2.4.2 改变值
      * 2.4.3 Trainable 可训练性
  * 3\. TensorBoard
    * 3.1 name scopes

* * *

为了能够对如何使用 TensorFlow 有个更全局的认识，更清晰地理解各概念，我们先从一个非常简单的代码例子入手：

下面这段代码主要实现了 "transformation" 的计算，就是输入一个向量，先分别计算一下元素间累积，和元素间和，再将上面两个结果相加。

**举个数例就是：**

![NBfWbH](https://images.gitbook.cn/NBfWbH.jpg)

    
    
    import tensorflow as tf
    
    graph = tf.Graph()                                        # 建立 graph，而不是用 default graph
    
    with graph.as_default():                                # 将新建立的 graph 设定为 default ，在其中建立模型
    
        with tf.name_scope("variables"):                    # 下面是两个 global 变量，因为这些变量是全局的，所以单独声明，
                                                            # 和其他的 node 区分开来，有它们单独的 name scope
    
            # Variable global_step 用来记录 graph 运行了多少次，这在 TF 中是个常见的范式
            # trainable = False 因为这个例子中不会进行训练，那就需要手动赋值了，而不是用算法训练的
            global_step = tf.Variable(0, dtype = tf.int32, trainable = False, name = "global_step")
    
            # Variable total_output 记录所有输出的总和
            total_output = tf.Variable(0.0, dtype = tf.float32, trainable = False, name = "total_output")
    
    
        with tf.name_scope("transformation"):               # 模型的核心计算所在，被封装在 name scope 为 “transformation” 中，
                                                            # 并且进一步划分为 “input”, “intermediate_layer”,  “output” 三个 name scopes
    
            # 输入结点是一个 tf.placeholder 可以接收任意长度的向量
            with tf.name_scope("input"):
                a = tf.placeholder(tf.float32, shape = [None])
    
            # 用 tf.reduce_prod() 和 tf.reduce_sum() 对整个输入向量进行相乘和相加
            with tf.name_scope("intermediate_layer"):
                b = tf.reduce_prod(a, name = "product_b")       
                c = tf.reduce_sum(a, name = "sum_c")
    
            # 将上面两个结果相加得到 ouput
            with tf.name_scope("output"):
                output = tf.add(b, c, name = "output")
    
    
        with tf.name_scope("update"):                       # 在上面的计算之后，要更新两个 Variables，建立 “update” scope 来记录这些改变
    
            # 用 Variable.assign_add() 来增加 total_output 和 global_step
            # 因为是记录所有的和，所以每次都把 output 加进去
            update_total = total_output.assign_add(output)      
    
            # global_step 当 graph 每运行依次就都加 1                                        
            increment_step = global_step.assign_add(1)          
    
    
        with tf.name_scope("summaries"):                    # 更新完变量之后，就建立我们感兴趣的 TensorBoard summaries
    
            # 记录平均输出值，这里用的是更新后的 update_total，确保 update_total 在 avg 之前完成
            # 同理，用 increment_step 保证 graph 有序运行
            avg = tf.div(update_total, tf.cast(increment_step, tf.float32), name="average")     
    
            # 计算完 avg 后，就建立 output, update_total， avg 各自的 summary
            tf.summary.scalar("output_summary", output)
            tf.summary.scalar("total_summary", update_total)
            tf.summary.scalar("average_summary", avg)
    
    
        with tf.name_scope("global_ops"):                   # 把下面这两项放到一个 "global_ops" 的 name scope 中
    
            # 建立变量初始化 Op
            init = tf.global_variables_initializer()                
    
            # 将所有的 summary 放到一个 Op 里
            merged_summaries = tf.summary.merge_all()   
    
    
    sess = tf.Session(graph=graph)                                        # 开启一个会话，启动刚建立的 graph
    
    writer = tf.summary.FileWriter('./improved_graph', graph)            # tf.summary.FileWriter 用来存储 summary
    
    sess.run(init)                                                        # 在做其他事之前，首先要初始化 variables
    
    def run_graph(input_tensor):                                        # run_graph 是个辅助函数，将输入向量传入，然后就运行 graph 并保存 summary
    
        # 建立 sess.run 要用的字典，和 a 这个 tf.placeholder 结点对应
        feed_dict = {a: input_tensor}    
    
        # 用上面的 feed_dict 来运行 graph，同时要保证运行 output, increment_step, merged_summaries 这几个 Ops，
        # 为了写 summary，我们需要保存 global_step 和 merged_summaries，分别存为 step 和 summary
        # _ 表示我们不需要保存 output 的值
        _, step, summary = sess.run([output, increment_step, merged_summaries],
                                      feed_dict=feed_dict)        
    
    
        # 将 summary 存到 summary.FileWriter，这个 global_step 是重要的，因为 TensorBoard 要用它作为 x 轴来可视化数据
        writer.add_summary(summary, global_step=step)    
    
    
    
    run_graph([2,8])                        # 用不同长度的输入向量来多次调用 run_graph
    run_graph([3,1,3,3])
    run_graph([8])
    run_graph([1,2,3])
    run_graph([11,4])
    run_graph([4,1])
    run_graph([7,3,1])
    run_graph([6,3])
    run_graph([0,2])
    run_graph([4,5,6])
    
    writer.flush()                            # 用 flush()  方法将 summaries 写入磁盘
    
    writer.close()                            # 关闭 summary.FileWriter
    
    sess.close()                            # 关闭 session
    

**上面这段代码的整体结构图为：**

![cz9q0X](https://images.gitbook.cn/cz9q0X.jpg)

> inputs：由 placeholders 接收，是一个任意长度的向量； 在箭头上：None 表示一个任意长度的向量， [] 表示一个标量；
> update 部分：包括更新变量的操作, 还有把数据传递给 Tensorboard； 用一个单独的 name scope 来包含两个变量
> variable，一个用来存储 outputs
> 的和，另一个用来记录图被运行的次数，因为这两个变量并不在我们要做的主要计算式之中，所以给它们放在一个单独的 name scope 中。 还有一个
> name scope 包含了 Tensorboard 要展示的所有 summaries，把它们都放在 update
> 部分之后，这样可以保证在更新完变量之后再去添加 summaries，不然顺序就会变乱。

### **通过这个例子，我们可以看到 TensorFlow 的代码结构为：**

  * 将计算流程表示成图 Graph；
    * 将数据表示为 Tensors；
    * 使用 Variables 来追踪变化的参数信息；
    * 用 Placeholder 为输入张量占位；
  * 通过 Sessions 来执行图计算；
    * 使用 Feeds 来填充数据；
    * 用 Fetches 抓取任意操作的结果；
  * 在 TensorBoard 中可视化想看的数据；
    * 用 Name scope 来组织图的组件； 

![50i1z5](https://images.gitbook.cn/50i1z5.jpg)

接下来让我们逐个了解这几个概念。

* * *

**1\. Graph 1.1 图的定义 1.2 建立图 1.3 运行图：会话 Session 1.3.1 Fetch**

### **1.1 图的定义**

在 TensorFlow 中用计算图来表示计算任务。 计算图，是一种有向图，用来定义计算的结构，实际上就是一系列的函数的组合。
用图的方式，用户通过用一些简单的容易理解的数学函数组件，就可以建立一个复杂的运算。

**图，在形式上由结点 Nodes 和边 Edges 组成。** \- Nodes，用圆圈表示，代表一些对数据进行的计算或者操作（Operation）。
\- Edges，用箭头表示，是操作之间传递的实际值（Tensor）。

例如，下图中 b 这个红色圆圈，它就是一个计算操作，是求乘积。 输入 Tensor ［5，3］，经过 prod 这个结点，输出了 Tensor 15。

![SLoaHZ](https://images.gitbook.cn/SLoaHZ.jpg)

在 TensorFlow 使用图，分为两步：建立计算图 和 执行图。

* * *

### **1.2 建立图**

建立 graph 很简单，只需要下面这行：

    
    
    graph = tf.Graph()
    

初始化 graph 之后，就可以通过使用 **`Graph.as_default()`** 访问它的上下文管理器，接着就可以在其中添加 Op：

    
    
    with graph.as_default():    
    

用 **with** 表示我们用 context manager 告诉 TensorFlow 我们要向某个具体的 graph 添加 Op 了。
如果不做此声明的话，TensorFlow 会自动建立一个 graph，并且把它当作 default， 所有在 `Graph.as_default()`
之外建立的 Op 和 tensor 都会被添加到这里。

在大多数 TensorFlow 程序中，只需要在 default graph 里操作就可以了， 不过当需要建立多个没有依赖关系的模型时，可以 **建立多个
graph** ， 当建立多个 graph 时，要么不用自动建立的 default graph，要么立刻给它分配个名字，以便添加结点时比较统一，不会混淆，

当然，也可以从其他脚本调用已经定义好的模型，并用 `Graph.as_graph_def() 和 tf.import_graph_def` 添加到
graph 里， 这样就可以在同一个文件中调用若干个不同的模型。

* * *

### **1.3 执行图：Session**

图必须在会话(Session)里被启动，会话(Session)将图的 op 分发到 CPU 或 GPU 之类的设备上，同时提供执行 op
的方法，这些方法执行后，将产生的张量(tensor)返回。

会话就是为了执行 graph 的，此时就需要**开启会话 Session **：

    
    
    sess = tf.Session()
    

一旦开启了 Session，就可以 **用 run() 来计算** 想要的 Tensor 的值，

例如下面这个小例子，run(b) 可以得到 21：

    
    
    import tensorflow as tf
    
    a = tf.add(2, 5)
    b = tf.mul(a, 3)
    
    sess = tf.Session()
    
    sess.run(b)  # 返回 21
    

用完会话，记得关掉：

    
    
    sess.close()
    

当然 **Session 也可以作为上下文管理器** context manager，这样在它的代码范围外就会自动被关闭。

    
    
    a = tf.constant(5)
    
    sess = tf.Session()
    
    with sess.as_default():            # 用 as_default()  将 Session 作为 context manager
        a.eval()
    
    sess.close()
    

* * *

### **1.3.1 Fetch**

在 `Session.run()` 中需要接收一个必须的参数 **fetches** ， 还有其他三个可选的参数 `feed_dict, options,
run_metadata`，本节将主要看 `fetches` 和 `feed_dict`。

**fetches** 可以接收任何一个我们想要执行的 op 或者 Tensor，或者它们的 list。 如果是 Tensor，那么 run()
的输出就是一个 NumPy array， 如果是 Op，那么 run() 的输出是 None，

`sess.run(b)` 在前面这个例子中 fetches 设为 b，就 **告诉 Session 要把所有计算 b
所需要的结点都找到，按顺序执行，并且输出结果** ，

**`tf.global_variables_initializer()`** 是将所有的 Variable 都准备好，以便进行后续使用，这个 Op
也可以传递给 Session.run()。

如代码中的这两行：

    
    
    init = tf.global_variables_initializer()
    ...
    sess.run(init)
    

* * *

看完了图的建立和执行后，我们来看 TensorFlow 中重要的数据结构。

在 TensorFlow 中用 Tensor 数据结构来代表所有的数据, 计算图中操作间传递的数据都是 Tensor。

**\- 2. Tensor 和 Operation \- 2.1 Tensor \- 2.1.1 Tensor 的 shape \- 2.2
Operations \- 2.3 Placeholder \- 2.3.1. Feed \- 2.4 Variable \- 2.4.1 变量的初始化
\- 2.4.2 改变变量的值 \- 2.4.3 变量的 Trainable 可训练性**

### **2.1 Tensor**

在 TensorFlow 中，结点到结点之间传递的就是 Tensor. **Tensors** , 简单来说可以理解成是 n 维矩阵 ，当 n＝1
时是个向量 vector, n＝2 时是个矩阵 matrix。

定义时 **可以直接使用 NumPy 的数据类型** ，而不是必须使用 TensorFlow 中的数据类型，因为任何 NumPy 矩阵都可以被传递给
TensorFlow 的 Op， TensorFlow 中的 Op 可以将 Python 的数据类型转化为 Tensor，包括 numbers,
booleans, strings, 或者上述的 lists。 单个值会被转为 0 维的 Tensor 或者叫 scalar，list 转为一维，lists
of lists 转为二维，以此类推，

* * *

### **2.1.1 Tensor 的 shape**

在 TensorFlow 中的 Op，很多都会用到 tensor 的 “shape”，shape 描述了 Tensor 的维度数目和每个维度的长度。

例如这个 tensor： [[1 ,2], [3, 4], [5, 6]] 它的 shape 为 (3, 2)，第一个维度长为 3，第二个维度长为 2，

用 `tf.shape` Op 就可以得到一个 tensor 的 shape，`shape = [None]` 表示可以任意长度。

* * *

### **2.2. Operations**

图中任何结点都叫做 **Operation** ，记作 Op。

Op 的输入 inputs 是计算所需要的 tensor ，和其他额外的用于计算的信息属性，除了 inputs 和 attributes，每个 Op
还有一个字符型参数 name。 输出是 0 或者多个 tensor，这个输出会被传递到其他 Op 或者 Session.run。

    
    
    output = tf.add(b, c, name = "output")
    

我们给 add 这个 Op 命名为 output，那么就可以在 TensorBoard 用名字来识别相应的 Op。 当然还可以用 `name_scope`
可以给一组 Op 一起命名。

* * *

### **2.3. Placeholder**

**Placeholder** 用来给节点输入数据，在 TensorFlow 中用 placeholder
来描述等待输入的节点，刚建立时是没有自己的值的，只需要指定类型即可，然后在执行节点的时候用一个字典来“喂”这些节点。 相当于先把变量 hold
住，为即将喂进来的 tensor 先占住一个位置，然后每次从外部传入数据，注意 **placeholder 和`feed_dict` 是绑定用的**。

建立 Placeholder 用 `tf.placeholder`：

    
    
    # 建立一个任意长度的 placeholder 向量，数据类型为 float32
    a = tf.placeholder(tf.float32, shape = [None])
    

它接收两个参数： dtype：输入的数据类型，是必须的 shape：喂入的 tensor 的 shape

* * *

### **2.3.1. Feed**

前面只是用 placeholder 占了个位置， **给`tf.placeholder` 赋予实际的值要用 `feed_dict`**，
`feed_dict` 的输入是一个 Python 字典，placeholder 输出的 Tensor 名作为 key， 想要传递的 Tensor 作为
value，value 可以是 numbers, strings, lists, 或者 NumPy arrays，value 和 key 必须是同一种类型。

    
    
    sess = tf.Session()
    
    # 建立一个字典 input_dict 传递给 `feed_dict`
    # Key: `a`, placeholder 输出的 Tensor 名
    # Value: 向量 [5, 3]， 数据类型 int32 
    input_dict = {a: np.array([5, 3], dtype = np.int32)}
    
    # Fetch `d` 的值, 将 `input_vector` 喂给 `a`
    sess.run(d, feed_dict = input_dict)
    

**fetch 的输出 d 所依赖的每个 placeholder 例如 a，都要把它的 key-value 对加入到`feed_dict`**，如果不是 d
依赖的 placeholder 就不用加入到 `feed_dict` 中。

* * *

### **2.4 Variable**

变量 Variable，是维护图执行过程中的状态信息的. 需要它来保持和更新参数值，是需要动态调整的。 Tensor 和 Operation
都是一成不变的，而 Variable 是可以随着时间改变的，

    
    
    # 将 3 传递给 variable
    my_var = tf.Variable(3, name = "my_variable")
    

**Variables** 可以用在任何使用 tensor 的 Op 中，它当前的值就会被传递给使用它的 Op，

通常它的初始值是一些很大的 0 ，1， 或者随机值的 tensor， 建立这样的 tensor 可以用内置的 Op ： `tf.zeros(),
tf.ones(), tf.random_normal(), tf.random_uniform(), tf.truncated_normal()`,
只需要输入参数 shape，就可以得到需要的 tensor.

`tf.truncated_normal()` 比 random_normal 更常用， 它生成的值都会在距离平均值为两个标准偏差之内，这样可以避免
tensor 的某些值会与其他值有明显的不同.

    
    
    random_var = tf.Variable(tf.truncated_normal([2, 2]))
    

* * *

### **2.4.1 变量的初始化**

Variable 虽然也在 graph 中，但它的状态是由 Session 管理，所以就需要在一个 Session 里进行初始化，这样 Session
就可以追踪 Variable 的当前值， 由下面代码完成：就是 **将`tf.global_variables_initializer()` 这个 Op
传递给 `Session.run()`**：

    
    
    init = tf.global_variables_initializer()
    sess = tf.Session()
    sess.run(init)
    

如果只是想 **初始化 graph 中定义的 Variable 的一部分** ， 可以用
`tf.variables_initializer`，并向其传入想要初始化的变量列表，

    
    
    var1 = tf.Variable(0, name="initialize_me")
    var2 = tf.Variable(1, name="no_initialization")
    init = tf.variables_initializer([var1], name="init_var1")
    sess = tf.Session()
    sess.run(init)
    

* * *

### **2.4.2 改变变量的值**

**改变 Variable 的值** 可以用 `Variable.assign()`，它是个 Op，并且必须在一个 Session 中执行才会生效，

    
    
    my_var = tf.Variable(1)                # 建立变量 my_var 初始值为 1
    
    # 建立一个 op 每次执行时 对 my_var 乘以 2 
    my_var_times_two = my_var.assign(my_var * 2)
    
    init = tf.initialize_all_variables()
    sess = tf.Session()
    sess.run(init)
    

还有 **递增递减** 的方法：

    
    
    # 每次增加 1
    increment_step = global_step.assign_add(1)
    

因为每个 Sessions 都是独立维护 Variable 的值的，所以 **对同一个变量，不同的会话各自可以是各自的当前值，**

    
    
    my_var = tf.Variable(0)
    init = tf.initialize_all_variables()
    
    sess1 = tf.Session()
    sess2 = tf.Session()
    
    # 在 sess1 中初始化变量, 每次增加 5
    sess1.run(init)
    sess1.run(my_var.assign_add(5))            ## OUT: 5
    
    # 在 sess2 中初始化变量, 每次增加 2
    sess2.run(init)
    sess2.run(my_var.assign_add(2))         ## OUT: 2
    
    sess1.run(my_var.assign_add(5))            ## OUT: 10
    sess2.run(my_var.assign_add(2))            ## OUT: 4
    

如果想要 **重置 Variables 的值** ，只需要再次调用 `tf.global_variables_initializer()`，

    
    
    my_var = tf.Variable(0)                    # my_var 初始为 0
    init = tf.global_variables_initializer()
    
    sess = tf.Session()
    sess.run(init)
    
    sess.run(my_var.assign(10))                # my_var 变为 10
    
    # 将变量重置为 0
    sess.run(init)
    

* * *

### **2.4.3 变量的 Trainable 可训练性**

当用各种 Optimizer 训练机器学习模型时，Variable 的值就会随之改变， 当你不需要它被 Optimizer 改变时，可以将
trainable 设为 False，

    
    
    global_step = tf.Variable(0, dtype = tf.int32, trainable = False, name = "global_step")
    

* * *

接下来看 TensorBoard， 它是 TensorFlow
上一个非常酷的功能，我们都知道神经网络很多时候就像是个黑盒子，里面到底是什么样，是什么样的结构，是怎么训练的，可能很难搞清楚，而 TensorBoard
的作用就是可以把复杂的神经网络训练过程给可视化，可以更好地理解，调试并优化程序。

**\- 3. TensorBoard \- 3.1 name scopes**

### **3\. TensorBoard**

首先要建立一个 **summary.FileWriter** 对象，以后会用它来存储数据和统计 summary ，命名为 writer.
它有两个参数：第一个是 output directory，是 graph 的描述存储的地方，这里生成的文件会存到
`improved_graph`，第二个参数是 graph。

    
    
    writer = tf.summary.FileWriter('./improved_graph', graph)
    

**打开 TensorBoard 的方法** ：

打开 terminal 输入下面代码，logdir 为 SummaryWriter 中定义的图描述存储的地方：

    
    
    $ tensorboard --logdir="improved_graph"
    

确保当前的目录就是 `improved_graph` 建立的地方，

然后会提示：“Starting TensorBoard on port 6006” 然后打开浏览器，并输入
`http://localhost:6006`，就可以打开 TensorBoard 了。

点击 Graph 就可以看到图，每个结点都由 name 标记，点开结点可以看到和它们相连的其他结点，可以看出这个图和前面画的结构图一致，

![3aAhNV](https://images.gitbook.cn/3aAhNV.jpg)

建立完图之后，可以关闭 Session 和 summary.FileWriter:

    
    
    writer.close()
    sess.close()
    

* * *

### **3.1 name scopes**

本节用的例子都是非常简单的，在实际的模型中会有数百个结点，几百万的参数，为了管理这么复杂的模型，可以用 name scopes 来管理图，

有了 **name scopes** 可以在 TensorBoard 中看到更清晰明了的图，它可以把一组 Op 放到一个组块里，

也可以在 name scope 里面建立 scope，例如代码中的：

    
    
        with tf.name_scope("transformation"):                
    
            with tf.name_scope("input"):
                a = tf.placeholder(tf.float32, shape = [None], name = "input_placeholder_a")
    
            with tf.name_scope("intermediate_layer"):
                b = tf.reduce_prod(a, name = "product_b")        
                c = tf.reduce_sum(a, name = "sum_c")
    
            with tf.name_scope("output"):
                output = tf.add(b, c, name = "output")
    

* * *

好了，到此我们学习了 **TensorFlow 的代码结构** ，以及其中涉及的
**Graph，Session，Tensor，Placeholder，Variable，Feed，Fetch**
几大基础概念。我们在以后的每个代码中都会遇到它们，相信经过本节内容，以后的代码也会写的顺畅自如。

在看过每个概念的详细解释后，这时可以再去文章开头的整体代码处对照体会，加深理解。

关于 TensorFlow 的安装可以查看官网 https://www.tensorflow.org/install/，这里就不赘述了。

此外初学者可以先通过 Google 的 Colab 学习使用 TensorFlow，这是 Google 为了更好地传播机器学习创建的 Jupyter
notebook 环境，不需要任何设置，直接在云端运行。
https://colab.research.google.com/notebooks/welcome.ipynb

## 入群

![H1zaIq](https://images.gitbook.cn/H1zaIq.jpg)

扫码添加微信，回复【530】加入读者交流群。

