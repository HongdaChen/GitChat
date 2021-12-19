上文中我们给出了 Dubbo 中基于 SPI 机制实现微内核模式的具体方式，可以看到 Dubbo 中采用了与 JDK 中 ServiceLoader
完全不同的实现机制。另一方面，应用微内核模式的目的就是为了实现扩展，因此 Dubbo 提供了一系列的扩展点。一方面，我们可以学习 Dubbo
中内置的扩展点实现，另一方面，我们也可以尝试自己提供对这些扩展点的实现。

### Dubbo 扩展点类型和实现机制

要想知道 Dubbo 中到底包含了的所有扩展点，我们可以整合所有工程中 META-
INF/dubbo/internal/目录下的配置文件来进行获取。结合本课程《基于架构演进过程剖析代码结构：Dubbo 框架的架构演进过程分析》篇中介绍的
Dubbo 组件分层图以及所有这些扩展点，我们可以获取如下所示的 Dubbo 中部分扩展点的分层图：

![20.01](https://images.gitbook.cn/2020-05-25-052635.png)

本课程无意是上图中所有的扩展点都做详细展开，获取各个扩展点详细信息的最后版本就是找到代码工程中的 META-INF/dubbo
目录下的配置文件。在这里，我们选择 RegistryFactory 和 LoadBalance 作为我们的扩展点示例，以此介绍 Dubbo
中扩展点的具体实现方法。

#### RegistryFactory 扩展点

我们先来看 RegistryFactory 扩展点。在 Dubbo 中，在表现形式上所有的扩展点都被定义为接口，RegistryFactory
接口定义如下（位于 dubbo-registry-api 工程的 com.alibaba.dubbo.registry 包中）：

    
    
    @SPI("dubbo")
    public interface RegistryFactory {
    
        @Adaptive({"protocol"})
        Registry getRegistry(URL url);
    }
    

@SPI 作用于 RegistryFactory 接口，表示该接口是一个扩展点。@SPI 注解有一个参数，该参数表示该扩展点的默认实现的别名。Dubbo
中针对上述 RegistryFactory 接口提供了一系列的实现类，其类层结构如下所示：

![20.02](https://images.gitbook.cn/2020-05-25-052636.png)

在上图中，我们看到了默认的 DubboRegistryFactory、基于多播模式的 MulticastRegistryFactory 以及分别基于
Redis 和 Zookeeper 的 RedisRegistryFactory 和 ZookeeperRegistryFactory。以
ZookeeperRegistryFactory 为例，它的实现方法如下所示：

    
    
    public class ZookeeperRegistryFactory extends AbstractRegistryFactory {
    
        private ZookeeperTransporter zookeeperTransporter;
    
        public void setZookeeperTransporter(ZookeeperTransporter zookeeperTransporter) {
            this.zookeeperTransporter = zookeeperTransporter;
        }
    
        public Registry createRegistry(URL url) {
            return new ZookeeperRegistry(url, zookeeperTransporter);
        }
    }
    

可以看到这里返回了一个 ZookeeperRegistry，而在 ZookeeperRegistryFactory 的父类
AbstractRegistryFactory 中，我们发现了如下所示的 getRegistry 方法：

    
    
    public Registry getRegistry(URL url) {
            url = url.setPath(RegistryService.class.getName())
                    .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
                    .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
            String key = url.toServiceString();
            // Lock the registry access process to ensure a single instance of the registry
            LOCK.lock();
            try {
                Registry registry = REGISTRIES.get(key);
                if (registry != null) {
                    return registry;
                }
                registry = createRegistry(url);
                if (registry == null) {
                    throw new IllegalStateException("Can not create registry " + url);
                }
                REGISTRIES.put(key, registry);
                return registry;
            } finally {
                // Release the lock
                LOCK.unlock();
            }
        }
    

可以看到，这里定义了一个名为 REGISTRIES 的 MAP 来对创建的 Registry 进行缓存，而具体创建 Registry 的过程则有子类的
createRegistry 来完成，正如我们在 ZookeeperRegistryFactory 中看到的那样。

然后，我们来检查以 dubbo-registry-xxx 方式命名的各个代码工程，都会发现在 META-INF/dubbo/internal 目录下存在一个
com.alibaba.dubbo.registry 文件，分别指向该工程中的 RegistryFactory 实现类。以 dubbo-registry-
zookeeper 工程为例，它的 com.alibaba.dubbo.registry 文件内容如下所示：

    
    
    zookeeper=com.alibaba.dubbo.registry.zookeeper.ZookeeperRegistryFactory
    

注册中心是微服务架构的核心组件，关于 ZookeeperRegistry 的详细讨论我们会放到后续的微服务架构模块中。

#### LoadBalance 扩展点

接下来，让我们来看看 LoadBalance 接口的定义（位于 dubbo-cluster 工程的
com.alibaba.dubbo.rpc.cluster 包中），如下所示：

    
    
    @SPI(RandomLoadBalance.NAME)
    public interface LoadBalance {
    
        @Adaptive("loadbalance")
        <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
    }
    

可以看到，作为负载均衡器，LoadBalance 接口只有一个 select 方法，该方法从多个 invoker
中选择其中一个。显然，该接口定义中也用到了@SPI 和@Adaptive 这两个与扩展点相关的注解。

我们在前面的课程中已经知道 LoadBalance 接口有
RandomLoadBalance、RoundRobinLoadBalance、LeastActiveLoadBalance 和
ConsistentHashLoadBalance 等四种负载均衡实现策略，而这里的@SPI 参数 RandomLoadBalance.NAME
是一个常量，值为"random"，即该注解使用 RandomLoadBalance 作为该扩展点的默认实现。

我们注意到 LoadBalance 接口也有一个对应的抽象实现类 AbstractLoadBalance，其 select 方法如下所示：

    
    
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
            if (invokers == null || invokers.size() == 0)
                return null;
            if (invokers.size() == 1)
                return invokers.get(0);
            return doSelect(invokers, url, invocation);
    }
    

显然，最终的负载均衡的实现过程交给了各个子类的 doSelect 方法。如果我们在 dubbo-cluster 工程中找到 META-
INF/dubbo/internal/com.alibaba.dubbo.rpc.cluster.LoadBalance
配置文件，我们不难想象将看到如下信息，该文件中定义了我们已知的四个 LoadBalance 扩展实现。

    
    
    random=com.alibaba.dubbo.rpc.cluster.loadbalance.RandomLoadBalance
    roundrobin=com.alibaba.dubbo.rpc.cluster.loadbalance.RoundRobinLoadBalance
    leastactive=com.alibaba.dubbo.rpc.cluster.loadbalance.LeastActiveLoadBalance
    consistenthash=com.alibaba.dubbo.rpc.cluster.loadbalance.ConsistentHashLoadBalance
    

如果想要获取一个 LoadBalance 接口的实例，我们就可以使用前面介绍的 ExtensionLoader 类。具体写法可以参考如下：

    
    
    LoadBalance loadBalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(loadbalanceName); 
    

这里使用 ExtensionLoader.getExtensionLoader(LoadBalance.class)方法获取一个
ExtensionLoader 的实例，然后调用 getExtension 方法传入一个扩展的别名来获取对应的扩展实例。

负载均衡同样是分布式服务架构体系中的重要一环，关于 Dubbo 以及 Spring Cloud 中负载均衡策略的实现原理我们会在后续的课程中进行详细展开。

### 基于 Dubbo 实现自定义扩展点

前面我们分析了 Dubbo 中已经存在的两个扩展点。对于 Dubbo 而言，提供这些扩展点的一个目的是希望其他开发人员可以基于 SPI
机制开发新的扩展点实现。在 Dubbo 中要实现一个自定义的扩展点实际上很简单，因为 Dubbo 已经为我们准备好了整个基础设施。而且，Dubbo
从一开始就希望通过扩展点机制让广大开发人员为 Dubbo 框架的发展贡献力量。我们也还是以 LoadBalance 为例来演示如何编写一个具体的扩展点。

首先，让我们新建一个工程，然后对前面介绍的 LoadBalance 接口提供一种实现，如下所示：

    
    
    public class HostFilteringLoadBalance extends AbstractLoadBalance {
    
        @Override
        protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
            for (Invoker<T> invoker : invokers) {
                if (invoker.getUrl().getHost().equals(invocation.getArguments()[2].toString())) {
                    return invoker;
                }
            }
    
            return invokers.get(0);
        }
    }
    

