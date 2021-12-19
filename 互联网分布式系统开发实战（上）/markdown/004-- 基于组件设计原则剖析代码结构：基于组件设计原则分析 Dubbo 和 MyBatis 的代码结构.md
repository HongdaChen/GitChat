上一节中我们介绍了组件设计原则的相关定义，那么如何使用这些原则来指导我们的开发工作呢？任何技术方案的落地都需要同时兼顾定性和定量指标，本节将介绍组件设计原则相关的量化标准以及针对这些量化指标的测量工具。

### 组件设计原则背后的量化标准和测量工具

组件量化标准有两大维度，分别是稳定度和抽象度。这两个维度分别对应组件设计原则中的稳定抽象原则和稳定依赖运作。让我们先来看稳定度。

#### 量化组件的稳定度

在组件设计原则中我们提到一个稳定抽象原则，即组件的抽象程度应该与其稳定程度保持一致。组件的稳定度可以用以下公式来衡量：

$$ I = Ce / (Ca + Ce) $$ 其中 Ca 代表向心耦合（Afferent Coupling），表示依赖该组件的外部组件数量。而 Ce
代表离心耦合（Efferent Coupling），表示被该组件依赖的外部组件的数量。I 代表
Instability，即不稳定性，显然它的值处于[0，1]之间。

针对上一小节介绍的 X 和 Y 两个组件，我们可以使用该公式做一个简单计算。不难得出组件 X 的 Ce=0（因为它没有依赖任何外部组件），所以不稳定性
I=0，说明它非常稳定。相反，组件 Y 的 Ce=3，Ca=0（因为没有任何组件依赖它），所以它的不稳定性 I=1，说明它非常不稳定。

组件之间存在一个依赖链，稳定性在该依赖链上具有传递性。下图展示的是一种更常见的场景，沿着依赖的方向，组件的不稳定性应该逐渐降低，稳定性应该逐渐升高。如果已经处于稳定状态的组件就不应该去依赖处于不稳定状态的组件。

