随着深度学习在语音、图像、自然语言等领域取得了广泛的成功，越来越多的企业、高校和科研单位开始投入大量的资源研发 AI
项目。同时，为了方便广大研发人员快速开发深度学习应用，专注于算法应用本身，避免重复造轮子的问题，各大科技公司先后开源了各自的深度学习框架，例如：TensorFlow（Google）、Torch/PyTorch（Facebook）、Caffe（BVLC）、CNTK（Microsoft）、PaddlePaddle（百度）等。

以上框架基本都是基于 Python 或者 C/C++ 开发的。而且很多基于 Python 的科学计算库，如 NumPy、Pandas
等都可以直接参与数据的建模，非常快捷高效。

然而，对于很多 IT 企业及政府网站，大量的应用都依赖于 Java 生态圈中的开源项目，如
Spring/Structs/Hibernate、Lucene、Elasticsearch、Neo4j 等。主流的分布式计算框架，如
Hadoop、Spark 都运行在 JVM 之上，很多海量数据的存储也基于 Hive、HDFS、HBase 这些存储介质，这是一个不容忽视的事实。

有鉴于此，如果有可以跑在 JVM 上的深度学习框架，那么不光可以方便更多的 Java/JVM 工程师参与到人工智能的浪潮中，更重要的是可以与企业已有的
Java 技术无缝衔接。无论是 Java EE 系统，还是分布式计算框架，都可以与深度学习技术高度集成。Deeplearning4j
正是具备这些特点的深度学习框架。

### Deeplearning4j 是什么

