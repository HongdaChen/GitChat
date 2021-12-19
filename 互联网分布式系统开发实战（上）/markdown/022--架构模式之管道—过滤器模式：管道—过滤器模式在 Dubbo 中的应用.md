Dubbo 中的过滤器概念基本上符合我们对管道-过滤器模式的理解。在本篇中，我们先来看一下 Dubbo 中过滤器链的构建过程，然后介绍 Dubbo
中现有过滤器的实现方法。

### Dubbo 过滤器链的构建过程

Dubbo 中的 Filter 实现入口是在 ProtocolFilterWrapper 类（位于 dubbo-rpc-api 工程的
com.alibaba.dubbo.rpc.protocol 包中），在服务暴露和服务引用时都会使用到过滤器链。所谓的
Wrapper，顾名思义，是对扩展类的一种包装，如下图所示。Dubbo 中的包装类同样实现扩展点接口，具有与扩展点一样的方法。目前，纵观整个 Dubbo
框架，只存在一个 Wrapper，即 ProtocolFilterWrapper。

![22.01](https://images.gitbook.cn/2020-05-25-52701.png)

ProtocolFilterWrapper 类实现了 Protocol 接口，并具有如下所示的 export 和 refer 方法实现：

    
    
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
            if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
                return protocol.export(invoker);
            }
            return protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY, Constants.PROVIDER));
     }
    
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
            if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                return protocol.refer(type, url);
            }
            return buildInvokerChain(protocol.refer(type, url), Constants.REFERENCE_FILTER_KEY, Constants.CONSUMER);
    }
    

可以看到这两个方法都使用了一个名为 buildInvokerChain 的方法，从命名上看，该方法就是用来构建调用链，如下所示：

    
    
    private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
            Invoker<T> last = invoker;
            List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
            if (filters.size() > 0) {
                for (int i = filters.size() - 1; i >= 0; i--) {
                    final Filter filter = filters.get(i);
                    final Invoker<T> next = last;
                    last = new Invoker<T>() {
                        //这里省略了构造一个最简化的 Invoker 作为调用链载体 Invoker 的过程
                        ...
                        public Result invoke(Invocation invocation) throws RpcException {
                            return filter.invoke(next, invocation);
                        }
                        ...
                    };
                }
            }
            return last;
    }
    

我们再一次看到了用于获取扩展点的
ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension()方法。注意到这里对通过扩展点加载的过滤器进行了排序从而确保过滤器链按设想的顺序进行执行。

另一方面，我们需要注意到上述代码中，我们构建了一个最简化的 Invoker 作为调用链载体 Invoker
的过程。由于篇幅有限，我们省略了这个构建过程的大部分代码，但保留了一个核心的 invoke 方法实现。我们可以看到，这个所构建出来的 Invoker 对象的
invoke 方法实际上是调用了 Filter 所提供的 invoke 方法，从而对原本 Invoker 直接执行的 invoke 方法进行了一层
Filter 拦截，接下来就来看一下 Filter 的实现方法。

### Dubbo 中的过滤器

看完过滤器链，我们反过来看一下过滤器。Dubbo 中的 Filter 接口定义如下：

    
    
    @SPI
    public interface Filter {
    
        Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException;
    }
    

可以看到 Filter 接口能够对获取传入的 Invoker 进行处理，从而对其进行拦截和处理。针对 Filter 接口，Dubbo 中一共存在 21
个实现类，类层结构如下图所示：

