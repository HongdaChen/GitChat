任何分布式系统都需要解决网络通信问题，Dubbo 作为一款代表性的 RPC
框架，网络通信是其基础组件。而网络通信作为一项可复用性很好但又相对独立的技术体系，业界也存在一批优秀的框架专门对网络底层的通信过程进行了封装，例如
Netty 和 Mina。Dubbo 同样也是充分利用了这些框架并整合到整个远程方法调用的过程中。

在 Dubbo 中，真正进行网络通信的组件同时涉及服务的提供者和消费者，本节也将分别从 Provider 和 Consumer 的角度出发梳理 Dubbo
的通信过程。在此之前，让我们回顾一下《基于架构演进过程剖析代码结构：Dubbo 框架的架构演进过程分析（下）》中介绍的 Remoting 模块。

### Remoting 模块和 Transport 接口

我们在介绍 Dubbo 整体架构的演进过程中提到了 Remoting 模块，该模块包含的组件如下图所示。

![10.01](https://images.gitbook.cn/2020-05-25-052805.png)

在上图中，Exchange 称为信息交换层，用于封装请求和响应；而 Transport
称为网络传输层，负责具体的网络通信过程。从功能职责上讲，Exchange 偏向于 Dubbo
对自身通信过程的抽象和封装，跟具体的网络通信关闭不大，具体网络传输工作都是由 Transport 负责完成。因此，基于 RPC
基础架构，我们在本节中更多应该关注的是 Transport 层，帮忙大家理解 Dubbo 的网络通信实际上底层也是基于 Netty、Mina
等框架进行的封装和扩展。

站在服务提供者的角度看，通信过程的目的就只有一个，即装配服务并启动监听，从而可以接收服务消费者的访问。而对于服务消费者而言，通信过程的目的无非是对远程服务进行连接、发送请求并获取响应。因此，在
Dubbo 的 com.alibaba.dubbo.remoting 包中存在一个 Transporter 接口，该接口提供了 bind()和
connect()方法分别用于封装这两个基本操作，如下所示。

    
    
    public interface Transporter {
    
        Server bind(URL url, ChannelHandler handler) throws RemotingException;
    
        Client connect(URL url, ChannelHandler handler) throws RemotingException;
    }
    

Transporter 接口存在多个实现类，Dubbo 提供了包括 Grizzly、Mina 和 Netty 在内的底层通信方案，如下所示。

![](https://images.gitbook.cn/2020-05-25-052806.png)

以 com.alibaba.dubbo.remoting.transport.netty4.NettyTransporter 为例，我们可以看到
Transporter 为网络传输层的抽象接口，其核心作用就是提供了创建 Server 和 Client 两个核心接口实现类的方法。基于 netty4 的
NettyTransporter 类代码如下所示：

    
    
    public class NettyTransporter implements Transporter {
    
        public static final String NAME = "netty4";
    
        public Server bind(URL url, ChannelHandler listener) throws RemotingException {
            return new NettyServer(url, listener);
        }
    
        public Client connect(URL url, ChannelHandler listener) throws RemotingException {
            return new NettyClient(url, listener);
        }
    }
    

### NettyServer 和 NettyClient 通信过程

明确了基于 Netty 来实现网络通信过程，就可以分析在 Dubbo 中使用 Netty 的具体方法。

#### NettyServer 类

我们先来看看 NettyServer 类，该类包含了三个核心方法，分别是 doOpen()、doClose()和 getChannels()，其中
doOpen()方法代码如下。

    
    
    @Override
        protected void doOpen() throws Throwable {
            NettyHelper.setNettyLoggerFactory();
    
            bootstrap = new ServerBootstrap();
    
            bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("NettyServerBoss", true));
            workerGroup = new NioEventLoopGroup(getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
                    new DefaultThreadFactory("NettyServerWorker", true));
    
            final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
            channels = nettyServerHandler.getChannels();
    
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
                    .childOption(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
                    .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                    .childHandler(new ChannelInitializer<NioSocketChannel>() {
                        @Override
                        protected void initChannel(NioSocketChannel ch) throws Exception {
                            NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                            ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                                    .addLast("decoder", adapter.getDecoder())
                                    .addLast("encoder", adapter.getEncoder())
                                    .addLast("handler", nettyServerHandler);
                        }
                    });
            // bind
            ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
            channelFuture.syncUninterruptibly();
            channel = channelFuture.channel();
    }
    

熟悉 Netty 的开发人员对上述代码一定不会陌生，这里使用 Netty 的 ServerBootstrap
完成服务启动监听，其中的网络事件处理器使用了自定义的 NettyServerHandler，而 NettyServerHandler 类又继承了 Netty
中的 ChannelDuplexHandler。

同样，NettyServer 提供的 doClose()方法调用 Netty 的 boostrap、channel 完成网络资源释放。

#### NettyClient 类

NettyClient 同样位于 com.alibaba.dubbo.remoting.transport.netty4 包中，分别提供了
doOpen()、doConnect()、doDisconnect()、doClose()和 getChannel()等扩展方法，我们同样来看一下
doOpen()方法，如下所示。

    
    
    @Override
        protected void doOpen() throws Throwable {
            NettyHelper.setNettyLoggerFactory();
            final NettyClientHandler nettyClientHandler = new NettyClientHandler(getUrl(), this);
            bootstrap = new Bootstrap();
            bootstrap.group(nioEventLoopGroup)
                    .option(ChannelOption.SO_KEEPALIVE, true)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                    //.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout())
                    .channel(NioSocketChannel.class);
    
            if (getTimeout() < 3000) {
                bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000);
            } else {
                bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout());
            }
    
            bootstrap.handler(new ChannelInitializer() {
    
                protected void initChannel(Channel ch) throws Exception {
                    NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
                    ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                            .addLast("decoder", adapter.getDecoder())
                            .addLast("encoder", adapter.getEncoder())
                            .addLast("handler", nettyClientHandler);
                }
            });
    }
    

