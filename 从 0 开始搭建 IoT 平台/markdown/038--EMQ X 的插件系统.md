这一部分的课程，我们将学习如何通过 **编写 EMQ X 插件的方式** 来扩展 EMQ X 的功能。

### Webhook 的局限性

在 IotHub 中我们使用了 EMQ X 的自带的 Webhook 插件，IotHub Server 通过使用 Webhook
插件来获取设备的上下线事件和 Publish 的数据，从开发和演示的功能的角度，这个插件是OK的，但是如果我们在生产环境中使用，你应该要注意到以下问题。

  * **Webhook 缺乏对身份的校验** ，EMQ X 在 Post 到指定的 Webhook URL 的时候，没有带上任何的身份认证信息，所以 IotHub 没有办法知道消息是否真的来自 EMQ X。
  * **性能的损耗** ，在每次设备上下线，和 Publish 数据的时候，EMQ X 都会发起一个 HTTP 请求：建立连接、发送数据、再关闭连接，这部分的开销对 EMQ X 的性能有可见的影响，这还不是最糟的，在那些我们不关注的事件，比如设备订阅、设备取消订阅、送达等发生的时候 EMQ X 依然向 Webhook URL 发起一个请求，这完全是性能的浪费。
  * **健壮性** ，假设 IotHub 的 Web 服务因为某种原因宕机了，在修复好之前，EMQ X 获取的上行数据都会丢失掉。 

这也就意味着， **IotHub 需要一个定制化更强的 Hook 机制** ：

  * 能够对消息和事件的提供者进行验证；
  * 只有在 IotHub 感兴趣的事件发生时，才触发 Hook 机制；
  * 用可持久化的队列来解耦消息和事件的提供者（EMQ X）和消费者（IotHub Server），同时也保证在 IotHub Server 不可用期间，消息和事件不会丢失。

EMQ X 自带了很多插件，不过没有满足上述需求的插件。 但是 EMQ X
的架构提供了很好的可扩展性，我们可以自行编写一个插件来实现上述的功能。在本课程中，我们将编写一个基于 RabbitMQ 的 Hook 插件：

  * 可配置触发 Hook 的事件；
  * 当事件发生时，插件将事件和数据放入 RabbitMQ 的可持久化的 Queue 中，保证事件数据不会丢失。

因为 RabbitMQ 有一套完整的接入认证和 ACL 功能，所以我们可以通过使用 RabbitMQ 的认证体系进行身份验证，保证事件来源的可靠性（只会来自
EMQ X）。

### EMQ X 的扩展插件和 Hook 机制

我们可以在 [EMQ X扩展插件](https://developer.emqx.io/docs/broker/v3/cn/plugins.html)
里看到 EMQ X 关于插件的相关信息，其实 EMQ X 插件机制的逻辑其实很简单，首先 EMQ X 定义了很多 Hook（钩子）：

Hook | 说明  
---|---  
client.authenticate | 连接认证  
client.check_acl | ACL 校验  
client.connected | 客户端上线  
client.disconnected | 客户端连接断开  
client.subscribe | 客户端订阅主题  
client.unsubscribe | 客户端取消订阅主题  
session.created | 会话创建  
session.resumed | 会话恢复  
session.subscribed | 会话订阅主题后  
session.unsubscribed | 会话取消订阅主题后  
session.terminated | 会话终止  
message.publish | MQTT 消息发布  
message.deliver | MQTT 消息进行投递  
message.acked | MQTT 消息回执  
message.dropped | MQTT 消息丢弃  
  
开发者可以通过插件在这些 Hook 上注册处理函数，在插件内部可以调用 EMQ X 内部的数据和方法，执行自定义的业务逻辑，通过这样的方式来对 EMQ X
的功能进行扩展。

我们之前使用 mongoDB 认证插件、jwt 认证插件和 Webhook 插件，都是使用这样的机制。

本课程实现的 RabbitMQ Hook 插件也使用同样的方式。

### Erlang语言

EMQ X 是用 Erlang 编写的，所以我们开发插件也必须使用 Erlang，Erlang
对很多人来说还比较陌生，可能对电信行业的从业者来说要相对好一点。多年前我在诺基亚工作的时候曾经使用过一段时间的Erlang，后来在编写 EMQTT（EMQ
X 3.0 之前被称为 EMQTT）插件的时候又用了一段时间。我使用 Erlang 的经验不是很丰富，也不如其他语言使用得熟练，但是我仍然会说，Erlang
是我使用过的表达力最强的一种语言，强过现在的高级脚本式语言，比如Ruby、Python、Node.js等。但是 Erlang
的学习曲线是比较陡的，在这一部分课程中，我会简单介绍一下 Erlang 语言的一些特性，但是仅局限于在插件编写中使用到的部分。

所以不期望在本课程的短短几节就能学会 Erlang 语言，但是我仍然强烈推荐大家有空去学习一下 Erlang
语言，这是一个伟大的语言，让我第一次有写程序犹如在写诗的感觉。

> 如果要学习 Erlang 的话，我的建议是看由 Erlang 之父编写的《Erlang 程序设计（第2版）》

### 安装 Erlang Runtime

为了开发和编译插件，我们需要按照 Erlang 的 Runtime，EMQ X 3.0 需要 Erlang/OTP-R21+ 来进行编译，这里我们安装最新的
Erlang/OTP-R22.0，你可以在 [Erlang downloads](http://www.erlang.org/downloads/21.3)
找到如何在你的平台上下载和安装 Erlang。

按照完成后，在终端运行"erl"，如果得到以下输出，那么说明安装成功了。

    
    
    Erlang/OTP 22 [erts-10.4.1] [source] [64-bit] [smp:8:8] [ds:8:8:10] [async-threads:1] [hipe] [dtrace]
    
    Eshell V10.4.1  (abort with ^G)
    1>
    

> 本课程建议使用源码编译，或者使用 kerl 安装 Erlang。

### 其他工具

本课程要编写的基于 RabbitMQ 的插件依赖于 Erlang 的 RabbitMQ Client 库，这个库有一部分依赖是用 Elixir
编写的，Elixir 是运行在 Erlang 虚拟机上的一种语言，类似与 Scala 之于 Java。所以还需要安装
[elixir](https://elixir-lang.org/install.html)，同时为了保证能够正确的编译 EMQ X，请保证你系统上的GNU
Make 为 4.0 之后的版本。

然后根据 [rebar3 安装文档](https://www.rebar3.org/docs/getting-started#section-
installing-from-source)安装编译工具 rebar3。

* * *

这一节介绍了 EMQ X 的插件系统，同时介绍了本课程将要编写的插件功能，安装了 Erlang 的 Runtime
和其他编译需要的工具，下一节，我们会简单学习一些 Erlang 语言的特性。