可以看到，这个示例的实现方式非常简单。我们直接从 AbstractLoadBalance 进行扩展，而不是实现 LoadBalance
接口。这里，我们根据传入的远程服务器的 Host 进行过滤，找到匹配的 Invoker 并返回。如果找不到，我们简单返回第一个 Invoker
对象。基于这种思想，我们实际上也可以实现类似 Eureka 中所具备的区域亲和性（Zone Affinity）概念。

然后，我们需要在该工程下的 META-INF/dubbo/目录中添加一个配置文件，该文件的名称应该是 LoadBalance 接口的完成类路径，即
com.alibaba.dubbo.rpc.cluster.LoadBalance。文件内容则指向刚才所创建的
HostFilteringLoadBalance 类，比方说：

    
    
    hostfiltering=com.demo.extensions.HostFilteringLoadBalance
    

自定义一个扩展点要做的事情貌似也就这么多。

### 面试题分析

#### Dubbo 中有哪些常见的扩展点？

  * 考点分析

实际上，这个问题的本意不光是考查 Dubbo
中的所有扩展点，而是考查对扩展点机制的了解。所以扩展点机制本身以及各种常见的扩展点都是这个问题的考点。一般这种问题，面试官都会结合面试者的回答挑选一两个扩展点就其具体实现过程进行展开。

  * 解题思路

