在 5.1 节中对 Flink Table API & SQL 的概述和常见 API 都做了介绍，这篇文章先来看下其与 DataStream 和
DataSet API 的集成。

### Flink Table 和 SQL 与 DataStream 和 DataSet 集成

两个 planner 都可以与 DataStream API 集成，只有以前的 planner 才可以集成 DataSet API，所以下面讨论
DataSet API 都是和以前的 planner 有关。

Table API & SQL 查询与 DataStream 和 DataSet 程序集成是非常简单的，比如可以通过 Table API 或者 SQL
查询外部表数据，进行一些预处理后，然后使用 DataStream 或 DataSet API 继续处理一些复杂的计算，另外也可以将 DataStream 或
DataSet 处理后的数据利用 Table API 或者 SQL 写入到外部表去。总而言之，它们之间互相转换或者集成比较容易。

#### Scala 的隐式转换

Scala Table API 提供了 DataSet、DataStream 和 Table 类的隐式转换，可以通过导入
org.apache.flink.table.api.scala._ 或者 org.apache.flink.api.scala._ 包来启用这些转换。

#### 将 DataStream 或 DataSet 注册为 Table

DataStream 或者 DataSet 可以注册为 Table，结果表的 schema 取决于已经注册的 DataStream 和 DataSet
的数据类型。你可以像下面这种方式转换：

    
    
    StreamTableEnvironment tableEnv = ...;
    
    DataStream<Tuple2<Long, String>> stream = ...
    
    //将 DataStream 注册为 myTable 表
    tableEnv.registerDataStream("myTable", stream);
    
    //将 DataStream 注册为 myTable2 表（表中的字段为 myLong、myString）
    tableEnv.registerDataStream("myTable2", stream, "myLong, myString");
    

#### 将 DataStream 或 DataSet 转换为 Table

除了可以将 DataStream 或 DataSet 注册为 Table，还可以将它们转换为 Table，转换之后再去使用 Table API
查询就比较方便了。

    
    
    StreamTableEnvironment tableEnv = ...;
    
    DataStream<Tuple2<Long, String>> stream = ...
    
    //将 DataStream 转换成 Table
    Table table1 = tableEnv.fromDataStream(stream);
    
    //将 DataStream 转换成 Table
    Table table2 = tableEnv.fromDataStream(stream, "myLong, myString");
    

#### 将 Table 转换成 DataStream 或 DataSet

Table 可以转换为 DataStream 或 DataSet，这样就可以在 Table API 或 SQL 查询的结果上运行自定义的
DataStream 或 DataSet 程序。当将一个 Table 转换成 DataStream 或 DataSet 时，需要指定结果
DataStream 或 DataSet 的数据类型，最方便的数据类型是 Row，下面几个数据类型表示不同的功能：

  * Row：字段按位置映射，任意数量的字段，支持 null 值，没有类型安全访问。
  * POJO：字段按名称映射，POJO 属性必须按照 Table 中的属性来命名，任意数量的字段，支持 null 值，类型安全访问。
  * Case Class：字段按位置映射，不支持 null 值，类型安全访问。
  * Tuple：按位置映射字段，限制为 22（Scala）或 25（Java）字段，不支持 null 值，类型安全访问。
  * 原子类型：Table 必须具有单个字段，不支持 null 值，类型安全访问。

##### 将 Table 转换成 DataStream

流查询的结果表会动态更新，即每个新的记录到达输入流时结果就会发生变化。所以在将 Table 转换成 DataStream 就需要对表的更新进行编码，有两种将
Table 转换为 DataStream 的模式：

  * 追加模式（Append Mode）：这种模式只能在动态表仅通过 INSERT 更改修改时才能使用，即仅追加，之前发出的结果不会更新。
  * 撤回模式（Retract Mode）：任何时刻都可以使用此模式，它使用一个 boolean 标志来编码 INSERT 和 DELETE 的更改。

    
    
    StreamTableEnvironment tableEnv = ...;
    
    //有两个字段(name、age) 的 Table
    Table table = ...
    
    //通过指定类，将表转换为一个 append DataStream
    DataStream<Row> dsRow = tableEnv.toAppendStream(table, Row.class);
    
    //将表转换为 Tuple2<String, Integer> 的 append DataStream
    TupleTypeInfo<Tuple2<String, Integer>> tupleType = new TupleTypeInfo<>(Types.STRING(), Types.INT());
    DataStream<Tuple2<String, Integer>> dsTuple = tableEnv.toAppendStream(table, tupleType);
    
    //将表转换为一个 Retract DataStream Row
    DataStream<Tuple2<Boolean, Row>> retractStream = tableEnv.toRetractStream(table, Row.class);
    

