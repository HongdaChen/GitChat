在这一节我们将 CoAP 协议接入 IotHub，IotHub 的 CoAP 包含以下功能：

  * 允许设备用 CoAP 协议接入，并上传数据和状态；
  * DeviceSDK 仍然需要向设备应用屏蔽底层的协议细节；
  * CoAP 设备使用和 MQTT 设备同样的认证和权限系统。

> 由于我只建议用 CoAP 来做数据上传功能，所以在这里只实现上行数据的功能。

### EMQ X 的 CoAP 插件

EMQ X 提供一个 emqx_coap 插件来提供 CoAP 协议的接入，这个插件其实是一个 CoAP Gateway，和上一节提到的 CoAP HTTP
Gateway 类似，不过它会把 CoAP 请求按照一定规则转换成 MQTT 的 Publish/Subscribe：

![avatar](https://images.gitbook.cn/FpqRpx9WKC9pzlZPO9LoIJKHxFIU)

**以下的 CoAP 请求会被转换成 MQTT Publish** ：

**方法** ： PUT **URL** ： coap://:/mqtt/?c=&u=&p=

例如 CoAP 请求 PUT coap://127.0.0.1:5683/mqtt/topic/test?c=c1&u=u1&p=p1
会被转换成主题为`topic/test`的 MQTT Publish 消息，使用的 username/password 为 u1/p1，ClientID 为
c1。

**以下的CoAP请求会被转换成Subscribe请求** ：

**方法** ： GET **URL** ：
`coap://<host>:<port>/mqtt/<topic>?c=<client_id>&u=<username>&p=<password>` 例如
CoAP 请求 GET coap://127.0.0.1:5683/mqtt/topic/test?c=c1&u=u1&p=p1
会被转换成主题为`topic/test`的 MQTT Subscribe 消息，使用的 username/password 为 u1/p1，ClientID
为 c1。

> 本课程，我们只关心 Publish 消息的转换。

运行`< EMQ X 安装目录>/bin/emqx_ctl plugins load emqx_coap`加载 emqx_coap 插件，默认配置下使用端口
5683 接收 CoAP 数据。

### CoAP 设备端代码

本课程使用 [node-coap](https://github.com/mcollina/node-coap) 作为 CoAP 库，在 DeviceSDK
中新建一个 IotCoAPDevice 类作为 CoAP 设备接入的入口：

    
    
    //IotHub_DeviceSDK/sdk/iot_coap_device.js
    class IotCoAPDevice {
        constructor({serverAddress = "127.0.0.1", serverPort = 5683, productName, deviceName, secret, clientID} = {}) {
            this.serverAddress = serverAddress
            this.serverPort = serverPort
            this.productName = productName
            this.deviceName = deviceName
            this.secret = secret
            this.username = `${this.productName}/${this.deviceName}`
            if (clientID != null) {
                this.clientIdentifier = `${this.username}/${clientID}`
            } else {
                this.clientIdentifier = this.username
            }
        }
    }
    

这里使用现有设备的 ProductName 和 DeviceName 进行接入认证。

上传数据和状态的代码很简单，按照转换规则去构造 CoAP 请求就可以了：

    
    
    //IotHub_DeviceSDK/sdk/iot_coap_device.js
    class IotCoAPDevice {
        ...
        publish(topic, payload) {
            var req = coap.request({
                hostname: this.serverAddress,
                port: this.serverPort,
                method: "put",
                pathname: `mqtt/${topic}`,
                query: `c=${this.clientIdentifier}&u=${this.username}&p=${this.secret}`
            })
            req.end(Buffer.from(payload))
        }
    
        uploadData(data, type){
            var topic = `upload_data/${this.productName}/${this.deviceName}/${type}/${new ObjectId().toHexString()}`
            this.publish(topic, data)
        }
    
        updateStatus(status){
            var topic = `update_status/${this.productName}/${this.deviceName}/${new ObjectId().toHexString()}`
            this.publish(topic, JSON.stringify(status))
        }
    }
    

### 代码联调

我们可以写一小段代码来测试 CoAP 的功能：

    
    
    //IotHub_DeviceSDK/samples/upload_coap.js
    var IotCoapDevice = require("../sdk/iot_coap_device")
    require('dotenv').config()
    var path = require('path');
    var device = new IotCoapDevice({
        productName: process.env.PRODUCT_NAME,
        deviceName: process.env.DEVICE_NAME,
        secret: process.env.SECRET,
        clientID: path.basename(__filename, ".js"),
    })
    device.updateStatus({coap: true})
    device.uploadData("this is a sample data", "coapSample")
    

运行`node upload_coap.js`，然后检查 MongoDB 里面对应设备的状态数据和消息数据，可以发现 CoAP 功能在正确的工作。

### CoAP 的连接状态

CoAP 是基于 UDP 的，按道理来说是没有连接的，但是如果我们查看对应设备的连接状态：

    
    
    curl http://localhost:3000/devices/IotApp/H9rTa3uSm
    ...
    {"connected":true,"client_id":"IotApp/H9rTa3uSm/upload_coap","ipaddress":"127.0.0.1","connected_at":1560432017}
    ...
    

会发现对应的设备下多了一个已连接的 connection，而 upload_coap.js 早就执行完毕退出了，这是为什么呢？

因为经过 emqx_coap 的转换之后，对于 EMQ X Broker 来说，它认为是一个 MQTT Client 接入并发布数据，所以会保留一个
MQTT connection， 而 upload_coap.js 执行完毕退出的时候，并没有发送 DISCONNECT 数据包，所以这个 MQTT
connection 状态是已连接。

那什么时候这个连接会变成未连接呢，按照 MQTT 协议的规范，当超过 keep_alive
的时间间隔内没有收到来自该连接的消息，就会认为该连接已关闭。在`< EMQ X
安装目录>/etc/plugins/emqx_coap.conf`中可以配置这个 keep_alive 的值，默认为 120 秒：

    
    
    coap.keep_alive = 120s
    

等待超过 120 秒以后，再次查询设备的连接状态：

    
    
    curl http://localhost:3000/devices/IotApp/H9rTa3uSm
    ...
    {"connected":false,"client_id":"IotApp/H9rTa3uSm/upload_coap","ipaddress":"127.0.0.1","connected_at":1560432017,"disconnect_at":1560432197}
    ...
    

这个连接的状态已变为未连接。

* * *

这一节我们配置 EMQ X Broker 支持 CoAP，并用现有的 IotHub 的设备体系来支持 CoAP
设备的接入和数据上传。本课程的内容到此就结束了。

