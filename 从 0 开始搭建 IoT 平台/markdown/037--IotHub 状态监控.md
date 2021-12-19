这一课，让我们来看一个与指令、数据、同步无关的话题： **IotHub 的状态监控** 。

### 状态监控

作为一个服务多个产品的物联网平台，目前我们还缺少一个功能。如何来监控当前物联网平台的运行情况呢？ 这里要讨论的不是监控内存、硬盘I/O、网络等，也不是监控
Web Server、MongoDB 的运行状态，因为这些监控都已经有非常成熟的方案了。这里我们要讨论是， **如何监控 EMQ X Broker
的运行状态** ，比如在线设备的数量有多少，IotHub 一共发送了多少消息、接收了多少消息等。

EMQ X Broker 提供了两种对运行状态进行监控的方式，接下来分别的讲一下这两种方法。

### 监控 REST API

EMQ X 的[监控管理 API](https://developer.emqx.io/docs/broker/v3/cn/rest.html#)
提供了多个可以获取 EMQ X 运行状态的接口，以连接数据统计为例，可以访问：

    
    
    GET api/v3/stats/
    

这个 API 会返回各个节点的连接信息统计数据，格式如下：

    
    
    {
      "code": 0,
      "data": [
        {
          "node": "emqx@127.0.0.1",
          "subscriptions/shared/max": 0,
          "subscriptions/max": 2,
          "subscribers/max": 2,
          "topics/count": 0,
          "subscriptions/count": 0,
          "topics/max": 1,
          "sessions/persistent/max": 2,
          "connections/max": 2,
          "subscriptions/shared/count": 0,
          "sessions/persistent/count": 0,
          "retained/count": 3,
          "routes/count": 0,
          "sessions/count": 0,
          "retained/max": 3,
          "sessions/max": 2,
          "routes/max": 1,
          "subscribers/count": 0,
          "connections/count": 0
        }
      ]
    }
    

> 更多的监控数据接口，可以参考 EMQ X 的文档。

那么我们可以定期（比如 10 分钟）地去调用监控
API，获取相应的数据统计，将结果放入存储，比如之前使用的时序数据库。这样便于查询历史状态数据和状态数据可视化。

### 系统主题

除了使用调用 REST API 这种 "Pull" 的模式，EMQ X 还提供了一种 "Push" 模式来获取运行状态的数据，EMQ X 定义了一系列以
"$SYS" 开头的系统主题，EMQ X 会周期性地向这些主题发布包含运行数据的 MQTT
消息，在[这里](https://developer.emqx.io/docs/broker/v3/cn/guide.html?highlight=%E7%B3%BB%E7%BB%9F%E4%B8%BB%E9%A2%98#sys)可以看到所有系统主题的列表。

那么，我们可以使用一个 MQTT 去订阅相应的主题，收到 MQTT 消息后，将数据存入时序数据库就可以了。

以当前连接数为例，EMQ X会把消息发向主题：`$SYS/brokers/:Node/stats/connections/count`，其中 Node 为
EMQ X 的节点名。如果想获取所有节点的连接数信息，只需要订阅:

`$SYS/brokers/+/stats/connections/count`。

系统主题也是支持共享订阅的，以当前连接数为例，我们可以启动多个 MQTT
Client，订阅`$queue/$SYS/brokers/+/stats/connections/count`，这些 Client
会依次获得当前订阅数的消息，这样就实现了订阅端负载均衡，并避免了单点故障。

默认设置下 EMQ X 会每隔一分钟向这些系统主题发布运行数据，这个时间周期是可以配置的。例如，在开发环境中，可以把这个时间间隔设置得短一点（比如2秒）：

    
    
    ### < EMQ X 安装目录>/conf/emqx.conf
    broker.sys_interval = 2s
    

> 在生产环境中建议设置一个较大的值。

默认情况下，EMQ X 只会向和节点运行在同一服务上的 MQTT Client 订阅这些系统主题，可以通过修改`< EMQ X
安装目录>/conf/acl.conf`来修改这个默认配置：

    
    
    {allow, {ipaddr, "127.0.0.1"}, pubsub, ["$SYS/#", "#"]}
    

本课程中，我们会以这种方案来实现 IotHub 的状态监控。

### EMQ X 的 Listener Zone

现在还剩下最后一个问题，目前 IotHub 的 EMQ X Broker 是做了 Client 认证的，Client
需要提供用户名和密码才能接入，那么对于用于接接收系统主题消息的 Client 也是这样。在现有的架构里面，可以用 jwt 给这些 Client
生成一个临时的 token 作为接入的用户名和密码，但是有没有办法在保持现有认证体系的前提下，让内部使用的 Client 安全地绕过认证呢？

EMQ X 提供了监听器（Listener）Zone 的功能，监听器是指用于接收 MQTT 连接的 server socket，比如我可以配置 EMQ X
分别在端口 1000 和 2000 接受 MQTT 的连接，那么端口 1000 和 2000 分别是两个监听器。Zone
的意思是可以将监听器进行分组，不同的组应用不同策略，比如通过 1000 接入的 Client 需要认证，而通过 2000 接入的 Client 不需要认证。

EMQ X 默认情况下提供了两个 Zone：External 和 Internal，分别对应于接受外部 MQTT 连接和内部 MQTT
连接。External Zone 的 MQTTS (MQTT over SSL) 监听器监听的是 8883
端口，设备连接的就是这个监听器，可以在配置文件找到这个监听器的配置：

    
    
    ### < EMQ X 安装目录>/conf/emqx.conf
    listener.ssl.external = 8883
    

Internal Zone 的 MQTT（非 SSL）监控器监听在 127.0.0.1 的 11883 端口：

    
    
    ### < EMQ X 安装目录>/conf/emqx.conf
    listener.tcp.internal = 127.0.0.1:11883
    

就向上面说的一样，每个 Zone 可以设置自己的一些策略，比如 Internal Zone 默认情况下是允许匿名接入的：

    
    
    ### < EMQ X 安装目录>/conf/emqx.conf
    zone.internal.allow_anonymous = true
    

所以用于接收系统主题的 MQTT Client 连接到
127.0.0.1:11883，这样就不需要用户名和密码了。当然在实际生产中，你应该用例如防火墙的规则来对外屏蔽掉 Internal
监听器的端口，只向外暴露 External 监听器的端口。

> 除了匿名登录以外，Zone 还可以配置很多参数，比如 publish rate、max connection 等，同时还可以定义更多的
> Zone，详情可以查看[ EMQX
> 配置说明](https://developer.emqx.io/docs/broker/v3/cn/config.html)。

### 代码演示

这里我们用一段简单代码来演示如何通过接受系统主题消息的方式，获取 EMQ X 当前的连接数，并存入
InfluxDB。首先通过共享订阅的方式订阅相应的系统主题：

    
    
    //IotHub_Server/monitor.js
    var mqtt = require('mqtt')
    var client = mqtt.connect('mqtt://127.0.0.1:11883')
    client.on('connect', function () {
        client.subscribe("$queue/$SYS/brokers/+/stats/connections/count")
    })
    
    client.on('message', function (topic, message) {
        console.log(`${topic}: ${message.toString()}`)
    })
    

运行这段代码，会得到以下的输出：

    
    
    $SYS/brokers/emqx@127.0.0.1/stats/connections/count: 1
    $SYS/brokers/emqx@127.0.0.1/stats/connections/count: 1
    ...
    

那么这里就获得了当前的 EMQ X 的连接数了，接下来定义连接数在 InfluxDB 里面的存储：

    
    
    //IotHub_Service/services/influxdb_service.js
    const Influx = require('influx')
    const influx = new Influx.InfluxDB({
        host: process.env.INFLUXDB,
        database: 'iothub',
        schema: [
            ...
            {
                measurement: 'connection_count',
                fields:{
                    count: Influx.FieldType.INTEGER
                },
                tags:["node_name"]
            }
        ]
    })
    class InfluxDBService {
        ...
        static writeConnectionCount(nodeName, count) {
            influx.writePoints([
                {
                    measurement: 'connection_count',
                    tags: {node_name: nodeName},
                    fields: {count: count},
                    timestamp: Math.floor(Date.now() / 1000)
                }
            ], {
                precision: 's',
            }).catch(err => {
                console.error(`Error saving data to InfluxDB! ${err.stack}`)
            })
        }
    }
    

这里使用 NodeName 作为 tag，所以需要从主题名里提取出 NodeName：

    
    
    //IotHub_Server/monitor.js
    client.on('message', function (topic, message) {
        var result
        var countRule = pathToRegexp("$SYS/brokers/:nodeName/stats/connections/count")
        if((result = countRule.exec(topic)) != null){
            InfluxDbService.writeConnectionCount(result[1], parseInt(message.toString()))
        }
    })
    

运行`monitor.js`，然后查询 InfluxDB，会得到以下输出：

    
    
    influx
    > use iothub
    Using database iothub
    > select * from connection_count
    name: connection_count
    time                count node_name
    ----                ----- ---------
    1559970858000000000 1     emqx@127.0.0.1
    1559970860000000000 1     emqx@127.0.0.1
    ...
    

说明当前连接数的信息已经被成功记录到 InfluxDB 了，如果需要记录更多的运行状态数据，可以按照同样的方法进行拓展。

* * *

这一节我们讨论了 EMQ X 状态监控的解决方案，并用代码做了演示，到此本课程的第四部分就完成了，我们基本上已经完成 Maque IotHub MQTT
相关的全部功能。在下一部分的课程里面，我们将学习如何编写插件来扩展 EMQ X 的功能。