![03.01](https://images.gitbook.cn/2020-05-25-052713.png)

#### 量化组件的抽象度

另一方面，组件的抽象度也同样存在类似的计算公式：

$$ A = AC / CC $$ 其中 A 代表抽象度（Abstractness）。AC（Abstract Class）表示组件中抽象类的数量，而
CC（Concrete Class）表示组件中所有类的总和，这样通过对比 AC 和 CC 就能简单得出该组件的抽象度。

正如上图所示，一个系统中多数的组件位于依赖链的中间，也就是说它们即具备一定的稳定性也表现出一定的抽象度。如果一个组件的稳定度和抽象度都是
1，意味着该组件里面全是抽象类且没有任何组件依赖它，那么这个组件就没有任何用处。相反，如果一个组件稳定度和抽象度都是
0，那么意味着这个组件不断在变化，不具备维护性，这也是我们不想设计的组件。所以，在稳定度和抽象度之间我们应该保持一种平衡，下图中中间的那个线就是平衡线。在有些资料中，这条平衡线有一个专业的名称，即主序列（Main
Sequence），如下图所示。

![03.02](https://images.gitbook.cn/2020-05-25-052715.png)

我们用距离（Distance）的概念来量化这种平衡，距离的计算公式：

$$ D = abs(1 - I - A) * sin(45) $$ 距离的图形化表示参考下图。

![03.03](https://images.gitbook.cn/2020-05-25-052716.png)

使用这个量化标准，可以全面分析一个代码结构的设计与主序列之间的一致性。当阅读某一个框架的代码时，这种分析非常有助于我们确定框架的设计者如何确定哪些包更容易维护，哪些包对变化则不那么敏感。

#### 量化稳定度和抽象度测量工具

有了量化标准，我们就可以使用它们在做具体的测量。这里介绍一款组件依赖关系分析的利器：JDepend。JDepend 是用来评价 Java
代码质量的优秀工具，它遍历 Java 类的文件目录，以 Java
包为单位，为每一个包自动生成包的依赖程度、稳定性、可靠度等的评价报告。根据这些报告，我们可以得到包之间的依赖关系，并分析出包的稳定程度、抽象程度、是否存在循环依赖关系等。这些报告中的各项指标与上一节中介绍的组件设计量化标准保持一致。

使用 JDepend 时，我们一般加载它为 Eclipse 提供的插件（可从
http://andrei.gmxhome.de/jdepend4eclipse/links.html 下载），也可以在 Eclipse 市场中直接搜索
JDepend 插件进行安装。

安装完 JDepend 插件之后，在 Eclipse 中，当我们在待分析项目的 src 图标上点击右键时，会看到新增了“Run JDepend
Analysis”菜单，直接执行安命令，就可以在打开的 JDepend 视图中看到分析结果。

### 组件设计原则与代码结构：Dubbo VS Mybatis

接下来我们就使用 JDepend 来分析 Dubbo 和 Mybatis 这两个开源框架的代码结构。

#### Dubbo 框架的稳定度和抽象度

我们首先来看 Dubbo 框架，Dubbo 在设计过程中同样采用的是稳定抽象和稳定依赖等原则。我们来回顾 Dubbo
的各个核心包，它们的名称和简要描述如下所示：

  * dubbo-common：公共逻辑模块，包括 Util 类和通用模型。

  * dubbo-remoting：远程通讯模块，内部包含了自定义 Dubbo 协议的实现

  * dubbo-rpc：远程调用模块，抽象了各种协议并实现了动态代理，针对普通的一对一 RPC 调用，不包含集群的管理

  * dubbo-cluster：集群模块，负责将多个服务提供方伪装为一个提供方，包括负载均衡、集群容错和路由等功能

  * dubbo-registry：注册中心模块，提供对各种注册中心的抽象。

  * dubbo-monitor：监控模块，统计服务调用次数和调用时间，并提供调用链跟踪的服务。

  * dubbo-config：配置模块，作为 Dubbo 对外的 API，开发人员通过配置模块隐藏了 Dubbo 内部的所有细节。

  * dubbo-container：容器模块，以简单的 Main 方法加载 Spring 启动。

在上一篇中，我们就给出了 Dubbo 中各个包之间的依赖关系，除了 dubbo.common 通用工具包之外，处于依赖关系底层的
dubbo.remoting 包和 dubbo.rpc 包是整个框架中的高层抽象。接下来我们通过 JDepend 来尝试对 Dubbo
中包结构进行量化分析。

我们来看一下整个依赖关系中居于中心位置的 dubbo.rpc，可以看到如下图所示的分析结果。JDepend
给出了四个子页面，分别是所选中的对象、存在循环依赖关系的包、多依赖的包和被依赖的包。从图中，我们看到具体类（CC）、抽象类（AC）、向心耦合（Ca）、离心耦合（Ec）、不稳定性（I）、抽象度（A）和距离（D）等组价设计原则中所介绍的指标数量，同时还使用“Cycle!”用来标识是否包结构是否存在循环依赖。

![03.04](https://images.gitbook.cn/2020-05-25-052720.png)

伴随着可视化界面，JDepend 还提供了完整的结果描述。由于内容比较多，这里以 com.alibaba.dubbo.rpc.protocol
包为例，截取部分数据供大家参考。

    
    
    Stats:
        Total Classes: 10
        Concrete Classes: 6
        Abstract Classes: 4
    
        Ca: 1
        Ce: 7
    
        A: 0.4
        I: 0.88
        D: 0.27
    
    Abstract Classes:
        com.alibaba.dubbo.rpc.protocol.AbstractExporter
        com.alibaba.dubbo.rpc.protocol.AbstractInvoker
        com.alibaba.dubbo.rpc.protocol.AbstractProtocol
        com.alibaba.dubbo.rpc.protocol.AbstractProxyProtocol
    
    Concrete Classes:
        com.alibaba.dubbo.rpc.protocol.AbstractProxyProtocol$1
        com.alibaba.dubbo.rpc.protocol.AbstractProxyProtocol$2
        com.alibaba.dubbo.rpc.protocol.InvokerWrapper
        com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper
        com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1
        com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper
    
    Depends Upon:
        com.alibaba.dubbo.common
        com.alibaba.dubbo.common.extension
        com.alibaba.dubbo.common.logger
        com.alibaba.dubbo.common.utils
        com.alibaba.dubbo.rpc
        com.alibaba.dubbo.rpc.listener
        com.alibaba.dubbo.rpc.support
    
    Used By:
        com.alibaba.dubbo.rpc.support
    

最后，JDepend 还为我们自动生成了主序列图以及各个包在该图中的分布情况。对于 com.alibaba.dubbo.rpc 包而言，内部包含 7
个子包，所以在该图上一共有 7 个点。点击某个点，可以看到该点所代表的包结构中的不稳定性、抽象度和主序列之间的距离值，如图中我们看到的就是
com.alibaba.dubbo.rpc.protocol
包的相关数据。图中所分布点分为三种颜色，绿色集中主序列线附件，代表在不稳定性和抽象度之间达成了比较好的一种平衡。黑色点位则相对差一下，如果如果红色点位，则表示设计上出现了问题，需要引起我们的注意。

![03.05](https://images.gitbook.cn/2020-05-25-052722.png)

#### Mybatis 框架的稳定度和抽象度

接下来我们分析 Mybatis 框架。Mybatis 的核心包比较多，我们同样找位于依赖关系中间位置的 session
包做相同的分析，得到的量化指标结果分别如下。

    
    
    Stats:
        Total Classes: 22
        Concrete Classes: 16
        Abstract Classes: 6
    
        Ca: 1
        Ce: 42
    
        A: 0.27
        I: 0.98
        D: 0.25
    

从指标结果上讲，Mybatis 中的 org.apache.ibatis.session 包与 Dubbo 中的
com.alibaba.dubbo.rpc.protocol 包相差不多。关于 Mybatis 中其他包的结果读者可以自行尝试。

### 面试题分析

#### 如何对一个框架的代码设计质量进行量化？有什么方法和工具？

  * 考点分析

该问题考查代码的质量，代码质量是一个无法回避的话题，也是一个好的面试点。而量化代码质量则更加是日常开发过程中的一个难点和痛点。这个问题也可能会有更加宽泛的提问方法，即如何评估代码的质量。

  * 解题思路

面对这个问题，同样也有很多思路，但很少会直接涉及到量化的过程。本文中介绍的量化方法可以让面试者从容应对面试官的这类问题，一方面，我们介绍了一套完整的理论体系来指导我们评估代码的质量。另一方面，通过
JDepend 等工具，评估代码的工作就可以做到自动化，并时刻进行改进。这些内容对于这种比较宽泛的话题显然能够起到非常好的回答效果。

  * 本文内容与建议回答

本文中基于组件的稳定度以及抽象度的量化计算方法，以及所使用的 JDepend 工具可以直接作为问题的答案。同时，通过列举 Dubbo 和 Mybatis
两个框架的部分组件量化数据，可以很好为理论体系提供案例支持。

#### Dubbo 中包含哪些核心的组件包？相互之间有什么依赖关系？

  * 考点分析

这个问题直接指向了 Dubbo 框架的包结构，因为 Dubbo
框架的包结构数量不大，且有很明显的分层和依赖关系，以笔者的真实经历，这个问题在面试场景中是有机会遇到的。

  * 解题思路

我们对 Dubbo 中核心的几个组件包的名称、定位、划分的依据、相互之前的依赖关系等还是需要了然于胸，尤其是远程通信 dubbo-remoting
和远程调用 dubbo-rpc 这两个核心模块。

  * 本文内容与建议回答

这个问题同时涉及到本文和上一篇中的内容，我们给出了 Dubbo 框架中八大核心包结构，以及基于组件图所描述的相互依赖关系，需要面试者进行记忆并描述。

### 日常开发技巧

本文给出了一种非常有效且具备较高操作性的方法来帮忙大家衡量和优化自己的代码。我们可以通过应用
JDepend，对自己所编写的代码进行评估得到量化数据。针对量化数据不合理的模块和组件，结合组件设计原则来进行重构。这种做法会倒逼开发人员时刻关注组件级别的代码设计和实现质量，也是加强团队开发能力的一个有效抓手。

### 小结与预告

本文对组件设计原则背后的量化标准进行了阐述，并给出了具体的测量工具和方法。同时，将这些量化标准和测试工具应用与 Dubbo 和 Mybatis
这两个开源框架，并对结果进行了分析。

大家注意到组件设计原则中还有一个无环依赖原则没有提到，下一篇中我们将基于这一原则指导我们日常的开发工作。

### 分享交流

为了方便与作者交流与学习，GitChat 编辑团队组织了一个专栏读者交流群，添加小助手，回复关键字【7525】给小助手伽利略获取入群资格。

![R2Y8ju](https://images.gitbook.cn/2020-05-25-WechatIMG17.jpeg)

