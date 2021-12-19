顾名思义，RPC 是一种远程过程调用的架构，肯定涉及到跨 JVM 的远程调用。今天我们将讨论与如何实现远程调用这一话题的基本方式以及 Dubbo
中的基本处理机制。

### 服务调用的基本方式

服务调用存在两种基本方式，即单向（One Way）模式和请求应答（Request-Response）模式，前者体现为异步操作，而后者一般执行同步操作。

同步调用会造成业务线程阻塞，但开发和管理相对简单。同步调用时序图参考下图，我们可以看到服务线程发送请求到 IO 线程之后就一直处于等待阶段，直到 IO
线程完成与网络的读写操作之后被主动唤醒。

![](https://images.gitbook.cn/2020-05-25-052819.png)

使用异步调用的目的在于获取高性能，队列思想和事件驱动架构都是实现异步调用的常见策略，但都需要依赖于基础中间件平台。这里，我们先将围绕 JDK 中的
Future 模式讨论如何实现异步调用。

Future
模式有点类似于商品订单，在网上购物提交订单后，在收货的这段时间里无需一直在家里等候，可以先干别的事情。类推到程序设计中，提交请求的目的是期望得到响应，但这个响应可能很慢。传统做法是一直等待到这个响应收到后再去做别的事情，但如果利用
Future 模式就无需等待响应的到来，在等待响应的过程中可以执行其他程序。传统调用和 Future 模式调用对比可以参考下图，我们可以看到在 Future
模式调用过程中，服务调用者得到服务消费者的请求时马上返回，可以继续执行其他任务直到服务消费者通知 Future 调用的结果，体现了 Future
调用异步化特点。

![](https://images.gitbook.cn/2020-05-25-052821.png)

作为 Future 模式的实现，Java 中的 Future 接口只包含如下 5 个方法。

    
    
    public interface Future<V> {
    
        boolean cancel(boolean mayInterruptIfRunning);
    
        boolean isCancelled();
    
        boolean isDone();
    
        V get() throws InterruptedException, ExecutionException;
    
        V get(long timeout, TimeUnit unit)?
            throws InterruptedException, ExecutionException, TimeoutException;
    }
    

Future 接口中的 cancel()方法用于取消任务的执行；isCancelled()方法用于判断任务是否已经取消；两个
get()方法会等待任务执行结束并获取结果，区别在于是否可以设置超时时间；最后 isDone()方法判断任务是否已经完成。

Dubbo 中大量使用了基于 Future 机制的异步调用过程，同时也提供了异步转同步的实现机制，让我们来看一下。

### Dubbo 中的远程调用

在 Dubbo 中，远程调用存在三种的调用方式，即异步有返回、异步无返回以及异步变同步，其中异步变同步是默认的实现方式。

让我们抛开方法调用链路的中间环节直接切入到 com.alibaba.dubbo.rpc.protocol.dubbo.DubboInvoker
类中，该类中的核心方法是 doInvoke()，该方法中定义了 Dubbo 中几种具体的远程调用实现方式，代码如下所示。

    
    
    @Override
        protected Result doInvoke(final Invocation invocation) throws Throwable {
            RpcInvocation inv = (RpcInvocation) invocation;
            final String methodName = RpcUtils.getMethodName(invocation);
            inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
            inv.setAttachment(Constants.VERSION_KEY, version);
    
            ExchangeClient currentClient;
            if (clients.length == 1) {
                currentClient = clients[0];
            } else {
                currentClient = clients[index.getAndIncrement() % clients.length];
            }
            try {
                boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
                boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
                int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
    
                if (isOneway) {//单向
                    boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                    currentClient.send(inv, isSent);
                    RpcContext.getContext().setFuture(null);
                    return new RpcResult();
                } else if (isAsync) {//异步
                    ResponseFuture future = currentClient.request(inv, timeout);
                    RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                    return new RpcResult();
                } else {//同步
                    RpcContext.getContext().setFuture(null);
                    return (Result) currentClient.request(inv, timeout).get();
                }
            } catch (TimeoutException e) {
                throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
            } catch (RemotingException e) {
                throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
            }
    }
    

在上述方法中，我们可以清晰看到代码执行的三条路径：

  * 单向

如果是 isOneway（不需要返回值），不管同步还是异步，请求直接发出，不会创建 Future，直接返回 RpcResult 空对象。

  * 异步

如果是 isAsync（异步），则先创建 ResponseFuture 对象，之后使用 FutureAdapter 包装该 ResponseFuture
对象。然后将该 FutureAdapter 对象设入当前线程的上下文中 RpcContext.getContext()，最后返回空的
RpcResult。这样，后续用户能够在合适的时候自己从 RpcContext 获取 Future 对象，并通过 Future 对象的
get()方法获取结果。

  * 同步

如果是同步，则先创建 ResponseFuture 对象，之后直接调用其 Future 的 get()方法进行阻塞调用。

这里比较复杂的是异步场景，Dubbo 基于 Netty
的非阻塞特性发送异步请求，但是这种异步请求最终会转化为同步处理过程并获取响应结果。这个过程就称为“异步转同步”，我们将在介绍《分布式服务架构》时基于
Dubbo 框架再对这一过程做进一步讨论。

### 面试题分析

#### 如何理解 JDK 中的 Future 机制？

  * 考点分析

在涉及到远程调用的应用场景，很多开源框架都会基于 Future 或它的一些变种（如 JDK 自身提供的改进版 CompleteFuture，或是
Google 的 guava 框架中提供的 ListenableFuture 等）。类似的问题主要还是关注于 Future
机制本身的一些特性，可以发散出一系列的问题，但基本的考点是一致的。

  * 解题思路

Future 机制本身提供的几个接口也并不复杂，需要理解它们的含义，也要理解它们存在的不足。普通 Future
机制的最大问题在于没有提供通知的机制，也就是说我们不知道 Future 什么时候能够完成。前面提到的 CompleteFuture 和
ListenableFuture 实际上都是为了改进普通 Future 存在的这一问题而诞生的。

  * 本文内容与建议回答

本文对 Future 的概念做了类比介绍，同时给出了 JDK 中 Future 接口的各个核心方法。通过掌握这些核心方法，针对这个问题我们就能拿到 60
分。如果我们还能够进一步分析基本 Future 机制的不足，然后引出 CompleteFuture 或 ListenableFuture 等改进版本的
Future，那么拿到 80 分就不成问题。

#### Dubbo 中远程调用的实现方式有哪几种？

  * 考点分析

这道题考查面试者对 Dubbo 中远程调用的几种实现方式的理解，实际上是有点难度的，因为如果我们不关注这块内容，没有从源码上理解 Dubbo
中的基本实现原理，还真的很难说清楚它的远程调用实现过程。

  * 解题思路

有了本文的介绍，这道题就属于送分题。我们知道 Dubbo 中远程调用的基本实现方式有三种，即单向、同步和异步，其中前面两种比较好理解，而针对异步，我们在使用
Dubbo 的过程中实际上最终也是转换为同步操作。

  * 本文内容与建议回答

显然，本文直接给出了这道题的标准答案。如果只是回答这个问题中所提出的实现方式的种类，那么本文内容是足够的，但要说明具体的实现细节，尤其是 Dubbo
中”异步转同步“的实现细节，那么还需要进一步学习后续的课程。

### 小结与预告

前面我们通过五篇文章从“基于基础架构组成剖析代码结构”这一主题出发如何把握 RPC
架构这种基础架构做了展开，并抽象了网络通信、序列化、传输协议和远程调用这四个核心组件。然后结合 Dubbo
框架详细分析了每一个组件的设计思路和实现方式。从下一篇开始，我们讨论剖析代码结构的最后一个主题，即基于可扩展性设计剖析代码结构。