![22.02](https://images.gitbook.cn/2020-05-25-052701.png)

上图中几乎所有的过滤器（除了 CompatibleFilter）都使用了@Activate
注解，即默认被激活。而且这些过滤器可以大致分成两类，即面向服务提供者的过滤器和面向服务消费者的过滤器。其中面向服务提供者的过滤器只会在服务暴露时对
Invoker 进行过滤，而面向服务消费者的过滤器发生作用的阶段自服务引用时。一般的过滤器只能属于这两种类型中的一种，但是 MonitorFilter
是个例外，它可以同时作用于服务暴露和服务引用阶段，因为它需要对这两个阶段都进行监控。

常见的面向服务提供者的过滤器包括
AccessLogFilter、ExecuteLimitFilter、ExceptionFilter、EchoFilter、TimeoutFilter、TpsLimitFilter、TraceFilter
等，而常见的面向服务消费者的过滤器则包括 ActiveLimitFilter、FutureFilter、ConsumerContextFilter
等。我们无意对所有这些过滤器组件做详细展开，这里挑选两个比较具有代表性的 Filter 进行展开，即 ExecuteLimitFilter 和
TokenFilter。

#### ExecuteLimitFilter

关于 ExecuteLimitFilter，我们前面已经在《架构模式之微内核模式：Dubbo
中的扩展点》一文中看到过它的定义，但是没有具体介绍它的实现方法。ExecuteLimitFilter 的目的是实现对服务端并发数的控制，可以想象该类势必在
Invoker 的 invoke 方法执行之前添加了一层拦截。ExecuteLimitFilter 实现如下：

    
    
    @Activate(group = Constants.PROVIDER, value = Constants.EXECUTES_KEY)
    public class ExecuteLimitFilter implements Filter {
    
        public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
            URL url = invoker.getUrl();
            String methodName = invocation.getMethodName();
            Semaphore executesLimit = null;
            boolean acquireResult = false;
    
            //从 URL 中获取最大并发量参数值
            int max = url.getMethodParameter(methodName, Constants.EXECUTES_KEY, 0);
            if (max > 0) {
                RpcStatus count = RpcStatus.getStatus(url, invocation.getMethodName());
    
                //使用信号量进行并发控制
                executesLimit = count.getSemaphore(max);
                if(executesLimit != null && !(acquireResult = executesLimit.tryAcquire())) {
                    throw new RpcException("Failed to invoke method " + invocation.getMethodName() + " in provider " + url + ", cause: The service using threads greater than <dubbo:service executes=\"" + max + "\" /> limited.");
                }
            }
            long begin = System.currentTimeMillis();
            boolean isSuccess = true;
            RpcStatus.beginCount(url, methodName);
            try {
                Result result = invoker.invoke(invocation);
                return result;
            } catch (Throwable t) {
                isSuccess = false;
                if (t instanceof RuntimeException) {
                    throw (RuntimeException) t;
                } else {
                    throw new RpcException("unexpected exception when ExecuteLimitFilter", t);
                }
            } finally {
                RpcStatus.endCount(url, methodName, System.currentTimeMillis() - begin, isSuccess);
                if(acquireResult) {
                    executesLimit.release();
                }
            }
        }
    }
    

上述方法展示了信号量的一种常见用法。我们基于基于 URL 和所调动方法本身构建一个 RpcStatus 实例，然后通过 RpcStatus
实例获取一个信号量。如果获取的这个信号量调用 tryAcquire 返回
false，则抛出异常，即意味着已经超过了所能请求的并发量。如果没有抛异常，那么就调用 RpcStatusbeginCount
静态方法为这个信号量进行计数。而当调用结束后则调用 RpcStatusendCount 静态方法结束计数并释放信号量。

#### TokenFilter

另一个在实现过程上有参考价值的 Filter 是 TokenFilter。TokenFilter 的作用很明确，即通过 Token 进行访问鉴权，通过对比
Invoker 中的 Token 和传入参数中的 Token 来判断是否是合法的请求，其代码如下所示。

    
    
    @Activate(group = Constants.PROVIDER, value = Constants.TOKEN_KEY)
    public class TokenFilter implements Filter {
    
        public Result invoke(Invoker<?> invoker, Invocation inv)
                throws RpcException {
    
            //获取本地 Token
            String token = invoker.getUrl().getParameter(Constants.TOKEN_KEY);
            if (ConfigUtils.isNotEmpty(token)) {
                Class<?> serviceType = invoker.getInterface();
                Map<String, String> attachments = inv.getAttachments();
    
                //获取远程 Token
                String remoteToken = attachments == null ? null : attachments.get(Constants.TOKEN_KEY);
    
                //比对本地 Token 和远程 Token
                if (!token.equals(remoteToken)) {
                    throw new RpcException("Invalid token! Forbid invoke remote service " + serviceType + " method " + inv.getMethodName() + "() from consumer " + RpcContext.getContext().getRemoteHost() + " to provider " + RpcContext.getContext().getLocalHost());
                }
            }
            //继续执行 Invoker
            return invoker.invoke(inv);
        }
    }
    

上述代码中，我们关注两点。首先我们看到可以通过 invoker.getUrl()方法获取 Invoker 中的 URL 对象，而我们知道 Dubbo 中的
URL 作为统一数据模型 Key-Value 对的形式包含了所有服务调用过程中的参数。同时，我们看到了 Invocation 对象，该对象可以理解为是一种
DTO（Data Transfer Object，数据传输对象），用来封装所需要传递的数据。这样，一方面我们通过 URL 对象获取本地 token
参数；另一方面，我们也通过 Invocation 的 Attachments 获取了 remoteToken，从而可以执行对比和校验操作。这也是 Dubbo
中处理调用信息传递的非常常见的一种做法，我们可以在很多地方看到类似的代码。

