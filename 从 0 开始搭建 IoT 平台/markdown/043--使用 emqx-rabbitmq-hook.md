这一课，我们准备用 emqx-rabbitmq-hook 插件来替换掉 IotHub 之前使用的 Webhook 插件。

### 发布 emqx-rabbitmq-hook 插件

虽然在我的电脑上已经有了可运行的 emqx-rabbitmq-hook 插件，但是为了在别的系统和机器上使用这个插件，还是必须要先发布这个插件。

EMQ X 插件的代码需要用一个 git 仓库来存放，你可以
[点击这里](https://github.com/sufish/emqx_rabbitmq_hook) 找到 emqx-rabbitmq-hook
插件的全部代码。

然后是编译 EMQX 和 emqx-rabbitmq-hook 插件，首先选择 EMQX 3.2.0 的版本：

    
    
    git clone -b v3.2.0 https://github.com/emqx/emqx-rel.git
    

然后根据 [rebar3 安装文档](https://www.rebar3.org/docs/getting-started#section-
installing-from-source)安装编译工具 rebar3。

然后编辑 emqx-rel 项目的 rebar.config 文件，加入 emqx-rabbitmq-hook：

    
    
    {deps,
        [
        ...
        , {emqx_rabbitmq_hook, {git, "https://github.com/sufish/emqx_rabbitmq_hook", {branch, "rebar3"}}}
        ]}.
    
    relx,
        [ 
        , {release, {emqx, git_describe},
            [ ...
            , {emqx_rabbitmq_hook, load}
            ]}    
    

然后运行 make

在上一节中我们讲过，新增的 EMQ X 插件不能单独发布，需要和编译生产的 EMQ X binaries 一起发布。 目录 `emqx-rel/
_build/emqx/rel/emqx/` 包含了完整 EMQ X 的文件和目录结构，以及 emqx-rabbitmq-hook
插件，你可以将这个目录打包，解压缩到任意你想安装 EMQ X 的位置，插件就发布成功了。

那么之前我们安装的 EMQ X 就不能用了，关闭旧的 EMQ X，然后把旧的 EMQ X 的配置文件
`emqx.conf`、`etc/plugins/emqx_auth_mongo.conf`、`etc/plugins/emqx_auth_jwt.conf`、`etc/plugins/emqx_web_hook.conf`复制到新的
EMQ X 安装目录的对应位置。然后在新的 EMQ X 的安装目录中运行`bin/emqx start`,。这样，新的包含 emqx-rabbitmq-
hook 插件 EMQ X broker 就运行起来了。之后本课程提到的 EMQ X Broker 就是指新发布的包含 emqx-rabbitmq-hook
插件的 EMQ X 了。

由于使用了新的 EMQ X，所以需要重新申请 EMQ X App 账号，用于调用 EMQ X 的 Rest API，运行：

    
    
    bin/emqx_ctl mgmt insert iothub MaqueIotHub
    

将得到的 secret 放入 `IotHub_Server/.env` 文件中。

最后把 IotHub 需要使用的插件都加载起来：

    
    
    bin/emqx_ctl plugins load emqx_auth_mongo
    bin/emqx_ctl plugins load emqx_auth_jwt
    bin/emqx_ctl plugins load emqx_rabbitmq_hook
    

### 使用 emqx-rabbitmq-hook

IotHub Server 需要从 RabbitMQ 对应的 exchange 中获取事件并进行处理了:

    
    
    //IotHub_Server/event_handler.js
    ...
    var addHandler = function (channel, queue, event, handlerFunc) {
        var exchange = "mqtt.events"
        channel.assertQueue(queue, {
            durable: true
        })
        channel.bindQueue(queue, exchange, event)
        channel.consume(queue, function (msg) {
            handlerFunc(bson.deserialize(msg.content))
            channel.ack(msg)
        })
    }
    amqp.connect(process.env.RABBITMQ_URL, function (error0, connection) {
        if (error0) {
            console.log(error0);
        } else {
            connection.createChannel(function (error1, channel) {
                if (error1) {
                    console.log(error1)
                } else {
                    addHandler(channel, "iothub_client_connected", "client.connected", function (event) {
                        event.connected_at = Math.floor(event.connected_at / 1000)
                        Device.addConnection(event)
                    })
                    addHandler(channel, "iothub_client_disconnected", "client.disconnected", function (event) {
                        Device.removeConnection(event)
                    })
                    addHandler(channel, "iothub_message_publish", "message.publish", function (event) {
                        messageService.dispatchMessage({
                            topic: event.topic,
                            payload: event.payload.buffer,
                            ts: Math.floor(event.published_at / 1000)
                        })
                    })
                }
            });
        }
    });
    

这里为了兼容 Webhook，将事件中的以微秒为单位的时间转换成了秒，这样是为了方便同样的代码在 Webhook 和 rabbitmq_hook
之间切换。如果需要更高精度的时间，可以自行扩展 IotHub Server 的代码。

运行 `node event_handler.js`，再运行之前章节里面的测试代码，可以发现 IotHub 在更换 Hook 插件后仍然在正常工作。

event_handler 必须保持运行，IotHub 才能正常的工作。

### IotHub 的全新架构

在使用 RabbitMQ Hook 之后，处理 MQTT 事件的功能就从运行 Server API 的 Web 服务中剥离出去了，现在 IotHub
由各个相对独立的模块组成：

![avatar](https://images.gitbook.cn/Fr6BLgqoQ3XheyWUlc0-9OnjZBc1)

  * **Server API** ： **对外提供 IotHub 服务的 Rest API 服务** ，通过运行 `bin/www` 启动。
  * **MQTT Event Handler** ： **IotHub 的核心模块** ，处理上行和下行数据的逻辑，通过运行`node event_handler.js`启动。
  * **Broker Monitor** ： **监控 MQTT Broker 运行状态的模块** ，通过运行`node monitor.js`启动。

为了方便启动这些服务，这里添加了 [foreman](https://www.npmjs.com/package/foreman)
的Procfile，首先安装 foreman：

    
    
    npm install -g foreman
    

然后：

    
    
    cd IotHub_Server
    nf start
    

所有的 IotHub 服务就都启动了。

> 细心的读者可能发现了，在目前的代码实现里，当 IotHub 向设备发布消息时，也会触发 "message.publish" 事件，并被发布到
> RabbmitMQ 里面。如果你不希望这样，可以自行扩展 RabbitMQ Hook
> 插件，设置一些主题规则，可以是主题名、正则表达式或者通配符主题名，在 "message.publish" 事件匹配到设置的主题规则时，跳过后续的处理。
>
> 当然不做这个优化也不会影响现有功能。

* * *

这一节我们将 IotHub 中的 Webhook 替换为了 RabbitMQ Hook。到这里，IotHub
的大部分功能和架构就都完成了，本部分的课程也就告一段落了。

在下一部分的课程中，我们将学习另外一种物联网协议 CoAP，并在 IotHub 中使用 CoAP。

