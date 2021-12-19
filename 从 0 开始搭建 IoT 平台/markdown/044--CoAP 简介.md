这一部分课程，我们会介绍另外一种物联网协议：CoAP ，并将这这种协议的接入到IotHub中。

### CoAP协议简介

在《MQTT 协议快速入门》的讨论群里面，我看到过这样的问题，「我的设备运行 MQTT 协议资源很紧张，发布消息很慢」「我的传感器只是发布数据，也要跑
MQTT 吗？」。 MQTT 协议最初是被设计在计算资源和网络资源有限的环境中运行的物联网协议，它在大多数的环境下运行都是很好的，不过 MQTT
需要建立一个 TCP 长连接，并需要在固定的时间间隔发送心跳包，这点 overhead
在一些使用电池供电的小型设备上还是明显的，对于一些采集设备的终端，大部分时间是在往上发布数据，建立一个可以用于反控的 TCP 连接，可能也有些多余。

如果你的应用场景和上面描述的类似，还有另外的一个协议可以选择：CoAP（Constrained Application
Protocol）。协议的详细内容可以查看
[CoAP协议RFC文档](https://tools.ietf.org/html/rfc7252)，当然本课程不会逐条读规范，这里可以用一句话来概括
CoAP：CoAP 是一个建立在 UDP 协议之上，弱化版的 HTTP 协议。

这个就很好理解了，假设设备只需要上传数据，那么完全可以调用服务器的 HTTP 接口，如果运行 HTTP 协议对你的设备来说功耗太大，那么用 CoAP
就好了。

### CoAP 实体

和 MQTT 协议的 Client-Broker-Client 的模型不同，CoAP 和 Web 一样是 C/S 架构的，设备是
Client，接收设备发送数据的就是 Server。

> 所以 CoAP 也被成为 "The Web of Things Protocol"。

### CoAP 的消息模型

CoAP 消息由一下三部分组成：

  * 二进制头（header）；
  * 消息选项（options）；
  * 负载（payload）。

CoAP 消息设计得非常紧凑，最小的 CoAP 消息为 4 个字节，每个 CoAP 消息都有一个唯一的 ID，便于消息的追踪和去重。

CoAP 消息分为两种，一种是需要被确认的消息 CON（Confirmable message），另外一种是不需要确认的消息 NON（Non-
confirmable message）。

#### CON

CON 是一种可靠的消息，当接收方收到 CON 消息时，需要对 CON 消息的的发送方进行回复，如果发送方没有收到接收方的回复，将不停地重试发送消息。

当 Server 正常处理完 CON 消息时，应该向 Client 回复 ACK，ACK 中包含了和 CON 消息一样的 ID：

![avatar](https://images.gitbook.cn/Fuyqls6lmCApRgjIbE-ZOaM4CIUx)

如果 Server 无法处理 Client 的 CON 消息，Server 可以向 Client 回复 RST，Client 将不再等待回复：

![avatar](https://images.gitbook.cn/FuWtZ-69xmqxt0NEpCNf8UatIGPb)

#### NON

NON是一种不可靠的消息，这种消息不需要接收方的回复：

![avatar](https://images.gitbook.cn/Fildo5yvTy3f3RVgYcz9W3z7-ZUB)

通常可以用这种消息来传输类似于传感器读数之类的数据。

### CoAP 的请求/应答机制

CoAP 的请求/应答是建立在上面的 CON 和 NON 的基础之上的，CoAP 的请求和 HTTP
的请求非常相似，可以包含四种方法（GET、POST、PUT、DELETE）和请求的 URL。下面这张图展示了一个典型的 CoAP 请求/应答流程：

![avatar](https://images.gitbook.cn/FoO6aTgJzU2r54tYFe0wh8ASEC5U)

如果请求是用 NON 发送的，那么 Server 端也会用 NON 来回复Client：

![avatar](https://images.gitbook.cn/FogSeR989B462S5q2ReQkClFR81-)

大家可能注意到请求和回复中都带了一个 Token 字段，Token 的作用是匹配请求和应答。如果 Server 没有办法在收到 Client 请求时马上给
Client 应答，它可以先回复一个空的 ACK 消息，当 Server 准备好对 Client 应答时，再向 Client 发送一个包含同样 Token
字段的 CON 消息，类似于一种异步的应答机制：

![avatar](https://images.gitbook.cn/Fij84Qxstarpy-rO6AWwSJu-eooM)

### CoAP OBSERVE

OBSERVE 是 CoAP 的一个扩展，Client 可以向 Server 请求"观察"一个资源，当这个资源状态发生变化时，Server 将通知
Client 资源当前的状态。利用这个机制，可以实现类似于 MQTT 的 Subscribe 进制，对设备进行反控。

Client 可以指定"观察"时长，当超过这个时长后，Server 将不再就资源的状态变化向 Client 发送通知。

> 在 NAT 转换后，尤其是在 3G/4G 的网络下，从 Server 到 Client 的 UDP 传输是很不可靠的，所以我不太建议单纯用 CoAP
> 来做除了数据收集之外的功能，比如设备控制等。

### CoAP HTTP Gateway

由于 CoAP 和 HTTP 协议有很大的相似性，所以我们通常可以用一个 Gateway 将 CoAP 转换成 HTTP 协议，以接入已有的 Web 系统：

![avatar](https://images.gitbook.cn/FhgVCoVYIMENHOEj77kQGb7CHWzl)

* * *

这一节我们学习了 CoAP 协议的内容和特性，下一节我们将学习如何将 CoAP 接入 IotHub。

