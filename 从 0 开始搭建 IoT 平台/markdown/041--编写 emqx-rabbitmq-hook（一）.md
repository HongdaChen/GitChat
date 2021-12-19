从这一节我们开始开发EMQ X插件：emqx-rabbitmq-hook。

和前面说的一样， emqx-rabbitmq-hook 插件会在一些事件发生时，比如设备连接、发布消息时，将事件的数据发送到 RabbitMQ 指定的
exchange 中。

在这一节中，我们会搭建 emqx-rabbitmq-hook 插件的代码框架，并实现第一个功能，在设备连接时将连接事件的信息发送到相应的 RabbitMQ
exchange 中去。

### 代码结构

在开发的时候我们可以直接在 `emqx-rel/deps` 创建一个目录 `emqx_rabbitmq_hook` 来存放 emqx-rabbitmq-
hook 插件的代码：

![avatar](https://images.gitbook.cn/Fq5xaegiJ_lXvCbtHCSXwhKG2uSE)

初始代码结构基本和 emqx-plugin-template 一致，然后再在这个基础上叠加代码。

### 建立 RabbitMQ 连接和连接池

我们需要在插件启动的时候建立和 RabbitMQ 的连接，同时我们希望用一个连接池对插件的 RabbitMQ 进行管理，第一步是在插件的
rebar.config 文件中添加相应的依赖：

    
    
    ### emqx_rabbitmq_hook/rebar.config
    
    {deps, [
        {amqp_client, "3.7.15"}, 
        {ecpool, "0.3.0"} ,
       ...
    ]}.
    {erl_opts, [debug_info]}.
    

接着在监控器启动的时候，初始化连接池：

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook_sup.erl
    init([]) ->
      application:set_env(amqp_client, prefer_ipv6, false),
      PoolSpec = ecpool:pool_spec(?APP, ?APP, emqx_rabbitmq_hook_cli, [{pool_size, 10}, {host, "127.0.0.1"}, {port, 5672}]),
      {ok, {{one_for_one, 10, 100}, [PoolSpec]}}.
    

这里暂时把配置都 hardcode 到代码里，下一课我们再学习如何从配置文件中读取配置。连接池需要我们自行实现 RabbitMQ 连接的功能：

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook_cli.erl
    connect(Opts) ->
      ConnOpts = #amqp_params_network{
        host = proplists:get_value(host, Opts),
        port = proplists:get_value(port, Opts)
      },
      {ok, C} = amqp_connection:start(ConnOpts),
      {ok, C}.
    

RabbitMQ 的连接和连接池就建立完成了。

### 处理 client.connected 事件

我们先以处理 "client.connected" 为例来打通整个流程，默认情况下 emqx-rabbitmq-hook 插件会把事件数据发送到名为
"mqtt.events" 的 exchange 中，exchange 的类型为 direct，事件的数据将用 BSON 进行编码。首先引入对 BSON
的依赖：

    
    
    ### emqx_rabbitmq_hook/rebar.config
    
    {deps, [
       ....
       {bson_erlang, "0.3.0"}
    ]}.
    {erl_opts, [debug_info]}.
    

在插件启动时，应该首先声明这个 exchange：

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook.erl
    load(Env) ->
      emqx_rabbitmq_hook_cli:ensure_exchange(),
      emqx:hook('client.connected', fun ?MODULE:on_client_connected/4, [Env]),
      ...
    

插件会从连接池中取一个连接来执行声明 exchange 的操作：

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook_cli.erl
    ensure_exchange() ->
      ensure_exchange(<<"mqtt.events">>).
    
    ensure_exchange(ExchangeName) ->
      ecpool:with_client(?APP, fun(C) -> ensure_exchange(ExchangeName, C) end).
    
    ensure_exchange(ExchangeName, Conn) ->
      {ok, Channel} = amqp_connection:open_channel(Conn),
      Declare = #'exchange.declare'{exchange = ExchangeName, durable = true},
      #'exchange.declare_ok'{} = amqp_channel:call(Channel, Declare),
      amqp_channel:close(Channel).
    