> [Deeplearning4j](https://deeplearning4j.org/) 是由美国 AI 创业公司
> [Skymind](https://skymind.ai/) 开源并维护的一个基于 Java/JVM 的深度学习框架。同时也是在 Apache
> Spark 平台上为数不多的，可以原生态支持分布式模型训练的框架之一。此外，Deeplearning4j 还支持多 GPU/GPU
> 集群，可以与高性能异构计算框架无缝衔接，从而进一步提升运算性能。在 2017 年下半年，Deeplearning4j 正式被 Eclipse
> 社区接收，同 Java EE 一道成为 Eclipse 社区的一员。

另外，就在今年的 4 月 7 号，Deeplearning4j 发布了最新版本
1.0.0-alpha，该版本的正式发布不仅提供了一系列新功能和模型结构，也意味着整个 Deeplearning4j 项目的趋于稳定和完善。

![enter image description
here](https://images.gitbook.cn/9928a020-f2e1-11e8-8d28-f50de28a2376)

Deeplearning4j 提供了对经典神经网络结构的支持，例如：

  * 多层感知机/全连接网络（MLP）
  * 受限玻尔兹曼机（RBM）
  * 卷积神经网络（CNN）及相关操作，如池化（Pooling）、解卷积（Deconvolution）、空洞卷积（Dilated/Atrous Convolution）等
  * 循环神经网络（RNN）及其变种，如长短时记忆网络（LSTM）、双向 LSTM（Bi-LSTM）等
  * 词/句的分布式表达，如 word2vec/GloVe/doc2vec 等

在最新的 1.0.0-alpha 版本中，Deeplearning4j 在开始支持自动微分机制的同时，也提供了对 TensorFlow
模型的导入，因此在新版本的 Deeplearning4j 中可以支持的网络结构将不再局限于自身框架。

DeepLerning4j 基于数据并行化理论，对分布式建模提供了支持（准确来说是基于参数同步机制的数据并行化，并在 0.9.0 版本后新增了
Gradients Sharing 的机制）。需要注意的是，Deeplearning4j 并没有创建自己的分布式通信框架，对于 CPU/GPU
集群的分布式建模仍然需要依赖 Apache Spark。在早期的版本中，Deeplearning4j 同样支持基于 MapReduce 的
Hadoop，但由于其在模型训练期间 Shuffle 等因素导致的迭代效率低下，加上 Spark
基于内存的数据存储模型的高效性，使得其在最近版本中已经停止了对 Hadoop 的支持。

此外，Apache 基金会下另一个分布式计算的顶级项目 Flink 正在积极考虑将 Deeplearning4j
进行集成，[详见这里](https://issues.apache.org/jira/browse/FLINK-5782)。

![enter image description
here](https://images.gitbook.cn/bfdc0900-fc28-11e8-8576-39c4102c68fe)

Deeplearning4j 生态圈中除了深度神经网络这个核心框架以外，还包括像 DataVec、ND4J、RL4J
等一些非常实用的子项目，下面就对这些子项目的主要功能模块做下介绍。

**1\. ND4J & LibND4J**

这两个子项目是 Deeplearning4j 所依赖的张量运算框架。其中，ND4J 提供上层张量运算的各种接口，而 LibND4J 用于适配底层基于
C++/Fortran 的张量运算库，如 OpenBLAS、MKL 等。

**2\. DataVec**

这是数据预处理的框架，该框架提供对一些典型非结构化数据（语音、图像、文本）的读取和预处理（归一化、正则化等特征工程常用的处理方法）。此外，对于一些常用的数据格式，如
JSON/XML/MAT（MATLAB 数据格式）/LIBSVM 也都提供了直接或间接的支持。

**3\. RL4J**

这是基于 Java/JVM 的深度强化学习框架，它提供了对大部分基于 Value-Based 强化学习算法的支持，具体有：Deep
Q-leaning/Dual DQN、A3C、Async NStepQLearning。

**4\. dl4j-examples**

这是 Deeplearning4j 核心功能的一些常见使用案例，包括经典神经网络结构的一些单机版本的应用，与 Apache Spark
结合的分布式建模的例子，基于 GPU 的模型训练的案例以及自定义损失函数、激活函数等方便开发者需求的例子。

**5\. dl4j-model-z**

顾名思义这个框架实现了一些常用的网络结构，例如：

  * ImageNet 比赛中获奖的一些网络结构 AlexNet/GoogLeNet/VGG/ResNet；
  * 人脸识别的一些网络结构 FaceNet/DeepFace；
  * 目标检测的网络结构 Tiny YOLO/YOLO9000。

在最近 Release 的一些版本中，dl4j-model-z 已经不再作为单独的项目，而是被纳入 Deeplearning4j
核心框架中，成为其中一个模块。

**6\. ScalNet**

这是 Deeplearning4j 的 Scala 版本，主要是对神经网络框架部分基于 Scala 语言的封装。

对这些生态圈中子项目的使用案例，我会在后续的文章中详细地介绍，在这里就不赘述了。

### 为什么要学习 Deeplearning4j

在引言中我们谈到，目前开源的深度学习框架有很多，那么选择一个适合工程师自己、同时也可以达到团队业务要求的框架就非常重要了。在这个部分中，我们将从计算速度、接口设计与学习成本，和其他开源库的兼容性等几个方面，给出
Deeplearning4j 这个开源框架的特点及使用场景。

#### ND4J 加速张量运算

JVM 的执行速度一直为人所诟病。虽然 Hotspot 机制可以将一些对运行效率有影响的代码编译成 Native Code，从而在一定程度上加速 Java
程序的执行速度，但毕竟无法优化所有的逻辑。另外，Garbage
Collector（GC）在帮助程序员管理内存的同时，其实也束缚了程序员的手脚，毕竟是否需要垃圾回收并不是程序员说了算；而在其他语言如 C/C++
中，我们可以 free 掉内存块。

对于机器学习/深度学习来说，优化迭代的过程往往非常耗时，也非常耗资源，因此尽可能地加速迭代过程十分重要。运算速度也往往成为评价一个开源库质量高低的指标之一。鉴于
JVM 自身的局限性，Deeplearning4j 的张量运算通过 ND4J 在堆外内存（Off-Heap Memory/Direct
Memory）上进行，[ 详见这里](https://deeplearning4j.org/memory)。大量的张量运算可以依赖底层的 BLAS 库（如
OpenBLAS、Intel MKL），如果用 GPU 的话，则会依赖 CUDA/cuBLAS。由于这些 BLAS 库多数由 Fortran 或 C/C++
写成，且经过了细致地优化，因此可以大大提高张量运算的速度。对于这些张量对象，在堆上内存（On-Heap
Memory）仅存储一个指针/引用对象，这样的处理也大大减少了堆上内存的使用。

#### High-Level 的接口设计

对于大多数开发者而言，开源库的学习成本是一个不可回避的问题。如果一个开源库可以拥有友好的接口、详细的文档和案例，那么无疑是容易受人青睐的。这一点对于初学者或者致力于转型
AI 的工程师尤为重要。

神经网络不同于其他传统模型，其结构与复杂度变化很多。虽然在大多数场景下，我们会参考经典的网络结构，如 GoogLeNet、ResNet
等。但自定义的部分也会很多，往往和业务场景结合得更紧密。为了使工程师可以快速建模，专注于业务本身和调优，Deeplearning4j
对常见的神经网络结构做了高度封装。以下是声明卷积层 + 池化层的代码示例，以及和 Keras 的对比。

**Keras 版本**

    
    
    model = Sequential()
    model.add(Convolution2D(nb_filters, (kernel_size[0], kernel_size[1]))) #卷积层  
    model.add(Activation('relu')) #非线性变换  
    model.add(MaxPooling2D(pool_size=pool_size)) #池化层  
    

**Deeplearning4j 版本**

    
    
    MultiLayerConfiguration conf = new NeuralNetConfiguration.Builder()
    //...
    .layer(0, new ConvolutionLayer.Builder(5, 5)    //卷积层
                            .nIn(nChannels)
                            .stride(1, 1)
                            .nOut(20)
                            .activation(Activation.RELU)    //非线性激活函数
                            .build())
    .layer(1, new SubsamplingLayer.Builder(PoolingType.MAX)    //最大池化
                            .kernelSize(2,2)
                            .stride(2,2)
                            .build())
    

可以看到，Deeplearning4j 和 Keras 很相似，都是以 Layer 为基本模块进行建模，这样的方式相对基于 OP
的建模方式更加简洁和清晰。所有的模块都做成可插拔的，非常灵活。以 Layer
为单位进行建模，使得模型的整个结构高度层次化，对于刚接触深度神经网络的开发人员，可以说是一目了然。除了卷积、池化等基本操作，激活函数、参数初始化分布、学习率、正则化项、Dropout
等 trick，都可以在配置 Layer 的时候进行声明并按需要组合。

当然基于 Layer 进行建模也有一些缺点，比如当用户需要自定义
Layer、激活函数等场景时，就需要自己继承相关的基类并实现相应的方法。不过这些在官网的例子中已经有一些参考的
Demo，如果确实需要，开发人员可以参考相关的例子进行设计，[详见这里](https://github.com/deeplearning4j/dl4j-examples/tree/master/dl4j-examples/src/main/java/org/deeplearning4j/examples/misc)。

此外，将程序移植到 GPU 或并行计算上的操作也非常简单。如果用户需要在 GPU 上加速建模的过程，只需要加入以下逻辑声明 CUDA 的环境实例即可。

**CUDA 实例声明**

    
    
    CudaEnvironment.getInstance().getConfiguration()
                .allowMultiGPU(true)
                .setMaximumDeviceCache(10L * 1024L * 1024L * 1024L)
                .allowCrossDeviceAccess(true);
    

ND4J 的后台检测程序会自动检测声明的运算后台，如果没有声明 CUDA 的环境实例，则会默认选择 CPU 进行计算。

如果用户需要在 CPU/GPU 上进行并行计算，则只需要声明参数服务器的实例，配置一些必要参数即可。整体的和单机版的代码基本相同。

**参数服务器声明**

    
    
    ParallelWrapper wrapper = new ParallelWrapper.Builder(model)
                .prefetchBuffer(24)
                .workers(8)
                .averagingFrequency(3)
                .reportScoreAfterAveraging(true)
                .useLegacyAveraging(true)
                .build();
    

参数服务器的相关参数配置在后续的课程中会有介绍，这里就不再详细说明了。

#### 友好的可视化页面

为了方便研发人员直观地了解神经网络的结构以及训练过程中参数的变化，Deeplearning4j 提供了可视化页面来辅助开发。需要注意的是，如果需要使用
Deeplearning4j 的可视化功能，需要 JDK 1.8 以上的支持，同时要添加相应的依赖：

    
    
    <dependency>
        <groupId>org.deeplearning4j</groupId>
        <artifactId>deeplearning4j-ui_${scala.binary.version}</artifactId>
        <version>${dl4j.version}</version>
    </dependency>
    

并且在代码中添加以下逻辑：

    
    
    //添加可视化页面监听器
    UIServer uiServer = UIServer.getInstance();
    StatsStorage statsStorage = new InMemoryStatsStorage();
    uiServer.attach(statsStorage);
    model.setListeners(new StatsListener(statsStorage));
    

训练开始后，在浏览器中访问本地 9000 端口，默认会跳转到 Overview 的概览页面。我们可以依次选择查看网络结构的页面（Model
页面）和系统页面（System），从而查看当前训练的模型以及系统资源的使用情况。详情见下图：

![enter image description
here](https://images.gitbook.cn/94aab060-f9f8-11e8-98b8-21d1b727d9e8)

![enter image description
here](https://images.gitbook.cn/a87dc320-f9f8-11e8-98b8-21d1b727d9e8)

![enter image description
here](https://images.gitbook.cn/72fb5640-f9f8-11e8-98b8-21d1b727d9e8)

Overview 页面主要会记录模型在迭代过程中 Loss 收敛的情况，以及记录参数和梯度的变化情况。根据这些信息，我们可以判断模型是否在正常地学习。从
Model 页面，我们则可以直观地看到目前网络的结构，比如像图中有 2 个卷积层 + 2 个池化层 + 1 个全连接层。而在 System
页面中我们可以看到内存的使用情况，包括堆上内存和堆外内存。

Deeplearning4j
提供的训练可视化页面除了可以直观地看到当前模型的训练状态，也可以基于这些信息进行模型的调优，具体的方法在后续课程中我们会单独进行说明。

#### 兼容其他开源框架

正如在引言中提到的，现在深度神经网络的开源库非常多，每个框架都有自己相对擅长的结构，而且对于开发人员，熟悉的框架也不一定都相同。因此如果能做到兼容其他框架，那么无疑会提供更多的解决方案。

Deeplearning4j 支持导入 Keras 的模型（在目前的 1.0.0-alpha 版本中，同时支持 Keras 1.x/2.x）以及
TensorFlow 的模型。由于 Keras 自身可以选择 TensorFlow、Theano、CNTK 作为计算后台，再加上第三方库支持导入 Caffe
的模型到 Keras，因此 Keras 已经可以作为一个“胶水”框架，成为 Deeplearning4j 兼容其他框架的一个接口。

![enter image description
here](https://images.gitbook.cn/4eb24900-f9f9-11e8-98b8-21d1b727d9e8)

兼容其他框架（准确来说是支持导入其他框架的模型）的好处有很多，比如说：

  * 扩展 Model Zoo 中支持的模型结构，跟踪最新的成果；
  * 离线训练和在线预测的开发工作可以独立于框架进行，减少团队间工作的耦合。

#### Java 生态圈助力应用的落地

Deeplearning4j 是跑在 JVM 上的深度学习框架，源码主要是由 Java 写成，这样设计的直接好处是，可以借助庞大的 Java
生态圈，加快各种应用的开发和落地。Java 生态圈无论是在 Web 应用开发，还是大数据存储和计算都有着企业级应用的开源项目，例如我们熟知的 SSH
框架、Hadoop 生态圈等。

Deeplearning4j 可以和这些项目进行有机结合，无论是在分布式框架上（Apache Spark/Flink）进行深度学习的建模，还是基于 SSH
框架的模型上线以及在线预测，都可以非常方便地将应用落地。下面这两张图就是 Deeplearning4j + Tomcat + JSP
做的一个简单的在线图片分类的应用。

![enter image description
here](https://images.gitbook.cn/80e27da0-f9f9-11e8-98b8-21d1b727d9e8) ![enter
image description
here](https://images.gitbook.cn/87a5c0c0-f9f9-11e8-98b8-21d1b727d9e8)

总结来说，至少有以下 4 种场景可以考虑使用 Deeplearning4j：

  * 如果你身边的系统多数基于 JVM，那么 Deeplearning4j 是你的一个选择；
  * 如果你需要在 Spark 上进行分布式深度神经网络的训练，那么 Deeplearning4j 可以帮你做到；
  * 如果你需要在多 GPU/GPU 集群上加快建模速度，那么 Deeplearning4j 也同样可以支持；
  * 如果你需要在 Android 移动端加入 AI 技术，那么 Deeplearning4j 可能是你最方便的选择之一。

以上四点，不仅仅是 Deeplearning4j 自身的特性，也是一些 AI 工程师选择它的理由。

虽然 Deeplearning4j 并不是 GitHub 上 Fork 或者 Star 最多的深度学习框架，但这并不妨碍其成为 AI 工程师的一种选择。就
Skymind 官方发布的信息看，在美国有像 IBM、埃森哲、NASA 喷气推进实验室等多家明星企业和实验机构，在使用 Deeplearning4j
或者其生态圈中的项目，如 ND4J。算法团队结合自身的实际情况选择合适的框架，在多数时候可以做到事半功倍。

### Deeplearning4j 的最新进展

在这里，我们主要介绍下 Deeplearning4j 最新的一些进展情况。

#### 版本

目前 Deeplearning4j 已经来到了 1.0.0-beta3 的阶段，马上也要发布正式的 1.0.0 版本。本课程我们主要围绕 0.8.0 和
1.0.0-alpha 展开（1.0.0-beta3 核心功能部分升级不大），这里罗列下从 0.7.0 版本到 1.0.0-alpha
版本主要新增的几个功能点：

  * Spark 2.x 的支持（>0.8.0）
  * 支持迁移学习（>0.8.0）
  * 内存优化策略 Workspace 的引入（>0.9.0）
  * 增加基于梯度共享（Gradients Sharing）策略的并行化训练方式（>0.9.0）
  * LSTM 结构增加 cuDNN 的支持（>0.9.0）
  * 自动微分机制的支持，并支持导入 TensorFlow 模型（>1.0.0-alpha）
  * YOLO9000 模型的支持（>1.0.0-aplpha）
  * CUDA 9.0 的支持（>1.0.0-aplpha）
  * Keras 2.x 模型导入的支持（>1.0.0-alpha）
  * 增加卷积、池化等操作的 3D 版本（>1.0.0-beta）

除此之外，在已经提及的 Issue 上，已经考虑在 1.0.0 正式版本中增加对 YOLOv3、GAN、MobileNet、ShiftNet
等成果的支持，进一步丰富 Model Zoo 的直接支持范围，满足更多开发者的需求。详见
[GAN](https://github.com/deeplearning4j/deeplearning4j/issues/5005)、[MobileNet](https://github.com/deeplearning4j/deeplearning4j/issues/4995)、[YOLOv3](https://github.com/deeplearning4j/deeplearning4j/issues/4986)、[ShiftNet](https://github.com/deeplearning4j/deeplearning4j/issues/5144)。

进一步的进展情况，可以直接跟进每次的
[releasenotes](https://deeplearning4j.org/releasenotes)，查看官方公布的新特性和已经修复的 Bug
情况。

#### 社区

Deeplearning4j 社区目前正在进一步建设和完善中，在社区官网上除了介绍 Deeplearning4j
的基本信息以外，还提供了大量有关神经网络理论的资料，方便相关研发人员的入门与进阶。Deeplearning4j 社区在 Gitter
上同时开通了英文/中文/日文/韩文频道，开发人员可以和核心源码提交者进行快速的交流以及获取最新的信息。

![enter image description
here](https://images.gitbook.cn/c3335270-f2f5-11e8-8d28-f50de28a2376)

  * Deeplearning4j 的 GitHub 地址，[详见这里](https://github.com/deeplearning4j)；
  * Deeplearning4j 社区官网，[详见这里](https://deeplearning4j.org/)；
  * Deeplearning4j 英文 Gitter Channel，[详见这里](https://gitter.im/deeplearning4j/deeplearning4j)；
  * Deeplearning4j 中文 Gitter Channel，[详见这里](https://gitter.im/deeplearning4j/deeplearning4j/deeplearning4j-cn)；
  * Deeplearning4j 官方 QQ 群， **289058486** 。

### 关于本课程

最后我们简单介绍一下本课程涉及的内容以及一些学习建议。

本课程主要面向深度学习/深度神经网络的企业研发人员、高校以及研究机构的研究人员。同时，对于致力于转型 AI 开发的 Java
工程师也会有很大的帮助。对于那些希望构建基于 JVM 上的 AI 项目的研发人员也有着一定的参考价值。

本课程会围绕 Deeplearning4j
框架，并结合深度学习在图像、语音、自然语言处理等领域的一些经典案例（如图像的分类、压缩、主体检测，文本分类、序列标注等），给出基于
Deeplearning4j 的解决方案。此外，对于具体的实例，我们将分别介绍建模的步骤及模型的部署上线这一全栈开发的流程。我们会结合
Deeplearning4j 的一些特性，分别介绍在单机、多 CPU/GPU、CPU/GPU 集群上进行建模的步骤，以及如何与 Java Web
项目进行整合实现 AI 应用的落地。

本课程力求做到对 Deeplearning4j 及其生态圈进行详细的介绍，包括对 Deeplearning4j 现在已经支持的迁移学习（Transfer
Learning）和强化学习（Reinforcement Learning）进行实例应用的介绍。同时希望通过本课程帮助真正有需要的研发人员落地 AI
项目或者提供一种可行的解决方案，尽量做到内容覆盖的全面性和语言的通俗易懂。

此外，我们给出一些学习本课程的建议：在学习本课程前，希望学员有一定的 Java
工程基础，以及对机器学习/深度学习理论的了解。如果对优化理论、微分学、概率统计有一定的认识，那么对理解神经网络的基础理论（如 BP 算法）将大有裨益。

最后，下面是本课程的大纲思维导图，方便大家参考学习。顺带跟大家说一声，购买课程的读者可以加入我们成立的 DL4J
课程微信交流群，我会抽出时间，不定期回复读者的疑问。预祝大家快速上手 DL4J 开发！

![enter image description
here](https://images.gitbook.cn/5c5211c0-fe19-11e8-9206-ebae2af18c23)