可以看到上述 doOpen()方法的结构也比较固定，创建一个 Netty 的 ClientBootstrap 并初始化参数设置。同样，这里也定义了一个继承自
ChannelDuplexHandler 的 NettyClientHandler。

至此，我们了解了 Dubbo 中进行网络通信的底层技术体系。这些底层技术体系是 Remoting 模块的一部分，整个 Remoting
模块的运行过程比较复杂，我们会在后续介绍《分布式服务架构》模块中进行详细展开。

### 面试题分析

#### Dubbo 如何将网络通信的职责进行了拆分？

  * 考点分析

这道题考查的是对 Dubbo 底层设计的理解，难度比较高。问题本身关注于对网络通信的抽象，Dubbo
框架对这块的抽象比较典型，有一定普适意义，值得进行深入剖析，所以有时候会作为面试题来考查高级别的面试者。

  * 解题思路

从解题思路上，针对网络通信，我们要明确 Dubbo 一方面集成了很多第三方工具，这部分功能只关注与底层的具体网络通信过程，称之为 Transport
层，即网络传输层。而 Dubbo 又提供了一个 Exchange 层，用于封装请求和响应，称为信息交换层。

  * 本文内容与建议回答

在本文的介绍中，Exchange 称为信息交换层，用于封装请求和响应；而 Transport
称为网络传输层，负责具体的网络通信过程。从功能职责上讲，Exchange 偏向于 Dubbo
对自身通信过程的抽象和封装，跟具体的网络通信关闭不大，具体网络传输工作都是由 Transport
负责完成。这两个层级内部也比较复杂，我们没有深入展开，但能够明确以上内容已经足够回答这个问题。

#### Dubbo 内部集成了哪些网络通信框架？默认使用的是哪个？

  * 考点分析

这道题考查的是细节问题，即 Dubbo 中所集成的一些第三方框架。如果对 Dubbo 框架有一定了解，这道属于送分题。

  * 解题思路

我们知道 Dubbo 内部集成了 Netty、Mina 和 Grizzly 这三个网络通信框架，默认使用的是 Netty。而针对
Netty，又分别提供了基于 Netty3 和 Netty4 的两套实现方案，默认的是 Netty4。

  * 本文内容与建议回答

针对这个问题，本文给出了 Dubbo 中集成的各种网络通信框架，并基于 Netty4 分析了基于 Netty 的服务器启动监听过程。

### 日常开发技巧

本文虽然没有全面介绍 Dubbo 中 Remoting
模块的具体实现方式，但当涉及到网络通信相关的场景时，从设计思想上讲，把底层的通信机制和实现过程提取出来放到一个独立的组件中进行管理是一项最佳实践。Dubbo
在此基础上还实现了对通信过程中核心接口的抽象，确保满足《基于组件设计原则剖析代码结构：框架代码结构与组件设计原则》中提到的组件依赖管理原则。这些思想可以扩展到很多应用场景，需要大家平时多观察和总结，提升自己的敏感度。，

### 小结与预告

本文基于 Dubbo 框架介绍了 RPC 架构中的第一个核心组件，即网络通信。我们给出了 Dubbo 框架对这一组件的抽象过程以及内部的核心接口，它们位于
Remoting 模块中。我们注意到在 Remoting 模块中还包含了 Serialize
组件，该组件为一个公共组件，负责数据的序列化和反序列化。这就是下一篇文章要讲的内容。

