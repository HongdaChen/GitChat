在上一篇中，我们讨论了 Dubbo 框架的架构演进过程，我们看到 Dubbo
框架按照五大步骤实现从简单到复杂、从核心功能到辅助机制逐步实现和完善的设计过程。前面我们给出了这五大步骤中的前两步，今天我们继续介绍后续的三个步骤。

### Cluster：负载均衡和集群容错机制

Dubbo 中的 Cluster 主要完成了两部分工作，一个是负载均衡，一个是集群容错。

我们在上一篇中提到，在 Dubbo 中 Invoker 是一个核心模型，可能是一个远程的实现，也可能一个集群实现。当 Invoker
代表集群实现时，获取具体某一个调用结果的过程就涉及到负载均衡。在 Dubbo 中，负载均衡功能由 LoadBalance 接口进行定义，如下所示。

    
    
    public interface LoadBalance {
    
        <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
    
    }
    

可以看到 LoadBalance 接口只有一个方法，即在一批 Invoker 列表中选择其中一个 Invoker
进行返回。显然，不同的负载均衡策略会导致不同的选择结果。Dubbo 中 LoadBalance 的实现主要有以下四种策略，如下图所示。

![](https://images.gitbook.cn/2020-05-25-052752.png)

诸如随机（Random）和轮询（RoundRobin）算法属于负载均衡的静态策略，实现比较简单。而最少活跃调用数（LeastActive）和一致性
Hash（Consistent Hash）算法属于动态策略，实现相对比较复杂。

我们再来看看集群容错。Dubbo 中的 Cluster 类定义如下：

    
    
    public interface Cluster {
    
        <T> Invoker<T> join(Directory<T> directory) throws RpcException;
    
    }
    

我们可以把 Cluster 可以看做是工厂类，将目录 Directory 下的多个 Invoker 合并成一个统一的 Invoker，基于不同集群策略的
Cluster 会创建不同的 Invoker。这里的 Directory 我们暂时不做展开，目前只需要知道它代表多个 Invoker, 可以简单看成是
List。Cluster 对上层透明，包含集群的容错机制。Dubbo 提供了多种 Cluster 实现方案，默认为
FailoverCluster。Cluster 的实现类如下图所示。

![](https://images.gitbook.cn/2020-05-25-052755.png)

以 FailoverCluster 为例，FailoverCluster 即失败转移集群，当出现失败将重试其它服务器，其代码如下。

    
    
    public class FailoverCluster implements Cluster {
        public final static String NAME = "failover";
    
        public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
            return new FailoverClusterInvoker<T>(directory);
        }
    }
    

FailoverCluster 的 join()方法返回的是一个 FailoverClusterInvoker，而其他 Cluster 也会返回对应的
ClusterInvoker。Dubbo 中包含了如下图所示的 ClusterInvoker：

![](https://images.gitbook.cn/2020-05-25-052756.png)

负载均衡和集群容错都是 Cluster 的核心功能，我们会在后续剖析分布式服务组件的课程中对它们的实现机制做源码分析。

### Remoting：自定义网络通信协议

Dubbo 的 remoting 模块是远程通信模块，相当于 Dubbo 自定义协议的实现，是一个为 Dubbo 项目处理底层网络通信的层。在 Dubbo
五个循序渐进的架构演进步骤中，这个环节在实现了网络通信的底层机制之外，还提供了一个 Dubbo
自定义的传输协议，所以从架构演进过程来看，这是一个具有较大扩展性的组件，我们从系统架构设计角度出发做一些参考。

引用 Dubbo 官网中的说明，remoting 模块中涉及的基础接口如下图所示。对于自定义协议而言，Dubbo 协议主要是在 TCP
协议的基础上添加了自定义的消息协议头（即图中的 Header 部分），解决 TCP 的粘包拆包问题。

![06.04](https://images.gitbook.cn/2020-05-25-052757.jpg)![](images\\06/06.04.jpg)

Dubbo 中通过 Codec 接口定义了消息协议头的编码解码过程，这里列出了新版的 Codec2 接口的代码定义。

    
    
    public interface Codec2 {
        void encode(Channel channel, ChannelBuffer buffer, Object message) throws IOException;
    
        Object decode(Channel channel, ChannelBuffer buffer) throws IOException;
    
        enum DecodeResult {
            NEED_MORE_INPUT, SKIP_SOME_INPUT
        }
    }
    

Codec2 接口的实现类层次结构如下图所示，这里的每一个实现都负责在不同层次上处理消息的编解码。例如，Transport 层负责网络传输，提取了业界主流的
mina 和 netty 作为统一接口，而 Exchange 则用于信息交换，封装了请求-响应模式。

![](https://images.gitbook.cn/2020-05-25-052759.png)

### Registry 和 Monitor：服务治理和监控辅助功能

作为一个基本的 RPC 框架而言，前面的各个步骤已经足够支撑 Dubbo 的基本功能。但在 Dubbo
架构的演进过程中，最后一个步骤还为它添加了注册中心（Registry）和服务监控（Monitor）功能。

注册中心的引入为 Dubbo 添加了服务注册（Registration）和服务发现（Discovery）机制，从而使得 Dubbo 从一个 RPC
框架上升到一个分布式服务框架（一定程度上也可以称之为微服务框架）。在 Dubbo 中，RegistryService
是对注册中心服务的抽象，该接口定义如下：

    
    
    public interface RegistryService {
        void register(URL url);
        void unregister(URL url);
        void subscribe(URL url, NotifyListener listener);
        void unsubscribe(URL url, NotifyListener listener);
        List<URL> lookup(URL url);
    }
    

从接口定义中可以看到所提供的方法包括注册和取消注册、订阅和取消订阅以及查找服务。而 Registry 接口继承了 RegistryService
接口并提供了如下所示的实现类，可以看到 Dubbo 分别基于 simple、multicast、redis 和 zookeeper
这四种模式提供了多种注册中心实现方式（默认使用的是 zookeeper）。

![06.06](https://images.gitbook.cn/2020-05-25-052800.png)

注册中心是一个复杂而重要的主题，在目前的分布式服务架构以及微服务架构中扮演着重要角色，我们将在剖析微服务架构组件的课程中对其进行详细展开。

另一方面，Dubbo 还提供了服务监控机制，并通过 MonitorService 接口实现了服务运行时的数据采集，如下所示。

    
    
    public interface MonitorService {
    
        void collect(URL statistics);
    
        List<URL> lookup(URL query);
    }
    

Dubbo 服务监控不是本课程的重点，不做具体展开。

### 整合：架构演进的结果

通过上一篇所提到的五个步骤，我们从架构演进过程这一特定角度剖析了 Dubbo 框架的代码结构。作为总结，我们从一个统一视图出发对整个过程做一个整合，见下图。

![06.07](https://images.gitbook.cn/2020-05-25-052802.png)

通过上图，我们把 Dubbo 中各个核心组件分成三大部分。最底部的 Remoting 部分主要完成网络通信，包含 Exchange、Transport 和
Serialize 组件，其中 Exchange 和 Transport 组件的功能在前面的内容中已有所提及，而 Serialize
主要完成数据的序列化和反序列化过程。上图中的中间部分称之为核心（Core）部分，包含了
Protocal、Proxy、Cluster、Registry、Monitor 等组件，这些组件也都在介绍 Dubbo
框架演进过程中做了介绍。而图中最上面的 Business 部分代表的是业务服务。

在本节最后，我们结合《基于组件设计原则剖析代码结构：基于组件设计原则分析 Dubbo 和 Mybatis 的代码结构》一文中的内容对 Dubbo
中包结构与代码层次之间的关系做简单梳理，我们在原始的包结构中分别添加了对应的分层组件，如下图所示。

![](https://images.gitbook.cn/2020-05-25-052803.png)

在上图中，我们不难看出：

  * Protocol 层和 Proxy 层放在 rpc 包中，构成 RPC 架构的核心，不考虑集群环境，可以只使用这两层完成 RPC 调用；
  * remoting 包实现 Dubbo 协议，Transport 层和 Exchange 层都放在 remoting 包中，如果不使用自定义 Dubbo 协议，则该层不需要使用；
  * Serialize 层放在 common 包中以便更大程度复用；
  * Registry、Monitor、Config 分别对应 registry、monitor 和 config 这三个包结构。

### 面试题分析

#### Dubbo 框架中最核心的几个概念及其对应的接口定义是什么样的？

  * 考点分析

这个问题考察大家对 Dubbo
中所必须要掌握的几个核心概念的理解，这些核心概念包括远程调用、代理机制、负载均衡、集群容错、网络协议、注册中心等。这些概念本身都比较好理解，表述起来也不大难，但如何结合
Dubbo 这个框架把这几个概念串联起来具有一定的挑战。

  * 解题思路：

从面试的角度讲，面试官问这个问题的目的也并不仅仅只是让我们说明这几个名词的含义而已，而是需要给出一些细节。Dubbo
而言，这些概念背后的抽象，即本文中描述的几个核心接口定义是需要掌握的，如何能用自己的语言进行组织并表述，相信在这个问题上面试官应该没有太多可以指摘的地方。

  * 本文内容与建议回答

结合上一篇和本文中的内容，我们梳理了远程调用、代理机制、负载均衡、集群容错、网络协议、注册中心等 Dubbo
中最核心的几个概念并以接口定义的方式给出了展示。从内容本身而言，这些接口定义也简洁明了，并没有太多复杂的内容。我们回答这个问题时，可以基于概念本身对接口的的基本操作进行展开，这些操作课程中都进行了梳理。

### 日常开发技巧

通过对“基于架构演进过程剖析代码结构”两篇文章的讨论，我们的开发技巧来自于对 UML
中组件图的合理利用，这点在日常开发过程中能够有效帮助我们梳理代码结构中组件之间的关系。当你对代码的结构产生模糊感时，画几张组件图是一个不错的思路。

### 小结与预告

至此，我们通过两篇文章从“基于架构演进过程剖析代码结构”这一主题出发对 Dubbo 框架的架构演进过程做了分析。现在，我们已经能够从宏观角度把握 Dubbo
框架中的各大核心组件，并理解 Dubob
中如何实现简单到复杂、从核心功能到辅助机制逐步完善的设计过程。这里面提到的很多核心组件我们都将在课程的后续内容中做全面展开。而在下一个主题中，我们将通过“基于核心执行流程剖析代码结构”这一主题继续讨论如何更好的把握开源框架的设计思想和实现方法。

