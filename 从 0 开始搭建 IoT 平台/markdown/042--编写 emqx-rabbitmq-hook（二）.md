这一课，我们继续完成处理其他事件的代码，但是因为本课程的篇幅有限，这里只完成 IotHub 目前需要的 "client.disconnected" 和
"message.publish" 事件的处理代码，其他事件的处理很简单，只需要依葫芦画瓢就可以了。有需要的话，大家可以自行进行扩展。

### 处理 "client.disconnected" 事件

这个事件的处理和 "client.connected" 事件，不过需要过滤掉 client 因为用户名和密码没有通过认证，触发的
"client.disconnected"：

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook.erl
    on_client_disconnected(#{}, auth_failure, _Env) ->
      ok;
    
    on_client_disconnected(#{client_id := ClientId, username := Username}, ReasonCode, _Env) ->
      Reason = if
                 is_atom(ReasonCode) ->
                   ReasonCode;
                 true ->
                   unknown
               end,
      Doc = {
        client_id, ClientId,
        username, Username,
        disconnected_at, emqx_time:now_ms(),
        reason, Reason
      },
      emqx_rabbitmq_hook_cli:publish(bson_binary:put_document(Doc), <<"client.disconnected">>),
      ok.
    

我们通过参数的模式匹配，没通过认证的 "client.disconnected" 事件会落入第一个 on_client_disconnected
函数中，不作任何处理。

### 处理 "message.publish" 事件

在处理这个事件时，需要过滤掉来自系统主题的 Publish 事件：

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook.erl
    on_message_publish(Message = #message{topic = <<"$SYS/", _/binary>>}, _Env) ->
      {ok, Message};
    
    on_message_publish(Message = #message{topic = Topic, flags = #{retain := Retain}}, _Env) ->
      Username = case maps:find(username, Message#message.headers) of
                   {ok, Value} -> Value;
                   _ -> undefined
                 end,
      Doc = {
        client_id, Message#message.from,
        username, Username,
        topic, Topic,
        qos, Message#message.qos,
        retained, Retain,
        payload, {bin, bin, Message#message.payload},
        published_at, emqx_time:now_ms(Message#message.timestamp)
      },
      emqx_rabbitmq_hook_cli:publish(bson_binary:put_document(Doc), <<"message.publish">>),
      {ok, Message}.
    

同样地，这里使用参数的模式匹配，来自于系统主题的 "message.publish" 事件会落入第一个 on_message_publish
函数中，不作任何处理。

这里使用 emqx_time:now_ms 函数获取到消息发布的以毫秒为单位的时间，这样可以解决之前 NTP 服务中以秒为单位而导致的计时不够精确的问题。

### 插件配置文件

#### .config 配置文件

插件的配置文件是放在`emqx_rabbitmq_hook/etc/` 下的，默认情况下是一个 Erlang 风格的 .config
文件，这种配置文件实际上就是 Erlang 的源文件，内容是一个 Erlang 的列表，例如：

    
    
    %% emqx_rabbitmq_hook/etc/emqx_rabbitmq_hook.config
    [
      {emqx_rabbitmq_hook, [{enabled, true}]}
    ].
    

EMQ X 在启动的时候会加载这个列表，在插件里可以通过下面的方式读取到这个列表里元素的值：

    
    
    (emqx@127.0.0.1)1> application:get_env(emqx_rabbitmq_hook, enabled).
    {ok,true}
    

这种风格的配置文件对 Erlang 用户来说是没什么问题的，但是对非 Erlang 的用户来说，可读性还是稍微差了一点，EMQ X 3.0 以后提供了非
Erlang 格式的 .conf 配置文件，我们在之前的课程中已经见到过了：

    
    
    xxx.xx.xx = xxx
    

这种配置文件需要配置一个映射规则，在 EMQ X 启动时通过
[cuttlefish](https://github.com/emqx/cuttlefish) 转换成上面的 Erlang 列表。接下来我们来看怎么做。

#### .conf 配置文件

#### 映射文件

以是否监听 "client.connected" 事件的配置为例，首先新增配置文件：

    
    
    ### emqx_rabbitmq_hook/etc/emqx_rabbitmq_hook.conf
    hook.rabbitmq.client.connected = on
    

然后，新增映射规则：

    
    
    %% emqx_rabbitmq_hook/priv/emqx_rabbitmq_hook.schema
    {mapping, "hook.rabbitmq.client.connected", "emqx_rabbitmq_hook.client_connected", [
      {default, on},
      {datatype, flag}
    ]}.
    

映射规则文件其实也是一个 Erlang 的源文件，上面的代码将 "hook.rabbitmq.client.connected"
进行映射，并指定它的默认值和类型。

在 emqx-rabbitmq-hook 的 Makefile 里面指明了用 cuttlefish 对配置文件和映射文件进行处理：

    
    
    emqx_rabbitmq_hook/Makefile
    app.config::
        ./deps/cuttlefish/cuttlefish -l info -e etc/ -c etc/emqx_rabbitmq_hook.conf -i priv/emqx_rabbitmq_hook.schema -d data
    

重新编译后，运行 `emqx-rel/_build/emqx/rel/emqx/bin/emqx
console`，在控制台中输入`application:get_env(emqx_rabbitmq_hook,
client_connected).`，就可以获取这个配置项的值：

    
    
    emqx@127.0.0.1)1> application:get_env(emqx_rabbitmq_hook, client_connected).
    {ok,true}
    

函数的参数 (emqx_rabbitmq_hook, client_connected)
和在映射文件里面配置的"emqx_rabbitmq_hook.client_connected"是对应的。

#### 更复杂的映射

在映射某些配置项时，还需要写一点代码，比如配置发布事件的 exchange 名时，RabbitMQ Erlang Client接受的 exchange
参数是二进制串，比如`<<mqtt.events>>`，而从 .conf 配置文件只能读取到字符串值，所以需要再做一个转化：

    
    
    %% emqx_rabbitmq_hook/priv/emqx_rabbitmq_hook.schema
    {mapping, "hook.rabbitmq.exchange", "emqx_rabbitmq_hook.exchange", [
      {default, "mqtt.events"},
      {datatype, string}
    ]}.
    {translation, "emqx_rabbitmq_hook.exchange", fun(Conf) ->
          list_to_binary(cuttlefish:conf_get("hook.rabbitmq.exchange", Conf))
    end}.
    

连接池 ecpool 的初始化方法接受的是一个配置项的列表，所以需要将配置文件中的 key-value 对转换成一个列表:

    
    
    %% emqx_rabbitmq_hook/priv/emqx_rabbitmq_hook.schema
    {translation, "emqx_rabbitmq_hook.server", fun(Conf) ->
        Pool = cuttlefish:conf_get("hook.rabbitmq.pool", Conf),
        Host = cuttlefish:conf_get("hook.rabbitmq.host", Conf),
        Port = cuttlefish:conf_get("hook.rabbitmq.port", Conf),
        [{pool_size, Pool},
         {host, Host},
         {port, Port}
        ]
    end}.
    

修改完配置映射文件后，需要重新编译。我们可以按照上面的规则继续添加更多的配置项。

### 使用配置项

最后一步是在代码里读取这些配置项，然后根据配置项的值进行相应操作。

**初始化连接池：**

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook_sub.erl
    init([]) ->
      {ok, PoolOpts} = application:get_env(?APP, server),
      PoolSpec = ecpool:pool_spec(?APP, ?APP, emqx_rabbitmq_hook_cli, PoolOpts),
      {ok, {{one_for_one, 10, 100}, [PoolSpec]}}.
    

**设置 exchange:**

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook_cli.erl
    ensure_exchange() ->
      {ok, ExchangeName} = application:get_env(?APP, exchange),
      ensure_exchange(ExchangeName).
    
    publish(Payload, RoutingKey) ->
      {ok, ExchangeName} = application:get_env(?APP, exchange),
      publish(ExchangeName, Payload, RoutingKey).  
    

**注册 Hook:** 这里首先实现一个根据配置进行 hook 注册的工具方法：

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook.erl
    hookup(Event, ConfigName, Func, InitArgs) ->
      case application:get_env(?APP, ConfigName) of
        {ok, true} -> emqx:hook(Event, Func, InitArgs);
        _ -> ok
      end.
    

然后在插件加载时调用：

    
    
    %% emqx_rabbitmq_hook/src/emqx_rabbitmq_hook.erl
    load(Env) ->
      ...
      hookup('client.connected', client_connected, fun ?MODULE:on_client_connected/4, [Env]),
      hookup('client.disconnected', client_disconnected, fun ?MODULE:on_client_disconnected/3, [Env]),
      hookup('message.publish', message_publish, fun ?MODULE:on_message_publish/2, [Env]).
    

重新编译之后，加载插件，修改几个配置项以后再重新加载插件，可以观察配置项是否生效。

* * *

这一节我们完成了 emqx-rabbitmq-hook 的全部功能，下一节我们将在 IotHub 中使用 emqx-rabbitmq-hook。