##### 将 Table 转换成 DataSet

将 Table 转换成 DataSet 的样例如下：

    
    
    BatchTableEnvironment tableEnv = BatchTableEnvironment.create(env);
    
    //有两个字段(name、age) 的 Table
    Table table = ...
    
    //通过指定一个类将表转换为一个 Row DataSet
    DataSet<Row> dsRow = tableEnv.toDataSet(table, Row.class);
    
    //将表转换为 Tuple2<String, Integer> 的 DataSet
    TupleTypeInfo<Tuple2<String, Integer>> tupleType = new TupleTypeInfo<>(Types.STRING(), Types.INT());
    DataSet<Tuple2<String, Integer>> dsTuple = tableEnv.toDataSet(table, tupleType);
    

### 查询优化

Flink 使用 Calcite 来优化和翻译查询，以前的 planner 不会去优化 join 的顺序，而是按照查询中定义的顺序去执行。通过提供一个
CalciteConfig 对象来调整在不同阶段应用的优化规则集，这个可以通过调用 CalciteConfig.createBuilder() 获得的
builder 来创建，并且可以通过调用tableEnv.getConfig.setCalciteConfig(calciteConfig) 来提供给
TableEnvironment。而在 Blink planner 中扩展了 Calcite 来执行复杂的查询优化，这包括一系列基于规则和成本的优化，比如：

  * 基于 Calcite 的子查询去相关性
  * Project pruning
  * Partition pruning
  * Filter push-down
  * 删除子计划中的重复数据以避免重复计算
  * 重写特殊的子查询，包括两部分：
    * 将 IN 和 EXISTS 转换为 left semi-joins
    * 将 NOT IN 和 NOT EXISTS 转换为 left anti-join
  * 重排序可选的 join
    * 通过启用 table.optimizer.join-reorder-enabled

注意：IN/EXISTS/NOT IN/NOT EXISTS 目前只支持子查询重写中的连接条件。

#### 解释 Table

Table API 提供了一种机制来解释计算 Table 的逻辑和优化查询计划。你可以通过 TableEnvironment.explain(table)
或者 TableEnvironment.explain() 方法来完成。explain(table) 会返回给定计划的 Table，explain()
会返回多路 Sink 计划的结果（主要用于 Blink planner）。它返回一个描述三个计划的字符串：

  * 关系查询的抽象语法树，即未优化的逻辑查询计划
  * 优化的逻辑查询计划
  * 实际执行计划

以下代码演示了一个 Table 示例：

    
    
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    StreamTableEnvironment tEnv = StreamTableEnvironment.create(env);
    
    DataStream<Tuple2<Integer, String>> stream1 = env.fromElements(new Tuple2<>(1, "hello"));
    DataStream<Tuple2<Integer, String>> stream2 = env.fromElements(new Tuple2<>(1, "hello"));
    
    Table table1 = tEnv.fromDataStream(stream1, "count, word");
    Table table2 = tEnv.fromDataStream(stream2, "count, word");
    Table table = table1.where("LIKE(word, 'F%')").unionAll(table2);
    
    System.out.println(tEnv.explain(table));
    

通过 explain(table) 方法返回的结果：

    
    
    == Abstract Syntax Tree ==
    LogicalUnion(all=[true])
      LogicalFilter(condition=[LIKE($1, _UTF-16LE'F%')])
        FlinkLogicalDataStreamScan(id=[1], fields=[count, word])
      FlinkLogicalDataStreamScan(id=[2], fields=[count, word])
    
    == Optimized Logical Plan ==
    DataStreamUnion(all=[true], union all=[count, word])
      DataStreamCalc(select=[count, word], where=[LIKE(word, _UTF-16LE'F%')])
        DataStreamScan(id=[1], fields=[count, word])
      DataStreamScan(id=[2], fields=[count, word])
    
    == Physical Execution Plan ==
    Stage 1 : Data Source
        content : collect elements with CollectionInputFormat
    
    Stage 2 : Data Source
        content : collect elements with CollectionInputFormat
    
        Stage 3 : Operator
            content : from: (count, word)
            ship_strategy : REBALANCE
    
            Stage 4 : Operator
                content : where: (LIKE(word, _UTF-16LE'F%')), select: (count, word)
                ship_strategy : FORWARD
    
                Stage 5 : Operator
                    content : from: (count, word)
                    ship_strategy : REBALANCE
    

### 数据类型

在 Flink 1.9 之前，Flink 的 Table API&SQL 的数据类型与 Flink 中的 TypeInformation
紧密相关。TypeInformation 在 DataStream 和 DataSet API 中使用，另外它还可以描述在分布式中序列化和反序列化基于
JVM 对象所需的所有信息。从 1.9 版本之后，Table API&SQL 会引入一种新类型来作为 API 稳定性和标准的长期解决方案。在以前的
planner 和 Blink planner 的数据类型有点不一致，具体差别可以参考官网。