之前的代码中，我们已经注册了处理 "client.connected" 事件的钩子函数，那么在事件发生时，应该向 exchange
中发布一条数据routing_key 为 "client.connected"：

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook.erl
    on_client_connected(#{client_id := ClientId, username := Username}, ConnAck, ConnInfo, _Env) ->
      {IpAddr, _Port} = maps:get(peername, ConnInfo),
      Doc = {
        client_id, ClientId,
        username, Username,
        keepalive, maps:get(keepalive, ConnInfo),
        ipaddress, iolist_to_binary(ntoa(IpAddr)),
        proto_ver, maps:get(proto_ver, ConnInfo),
        connected_at, emqx_time:now_ms(maps:get(connected_at, ConnInfo)),
        conn_ack, ConnAck
      },
      emqx_rabbitmq_hook_cli:publish(bson_binary:put_document(Doc), <<"client.connected">>),
      ok.
    

注意这里我们使用 "emqx_time:now_ms" 获取了消息以毫秒为单位的到达时间，比使用 Webhook 获取的 ts 更加精确了。

> `<<"client.connected">>`代表用一个字符串生成的二进制数据。

发布时，同样是从连接池中取一个连接进行操作：

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook.erl
    publish(Payload, RoutingKey) ->
      publish(<<"mqtt.events">>, Payload, RoutingKey).
    
    publish(ExchangeName, Payload, RoutingKey) ->
      ecpool:with_client(?APP, fun(C) -> publish(ExchangeName, Payload, RoutingKey, C) end).
    
    publish(ExchangeName, Payload, RoutingKey, Conn) ->
      {ok, Channel} = amqp_connection:open_channel(Conn),
      Publish = #'basic.publish'{exchange = ExchangeName, routing_key = RoutingKey},
      Props = #'P_basic'{delivery_mode = 2},
      Msg = #amqp_msg{props = Props, payload = Payload},
      amqp_channel:cast(Channel, Publish, Msg),
      amqp_channel:close(Channel).
    

最后需要在 `emqx_rabbitmq_hook.app.src` 中配置插件运行需要的信息：

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook.app.src
    {application,emqx_rabbitmq_hook,
                 [{description,"EMQ X RabbitMQ Hook"},
                  {vsn,"0.0.1"},
                  {modules,[]},
                  {registered,[emqx_rabbitmq_hook_sup]},
                  {applications,[kernel,stdlib,ecpool,amqp_client,bson]},
                  {mod,{emqx_rabbitmq_hook_app,[]}}]}.
    

### 编译插件

上一节，我们已经学会了如何编译插件。不过有一点不同的是，如果你新增了一个插件，那么这个插件就只能和一同编译出来的 emqx binaries
一起发布使用，不能只是把插件的 binary 复制到已经安装好的 emqx 的 plugins 目录下，否则的话，插件是无法使用的。

但是修改一个已发布的插件代码，编译以后就无需再发布一次 emqx binaries 了。只需要将插件的 binary 复制过来就可以了。

在编译 emqx-rabbitmq-hook 时， 需要到"rebar.config"去添加如下内容：

    
    
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
    

编译完成以后，可发布的 emqx binaries 和插件会被放到 `emqx-rel/ _build/emqx/rel/emqx/`
目录下，里面包含了完整的 EMQ X 的文件和目录结构。我们运行这个目录下的 EMQ X 就可以测试刚编写的插件了：

运行 emqx：`emqx-rel/ _build/emqx/rel/emqx/bin/emqx console`；

加载 emqx-rabbitmq-hook： `emqx-rel/ _build/emqx/rel/emqx/bin/emqx_ctl plugins
load emqx_rabbitmq_hook`。

如果不出意外的话，可以得到下面输出：

    
    
    Start apps: [emqx_rabbitmq_hook]
    Plugin emqx_rabbitmq_hook loaded successfully.
    

### 测试插件

最后我们写一段 RabbitMQ Client 的代码测试一下插件：

    
    
    require('dotenv').config()
    const bson = require('bson')
    var amqp = require('amqplib/callback_api');
    var exchange = "mqtt.events"
    amqp.connect(process.env.RABBITMQ_URL, function (error0, connection) {
        if (error0) {
            console.log(error0);
        } else {
            connection.createChannel(function (error1, channel) {
                if (error1) {
                    console.log(error1)
                } else {
                    var queue = "iothub_client_connected";
                    channel.assertQueue(queue, {
                        durable: true
                    })
                    channel.bindQueue(queue, exchange, "client.connected")
                    channel.consume(queue, function (msg) {
                        var data = bson.deserialize(msg.content)
                        console.log(`received: ${JSON.stringify(data)}`)
                        channel.ack(msg)
                    })
                }
            });
        }
    });
    

运行这段代码，接着使用任意的 MQTT Client 连接到 Broker，比如：`mosquitto_sub -t
"test/pc"`，然后我们会得到以下输出：

    
    
    received: {"client_id":"mosq/Rmkn7f4VZyUbeduN1t","username":null,"keepalive":60,"ipaddress":"127.0.0.1","proto_ver":4,"connected_at":1560250142384,"conn_ack":0}
    

那么 emqx-rabbitmq-hook 的主要流程就打通了。

* * *

这一节我们完成 emqx-rabbitmq-hook 的主要流程，下一节我们继续完成对其他事件的处理，并使用配置文件对插件进行配置。