Dubbo 中的扩展点非常多，基于文中提供的分层结构对其进行记忆是一个比较好的方法。当然，通过系统的学习，诸如 Protocol、Transport
以及本文中提到的 RegistryFactory 和 LoadBalance
扩展点是不可能绕开的基本概念。在面试过程中，需要引导面试官朝自己擅长的某个或某些扩展点进行引导，从而得到跟多正面的评价。

  * 本文内容与建议回答

本文给出了基于分层结构的扩展点记忆方法，可以结合《基于架构演进过程剖析代码结构：Dubbo
框架的架构演进过程分析》中的内容以及自己的理解对其进行展开。至于具体扩展点的示例，本文也给出了 RegistryFactory 和 LoadBalance
这两个扩展点供大家在面试过程中进行展开。

#### 如何在 Dubbo 中实现一个自定义的扩展点？

  * 考点分析

Dubbo 中实现一个自定义扩展点的过程并不复杂，这也是 Dubbo 采用微内核架构的初衷。

  * 解题思路

这道题基本属于送分题，只要对 Dubbo 中的扩展点机制有一定的了解就可以直接进行回答。

  * 本文内容与建议回答

本文中针对 LoadBalance 扩展点提供了一个实现 Demo，尽管很简单，但也已经包含了所需要实现的所有步骤。大家可以直接参考这个 Demo
来进行展开。

### 日常开发技巧

对于 Dubbo 框架而言，尽管我们很少需要自己实现一个扩展点，但有时候确实有这样的应用场景，例如前面介绍的基于 LoadBalance 扩展点来实现类似
Eureka 中的区域亲和性。如果我们有足够好的扩展点实现方案，甚至可以提交到 Dubbo 社区成为一名 Committer。

### 小结与预告

本文在微内核架构的基础上，讨论了 Dubbo
中的部分扩展点以及内置实现。这样，微内核架构模式方面的内容就介绍完毕，下一篇我们将讨论另一种常见的架构模式，即管道-过滤器模式。