### 时间属性

在 3.1 节中介绍过 Flink 的多种时间语义，常用的比如 Event time 和 Processing time，那么在 Table API&SQL
中怎么去定义时间语义呢？

#### Processing Time

因为处理时间是额外的数据字段，在原始的事件中是不存在该字段的，那么在将数据流转换成 Table 的时候就需要将这个 Processing time 当作
Table 的一个字段，以供后面需要，比如定义窗口。你可以像下面这样定义：

    
    
    DataStream<Tuple2<String, String>> stream = ...;
    
    //将附加的逻辑字段声明为 Processing time 属性
    Table table = tEnv.fromDataStream(stream, "Username, Data, UserActionTime.proctime");
    
    WindowedTable windowedTable = table.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
    

如果是直接使用 TableSource 的话，那么需要实现 DefinedProctimeAttribute 接口，然后去重写
getProctimeAttribute 方法，返回的字符串表示 Processing time 在 Table 中的字段名。

#### Event time

Event time 是在采集上来的事件中就有的，将数据流转换成 Table 的时候需要像下面这样定义：

    
    
    //第一种方法：
    //提取流数据时间戳并分配水印
    DataStream<Tuple2<String, String>> stream = inputStream.assignTimestampsAndWatermarks(...);
    //将附加的逻辑字段声明为 Event time 属性，和 Processing time 不同的是这里使用 rowtime
    Table table = tEnv.fromDataStream(stream, "Username, Data, UserActionTime.rowtime");
    
    //第二种方法：
    //从第一个字段提取时间戳，并分配水印
    DataStream<Tuple3<Long, String, String>> stream = inputStream.assignTimestampsAndWatermarks(...);
    Table table = tEnv.fromDataStream(stream, "UserActionTime.rowtime, Username, Data");
    
    //使用方式：
    WindowedTable windowedTable = table.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
    

使用 TableSource 的话则需要实现 DefinedRowtimeAttributes 接口，重写
getRowtimeAttributeDescriptors 方法，该方法返回一个 RowtimeAttributeDescriptor
列表，其用于描述时间属性的最终名称、时间提取器以及该属性关联的水印策略。

### SQL Connector

在第三部分中介绍了大量的 Flink Connectors 的使用，但是那些都是通过 DataStream API 是去使用，放在 Table
API&SQL 中其实不再适合，其实 Flink Table API&SQL
是可以直接连接到外部系统的，然后读取和写入批处理表和流处理表。TableSource 提供从外部系统（数据库、MQ、文件系统等）读取数据，TableSink
将结果存储到数据库中。这里讲解一下该如何去定义 TableSource 和 TableSink 并将它们注册。在官网，它提供了如下这些 Connectors
和 Formats 的下载。

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-11-04-071612.png)

从 Flink 1.6 开始，不仅可以使用编程的方式指定 Connector，还可以使用声明式去定义。下面举个例子（读取 Kafka 中 Avro
格式的数据）来讲解这两种区别。

#### 使用代码

    
    
    tableEnvironment
      //声明要连接的外部系统
      .connect(
        new Kafka()
          .version("0.10")
          .topic("zhisheng_user")
          .startFromEarliest()
          .property("zookeeper.connect", "localhost:2181")
          .property("bootstrap.servers", "localhost:9092")
      )
      //定义数据格式
      .withFormat(
        new Avro()
          .avroSchema(
            "{" +
            "  \"namespace\": \"com.zhisheng\"," +
            "  \"type\": \"record\"," +
            "  \"name\": \"UserMessage\"," +
            "    \"fields\": [" +
            "      {\"name\": \"timestamp\", \"type\": \"string\"}," +
            "      {\"name\": \"user\", \"type\": \"long\"}," +
            "      {\"name\": \"message\", \"type\": [\"string\", \"null\"]}" +
            "    ]" +
            "}"
          )
      )
      //定义 Table schema
      .withSchema(
        new Schema()
          .field("rowtime", Types.SQL_TIMESTAMP)
            .rowtime(new Rowtime()
              .timestampsFromField("timestamp")
              .watermarksPeriodicBounded(60000)
            )
          .field("user", Types.LONG)
          .field("message", Types.STRING)
      )
      .inAppendMode()  //指定流表的 update-mode
      .registerTableSource("zhisheng");    //注册表的名字
    