### 自定义 Dubbo 过滤器

最后，让我们基于对 Dubbo 中过滤器机制的理解来实现一个自定义的过滤器组件。这个过滤器非常简单，就是记录一下 invoke
调用所使用的时间，代码如下所示。

    
    
    public class TimeLogFilter implements Filter {     
            private static Logger log = LoggerFactory.getLogger(TimeLogFilter.class);
    
            @Override
            public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
                long start = System.currentTimeMillis();
                Result result = invoker.invoke(invocation);
                long elapsed = System.currentTimeMillis() - start;
                if (invoker.getUrl() != null) {  
                    log.info("[{}], [{}], {}, [{}], [{}], [{}]   ", 
                        invoker.getInterface(), 
                        invocation.getMethodName(), 
                        Arrays.toString(invocation.getArguments()), 
                        result.getValue(),
                        result.getException(), elapsed);     
                }
                return result;
            }    
    }
    

可以看到，实现一个自定义过滤器唯一要做的实现就是实现 Filter 接口，并利用传入的 Invoker 和 Invocation
对象进行相关的处理。如果想要记录执行时间，我们只需要在前一个过滤器所执行的 invoker.invoke(invocation)方法前后添加时间记录即可。

### 面试题分析

#### Dubbo 中内置了哪些常见的过滤器？有什么作用？

  * 考点分析

这个问题的考点比较明确，就是考查 Dubbo
中的过滤器类型以及相应的作用。有点属于偏记忆性的话题，但面试官更多关注的还是回答问题的思路而不是具体某一个过滤器的细节。我们在回答的过程中也应该尽量避免局限于细节本身，而是需要站在整个
Dubbo 过滤器体系的角度。

  * 解题思路

Dubbo
中内置了一大批过滤器，按照面向对象而言可以大致分成两类，即面向服务提供者的过滤器和面向服务消费者的过滤器。过滤器的作用都是为了拦截当前的执行流程并嵌入一些非功能性需求，例如本文中介绍的限流和安全认证等。对这个问题，我们需要了解
Dubbo 中 Filter 接口的传入对象，即一个 Invoker 和一个 Invocation，然后结合具体的场景能够用自己的语言组织常见集中
Filter 的实现机制。

  * 本文内容与建议回答

本文中给出了两大类过滤器中的典型代表的简单描述。同时，给出了 ExecuteLimitFilter 和 TokenFilter
这两个典型的过滤器的实现细节。在回答该问题上，建议可以选择自己最熟悉的一到两个过滤器进行展开，分别阐述它的作用以及实现细节。

#### Dubbo 中过滤器链的构建方法？

  * 考点分析

这个问题有一定的难度，属于源码层级的原理性问题，考查面试者对 Dubbo 实现原理的掌握程度。Dubbo
中采用了一种特殊的方式来构建过滤器链，所以比较容易作为面试题，用于筛选高级别的开发人员。

  * 解题思路

Dubbo 中实现管道过滤器架构模式的方式也是采用了过滤器链，并引入了 Wrapper 概念。ProtocolFilterWrapper 类 Dubbo
中唯一的一个 Wrapper 类，所以需要重点进行记忆。ProtocolFilterWrapper
中构建过滤器链的方法是通过扩展点加载的过滤器进行了排序从而确保过滤器链按设想的顺序进行执行。

  * 本文内容与建议回答

对于理解 Dubbo 中的过滤器链，本文中所介绍的两处要点是回答该问题的关键。首先，ProtocolFilterWrapper 类的设计和实现过程引出了
Dubbo 中代表过滤器链的 InvokerChain 概念，而本文中提供的 buildInvokerChain
方法则用于完成过滤器链的构建，如果能够阐述该方法内部使用一个 Invoker
对象来嫁接过滤器功能这一设计技巧，那么对于该问题而言应该应该能够满足面试官的要求。

### 日常开发技巧

作为一种常见的扩展机制，为 Dubbo 中添加自定义 Filter
尽管不是很常见，但在某些特定场景下也可以帮助我们解决一些面向切面的需求，比较典型的就是添加类似本文示例所示的监控或报警方面的增强型功能。

### 小结与预告

本文结合 Dubbo 框架详细剖析了管道过滤器架构模式的实现和应用方式。在下一篇中，我们将采用同样的描述方式来对这一模式在 Mybatis
中的实现机制进行展开讨论。

