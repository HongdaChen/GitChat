本文将介绍 RPC 架构中的另一个核心组件，即传输协议。Dubbo 中对传输协议进行了一定程度的定制化。在介绍这个主题之前，我们先来简单回顾 ISO/OSI
网络模型。

### ISO/OSI 网络模型与自定义协议

我们知道 ISO/OSI 网络模型分成 7
个层次，自上而下分别是应用层、表示层、会话层、传输层、网络层、数据链路层和物理层。其中传输层实现端到端连接、会话层实现互连主机通讯、表示层用于数据表示、应用层则直接面向应用程序。

传输协议的消息包括消息头和消息体两部分，消息体表示需要传输的业务数据，而消息头用于进行传输控制。下图就是 ISO/OSI 网络 7
层模型中一个协议消息的传输示意图，我们可以看到每个层次都从上层取得数据，加上消息头信息形成新的数据单元，并将新的数据单元传递给下一层次。作为协议的元数据，我们可以在消息头中添加各种定制化信息形成自定义协议。

![](https://images.gitbook.cn/2020-05-25-052737.png)

### RPC 架构与网络传输协议

RPC 架构的设计和实现通常会涉及 ISO/OSI 网络模型中传输层及以上各个层次的相关协议，通常所说的 TCP 协议就属于传输层，而 HTTP
协议则位于应用层。TCP 协议面向连接、可靠的字节流服务，可以支持长连接和短连接。HTTP 是一个无状态的面向连接的协议，基于 TCP
的客户端/服务器端请求和应答标准，同样支持长连接和短连接，但 HTTP 协议的长连接和短连接本质上还是 TCP 的连接模式。

我们可以使用 TCP 协议和 HTTP 协议等公共协议作为基本的传输协议构建 RPC 架构，也可以使用基于 HTTP 协议的 Web Service 和
RESTful 风格设计更加强大和友好的数据传输方式。但大部分 RPC
框架内部往往使用私有协议进行通信，这样做的主要目的在于提升性能，因为公共协议出于通用性考虑添加了很多辅助性功能，这些辅助性功能会消耗通信资源从而降低性能，设计私有协议可以确保协议尽量精简。另一方面，出于扩展性的考虑，具备高度定制化的私有协议也比公共协议更加容易实现扩展。当然，私有协议一般都会实现对公共协议的外部对接。实现自定义私有协议的过程可以抽象成一个模型，自定义协议的通信模型和消息定义、支持点对点长连接通信、使用
NIO 模型进行异步通信、提供可扩展的编解码框架以及支持多种序列化方式是该模型的基本需求。

### Dubbo 自定义协议

下图（来自 Dubbo 官网）是 Dubbo 中采用的私有 Dubbo 协议的定义方式，我们可以看到 Dubbo
协议在会话层中添加了自定义消息头，该消息头包括多协议支持和兼容的 Magic Code 属性、支持同步转异步并扩展消息头的 Id 属性等。Dubbo
协议的设计者认为远程服务调用时间主要消耗在于传输数据包大小，所以 Dubbo 序列化主要优化目标在于减少数据包大小，提高序列化反序列化性能。

![](https://images.gitbook.cn/2020-05-25-052740.png)

#### Dubbo 消息头编码和解码

Dubbo 协议采用固定长度的消息头（16 字节）和不定长度的消息体来进行数据传输，其中消息头报文格式（见上图中的底部“Dubbo”段）中包含以下内容：

  * 0-1byte：一个魔数数字 MAGIC，就是一个固定的数字 0xdabb，用来判断是不是 dubbo 协议的数据包。

  * 2byte：请求的双向或单向标记，一共 8 个地址位。低四位用来表示消息体数据用的序列化工具的类型（默认 hessian），高四位中，第一位为 1 表示是 request 请求，第二位为 1 表示双向传输（即有返回 response），第三位为 1 表示是心跳 ping 事件。

  * 3byte：状态位, 设置请求响应状态，dubbo 定义了一些响应的类型，例如 CLIENT _TIMEOUT = 30 代表客户端超时，SERVER_ TIMEOUT = 31 代表服务端超时。在 com.alibaba.dubbo.remoting.exchange.Response 类中定义了所有响应类型。

  * 4-11byte：请求 ID，long 类型，每一个请求的唯一 ID，主要应用于异步通信场景。由于采用异步通讯的方式，需要把请求 request 和返回的 response 对应上，该请求 ID 用来做 consumer 和 provider 的来回通信标记。。

  * 12-15byte：消息体长度，int 类型。用来记录消息头+请求数据的长度。

Dubbo 协议中对字节流的处理在 com.alibaba.dubbo.remoting.exchange.codec.ExchangeCodec
类中，包含了对请求的编码和解码，也包括了对响应的编码和解码。这里以对请求编码为例分析 Dubbo 的实现过程，体现在
encodeRequest()方法中，如下所示。

    
    
    protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {
            Serialization serialization = getSerialization(channel);
            // header.
            byte[] header = new byte[HEADER_LENGTH];
            // set magic number.
            Bytes.short2bytes(MAGIC, header);
    
            // set request and serialization flag.
            header[2] = (byte) (FLAG_REQUEST | serialization.getContentTypeId());
    
            if (req.isTwoWay()) header[2] |= FLAG_TWOWAY;
            if (req.isEvent()) header[2] |= FLAG_EVENT;
    
            // set request id.
            Bytes.long2bytes(req.getId(), header, 4);
    
            // encode request data.
            int savedWriteIndex = buffer.writerIndex();
            buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
            ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
            ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
            if (req.isEvent()) {
                encodeEventData(channel, out, req.getData());
            } else {
                encodeRequestData(channel, out, req.getData());
            }
            out.flushBuffer();
            bos.flush();
            bos.close();
            int len = bos.writtenBytes();
            checkPayload(channel, len);
            Bytes.int2bytes(len, header, 12);
    
            // write
            buffer.writerIndex(savedWriteIndex);
            buffer.writeBytes(header); // write header.
            buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
    }
    

可以看到 encodeRequest()方法的主要流程就是根据所定义的消息头信息依次填充 header 数值中的各个字段。

#### Dubbo 消息体编码和解码

讨论完消息头，我们来看消息体。消息体的处理同样包括编码和解码过程，这些过程都包含在
com.alibaba.dubbo.rpc.protocol.dubbo.DubboCodec（继承自 ExchangeCodec
类）类中。这里同样以编码为例来进行说明，编码过程位于 encodeRequestData()方法，如下所示。

    
    
    @Override
    protected void encodeRequestData(Channel channel, ObjectOutput out, Object data) throws IOException {
            RpcInvocation inv = (RpcInvocation) data;
    
            out.writeUTF(inv.getAttachment(Constants.DUBBO_VERSION_KEY, DUBBO_VERSION));
            out.writeUTF(inv.getAttachment(Constants.PATH_KEY));
            out.writeUTF(inv.getAttachment(Constants.VERSION_KEY));
    
            out.writeUTF(inv.getMethodName());
            out.writeUTF(ReflectUtils.getDesc(inv.getParameterTypes()));
            Object[] args = inv.getArguments();
            if (args != null)
                for (int i = 0; i < args.length; i++) {
                    out.writeObject(encodeInvocationArgument(channel, inv, i));
                }
            out.writeObject(inv.getAttachments());
    }
    

Dubbo 中请求消息体封装在 com.alibaba.dubbo.rpc.RpcInvocation 类中，而 RpcInvocation 又继承了
Invocation 类。RpcInvocation 中包含了一次请求所需要的各种元数据，包括接口路径、接口版本、请求方法名称、参数类型和参数值等。上述
encodeRequestData()方法依次对这些元数据进行了填充。

### Dubbo 协议粘包拆包的解决

讲到这里，可能很多人要问，Dubbo 为什么要自己开发一套复杂的传输协议？除了为框架本身提供更多的扩展性和灵活性之外，Dubbo
协议的目的是为了解决网络传输过程中的粘包拆包问题。

我们知道 TCP 是个“流”协议，所谓流，就是没有界限的一串数据。TCP 底层并不了解上层业务数据的具体含义，它会根据 TCP
缓冲区的实际情况进行包的划分，所以在业务上认为，一个完整的包可能会被 TCP
拆分成多个包进行发送，也有可能把多个小的包封装成一个大的数据包发送，这就是所谓的 TCP 粘包和拆包问题。

由于底层的 TCP
无法理解上层的业务数据，所以在底层是无法保证数据包不被拆分和重组的，这个问题只能通过上层的应用协议栈设计来解决，根据业界的主流协议的解决方案，可以归纳如下几种：

  * 采用固定长度的消息，例如每个报文的长度固定为 1000 字节，长度不够的情况下则可以填充空格
  * 在消息尾部增加特定的符号进行分割，常见的分隔符包括回车符、换行符等
  * 将消息分为消息头和消息体，其中消息头中包含表示消息长度的特定字段
  * 采用更复杂的应用层协议自定义消息分包实现策略

前面三种都相对容易理解，实现过程也比较简单。但 Dubbo 中采用的则是以上策略中的最后一种，通过规定应用层的 Dubbo
协议来解决这个问题。通过上一小结的分析，我们可以知道 Dubbo
协议分为消息头和消息体，其中消息头中包含了整个消息体的大小。由于粘包拆包问题只涉及到解码部分，所以本小节只对 Dubbo 协议的解码过程进行解析。

我们再回到 com.alibaba.dubbo.remoting.exchange.codec.ExchangeCodec 类，入口
decode()方法如下所示。

    
    
    public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
            int readable = buffer.readableBytes();
            byte[] header = new byte[Math.min(readable, HEADER_LENGTH)];
            buffer.readBytes(header);
            return decode(channel, buffer, readable, header);
    }
    

该
decode()方法首先先判断此次传输的信息包的大小，然后根据传输包的大小，确定本次传输的信息是否包含整个请求头，取与请求头固定长度比较最小值，然后读取相关信息到
header 中。如果此次信息包大于等于 16 字节，说明请求头是完整的。

我们再来看上述代码中嵌套的 decode()方法，代码如下所示。

    
    
    protected Object decode(Channel channel, ChannelBuffer buffer, int readable, byte[] header) throws IOException {
            // check magic number.
            if (readable > 0 && header[0] != MAGIC_HIGH
                    || readable > 1 && header[1] != MAGIC_LOW) {
                int length = header.length;
                if (header.length < readable) {
                    header = Bytes.copyOf(header, readable);
                    buffer.readBytes(header, length, readable - length);
                }
                for (int i = 1; i < header.length - 1; i++) {
                    if (header[i] == MAGIC_HIGH && header[i + 1] == MAGIC_LOW) {
                        buffer.readerIndex(buffer.readerIndex() - header.length + i);
                        header = Bytes.copyOf(header, i);
                        break;
                    }
                }
                return super.decode(channel, buffer, readable, header);
            }
            // check length.
            if (readable < HEADER_LENGTH) {
                return DecodeResult.NEED_MORE_INPUT;
            }
    
            // get data length.
            int len = Bytes.bytes2int(header, 12);
            checkPayload(channel, len);
    
            int tt = len + HEADER_LENGTH;
            if (readable < tt) {
                return DecodeResult.NEED_MORE_INPUT;
            }
    
            // limit input stream.
            ChannelBufferInputStream is = new ChannelBufferInputStream(buffer, len);
    
            try {
                return decodeBody(channel, is, header);
            } finally {
                if (is.available() > 0) {
                    try {
                        if (logger.isWarnEnabled()) {
                            logger.warn("Skip input stream " + is.available());
                        }
                        StreamUtils.skipUnusedStream(is);
                    } catch (IOException e) {
                        logger.warn(e.getMessage(), e);
                    }
                }
            }
    }
    

这个 decode()方法主要是检查请求头的相关信息，首先检查魔法数，魔法数高位和低位各占 1
字节。然后检查当前请求头是否完整，如果不完整直接返回，如果完整则获取此次请求体的长度。接着判断请求头+消息体长度是否大于此次消息包的长度，如果大于的话，说明此次的消息不是完整的一个消息，也意味着进行拆包了，直接返回，等待其它信息；反之就直接解析消息体。

从整个解码过程中，我们不难得出 Dubbo
实现粘包拆包的基本原理就是不断从输入缓冲区中读取数据，并将新读取到的数据向后追加到本地消息缓存中，然后进行解码处理。如果当前本地消息缓存中不足以拼接成一个业务数据包，那就保留数据，继续从输入缓冲区中读取数据；如果当前本地消息缓存中能够拼接成一个业务数据包，那就将对应数据解码成一个完整的业务数据包并传递给业务逻辑处理，本地消息缓存中剩余的多余数据仍然保留，以便和下次读到的数据进行拼接。

### 面试题分析

#### 如果让你来设计 RPC 架构中采用的网络传输协议，你有什么样的一些思考？

  * 考点分析

这是一道开放式问题，有一定难度。考查面试者对网络传输协议本身的理解，同时又关注于如何利用现有的网络传输协议进行一定的扩展和应用。

  * 解题思路

很多同学在面对这种问题时会显得没有什么思路，实际上这种问题的应对方法不外乎两个步骤。第一步需要针对网络传输协议这个话题本身给出一些说明和描述，这属于知识体系部分内容，是需要死记硬背的基础。有了对一个话题的基本了解之后，通常我们对这个话题肯定有一些比较熟悉的点，围绕这些点来进行展开和引导即可。例如，对于网络传输协议，如果你了解
ISO/OSI 模型，那就可以在这个模型上基于类似消息头这种自己比较熟悉的点进行发散。

  * 本文内容与建议回答

本文基于 RPC 架构与网络传输协议，讨论了 TCP、HTTP 协议以及长连接和短连接这些基本概念。同时又引入了私有协议的设计理念，并基于 Dubbo
协议给出了自定义私有协议的一种设计和实现过程。整个分析过程可以说理论结合实践，面试过程中能够给把本文中的内容结合起来进行讲解，相信这道面试题会成为你相比其他面试者的亮点。

#### Dubbo 协议如何解决粘包拆包问题？

  * 考点分析

这也是一个有点难度的问题，一方面考查了粘包拆包问题及其相关的解决方案，另一方面也需要结合 Dubbo 框架给出具体的实现机制。

  * 解题思路

粘包拆包是网络传输过程中一个常见问题，首先需要明确该问题产生的原因以及常见的解决方案。然后针对
Dubbo，我们也可以在理解源代码的基础上，尝试用自己的语言组织该问题的解答思路和过程。

  * 本文内容与建议回答

本文中给出了几种常见的解决粘包拆包问题的设计思路，然后基于 Dubbo
的自定义协议完成了这个设计思路的最终实现。整个过程比较复杂，所以我们在最后也给出了相应的总结。在面试时也建议采用两阶段回答，即先给出解决思路，然后能够把总结部分的内容进行展开就可以了。

### 小结和预告

本文基于 Dubbo 框架介绍了 RPC 架构中的第三个核心组件，即网络传输协议。针对这个组件，Dubbo 框架实现了一套自定的 Dubbo
协议，用于完成针对消息头和消息体的编码和解码工作。下一篇，我们将讨论 Dubbo 中的远程调用过程，这是 Dubbo 作为一个 RPC
框架而言的最后一个核心组件。