#### 使用 YAML 文件

    
    
    tables:
      - name: zhisheng      #表的名字
        type: source           #定义是 source，还是 sink，或者 both
        update-mode: append    #指定流表的 update-mode
        #定义要连接的系统
        connector:
          type: kafka
          version: "0.10"
          topic: zhisheng_user
          startup-mode: earliest-offset
          properties:
            - key: zookeeper.connect
              value: localhost:2181
            - key: bootstrap.servers
              value: localhost:9092
    
        #定义格式
        format:
          type: avro
          avro-schema: >
            {
              "namespace": "com.zhisheng",
              "type": "record",
              "name": "UserMessage",
                "fields": [
                  {"name": "ts", "type": "string"},
                  {"name": "user", "type": "long"},
                  {"name": "message", "type": ["string", "null"]}
                ]
            }
        #定义 table schema
        schema:
          - name: rowtime
            type: TIMESTAMP
            rowtime:
              timestamps:
                type: from-field
                from: ts
              watermarks:
                type: periodic-bounded
                delay: "60000"
          - name: user
            type: BIGINT
          - name: message
            type: VARCHAR
    

#### 使用 DDL

    
    
    CREATE TABLE zhisheng (
      `user` BIGINT,
      message VARCHAR,
      ts VARCHAR
    ) WITH (
      'connector.type' = 'kafka',
      'connector.version' = '0.10',
      'connector.topic' = 'zhisheng_user',
      'connector.startup-mode' = 'earliest-offset',
      'connector.properties.0.key' = 'zookeeper.connect',
      'connector.properties.0.value' = 'localhost:2181',
      'connector.properties.1.key' = 'bootstrap.servers',
      'connector.properties.1.value' = 'localhost:9092',
      'update-mode' = 'append',
      'format.type' = 'avro',
      'format.avro-schema' = '{
                                "namespace": "com.zhisheng",
                                "type": "record",
                                "name": "UserMessage",
                                "fields": [
                                    {"name": "ts", "type": "string"},
                                    {"name": "user", "type": "long"},
                                    {"name": "message", "type": ["string", "null"]}
                                ]
                             }'
    )
    

上面演示了 Kafka Connector 和 avro 数据格式化在 Table API&SQL 中的使用方式，在官网中还有文件系统和
Elasticsearch Connector、CSV 和 JSON 等的使用说明。

### SQL Client

虽然 Flink Table API&SQL 让使用 SQL 去查询流数据有了可能，但是这些查询语句通常要嵌入在 Java 或者 Scala
程序中，最后在提交到集群运行之前还要通过构建工具打包，这就导致 Table API&SQL 的限制性很大，所以 SQL Client
就起到这么个作用，让用户不再编写任何 Java 或者 Scala 代码，直接编写 SQL
就可以去调试运行，并且可以通过其他命令行实时查看运行的结果，但是该功能目前还比较弱。

在启动 Flink 后可以通过运行 `./bin/sql-client.sh embedded` 命令来启动 SQL Client CLI，如下图所示：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-11-04-075132.png)

你可以运行下面的命令就可以知道名字和其出现的次数的结果。

    
    
    SELECT name, COUNT(*) AS cnt FROM (VALUES ('Bob'), ('Alice'), ('Greg'), ('Bob')) AS NameTable(name) GROUP BY name;
    

另外它还支持传入 YAML 文件，你可以在 YAML 文件中如前面内容一样定义的 Kafka Connector 等信息，关于 SQL Client
的更多功能可以查阅官网。

### Hive

Hive 是建立在 Hadoop 上的数据仓库基础构架，它提供了一系列的工具，可以用来进行数据提取转化加载（ETL），这是一种可以存储、查询和分析存储在
Hadoop 中的大规模数据的机制。Hive 定义了简单的类 SQL 查询语言，称为 HQL，它允许熟悉 SQL 的用户查询数据。

Flink 在 1.9 版本中提供了与 Hive 的双重集成。首先是利用 Hive 的 Metastore 存储 Flink 特定元数据，另一个是
Flink 支持读取和写入 Hive 表。支持的 Hive 2.3.4 和 1.2.1 版本，如果你要使用的话，注意它们的依赖是有点不一样。

你可以通过 Java、Scala、YAML 连接 Hive，比如使用 Java 代码如下：

    
    
    String name            = "myhive";
    String defaultDatabase = "mydatabase";
    String hiveConfDir     = "/opt/hive-conf";
    String version         = "2.3.4"; //或者 1.2.1
    
    HiveCatalog hive = new HiveCatalog(name, defaultDatabase, hiveConfDir, version);
    tableEnv.registerCatalog("myhive", hive);
    

### 小结与反思

本节继续介绍了 Flink Table API&SQL 中的部分 API，然后讲解了 Flink 之前的 planner 和 Blink planner
在某些特性上面的区别，还讲解了 SQL Connector，最后介绍了 SQL Client 和 Hive。

