前面的内容都是讲解 DataStream 和 DataSet API 相关的，在 1.2.5 节中讲解 Flink API 时提及到 Flink 的高级
API——Table API&SQL，本节将开始 Table&SQL 之旅。

### 新增 Blink SQL 查询处理器

在 Flink 1.9 版本中，合进了阿里巴巴开源的 Blink 版本中的大量代码，其中最重要的贡献就是 Blink SQL 了。在 Blink 捐献给
Apache Flink 之后，社区就致力于为 Table API&SQL 集成 Blink 的查询优化器和 runtime。先来看下 1.8 版本的
Flink Table 项目结构如下图：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-30-130607.png)

1.9 版本的 Flink Table 项目结构图如下：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-30-130751.png)

可以发现新增了 flink-sql-parser、flink-table-planner-blink、flink-table-runtime-
blink、flink-table-uber-blink 模块，对 Flink Table 模块的重构详细内容可以参考
[FLIP-32](https://cwiki.apache.org/confluence/display/FLINK/FLIP-32%3A+Restructure+flink-
table+for+future+contributions)。这样对于 Java 和 Scala API 模块、优化器以及 runtime
模块来说，分层更清楚，接口更明确。

另外 flink-table-planner-blink 模块中实现了新的优化器接口，所以现在有两个插件化的查询处理器来执行 Table
API&SQL：1.9 以前的 Flink 处理器和新的基于 Blink 的处理器。基于 Blink 的查询处理器提供了更好的 SQL
覆盖率、支持更广泛的查询优化、改进了代码生成机制、通过调优算子的实现来提升批处理查询的性能。除此之外，基于 Blink
的查询处理器还提供了更强大的流处理能力，包括了社区一些非常期待的新功能（如维表
Join、TopN、去重）和聚合场景缓解数据倾斜的优化，以及内置更多常用的函数，具体可以查看 flink-table-runtime-blink
代码。目前整个模块的结构如下：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-30-124512.png)

注意：两个查询处理器之间的语义和功能大部分是一致的，但未完全对齐，因为基于 Blink 的查询处理器还在优化中，所以在 1.9 版本中默认查询处理器还是
1.9 之前的版本。如果你想使用 Blink 处理器的话，可以在创建 TableEnvironment 时通过 EnvironmentSettings
配置启用。被选择的处理器必须要在正在执行的 Java 进程的类路径中。对于集群设置，默认两个查询处理器都会自动地加载到类路径中。如果要在 IDE
中运行一个查询，需要在项目中添加 planner 依赖。

### 为什么选择 Table API&SQL？

在 1.2 节中介绍了 Flink 的 API 是包含了 Table API&SQL，在 1.3 节中也介绍了在 Flink 1.9 中阿里开源的
Blink 分支中的很强大的 SQL 功能合并进 Flink 主分支，另外通过阿里 Blink 相关的介绍，可以知道阿里在 SQL
功能这块是做了很多的工作。从前面章节的内容可以发现 Flink 的 DataStream/DataSet API
的功能已经很全并且很强大了，常见复杂的数据处理问题也都可以处理，那么社区为啥还在一直推广 Table API&SQL 呢？

其实通过观察其它的大数据组件，就不会好奇了，比如 Spark、Storm、Beam、Hive 、KSQL（面向 Kafka 的 SQL
引擎）、Elasticsearch、Phoenix（使用 SQL 进行 HBase 数据的查询）等，可以发现 SQL
已经成为各个大数据组件必不可少的数据查询语言，那么 Flink 作为一个大数据实时处理引擎，笔者对其支持 SQL
查询流数据也不足为奇了，但是还是来稍微介绍一下 Table API&SQL。

Table API&SQL 是一种关系型 API，用户可以像操作数据库一样直接操作流数据，而不再需要通过 DataStream API
来写很多代码完成计算需求，更不用手动去调优你写的代码，另外 SQL
最大的优势在于它是一门学习成本很低的语言，普及率很高，用户基数大，和其他的编程语言相比，它的入门相对简单。

除了上面的原因，还有一个原因是：可以借助 Table API&SQL 统一流处理和批处理，因为在 DataStream/DataSet API
中，用户开发流作业和批作业需要去了解两种不同的 API，这对于公司有些开发能力不高的数据分析师来说，学习成本有点高，他们其实更擅长写 SQL
来分析。Table API&SQL 做到了批与流上的查询具有同样的语法语义，因此不用改代码就能同时在批和流上执行。

总结来说，为什么选择 Table API&SQL：

  * 声明式语言表达业务逻辑
  * 无需代码编程——易于上手
  * 查询能够被有效的优化
  * 查询可以高效的执行

### Flink Table 项目模块

在上文中提及到 Flink Table 在 1.8 和 1.9
的区别，这里还是要再讲解一下这几个依赖，因为只有了解清楚了之后，我们在后面开发的时候才能够清楚挑选哪种依赖。它有如下几个模块：

  * flink-table-common：table 中的公共模块，可以用于通过自定义 function，format 等来扩展 Table 生态系统
  * flink-table-api-java：支持使用 Java 语言，纯 Table＆SQL API
  * flink-table-api-scala：支持使用 Scala 语言，纯 Table＆SQL API
  * flink-table-api-java-bridge：支持使用 Java 语言，包含 DataStream/DataSet API 的 Table＆SQL API（推荐使用）
  * flink-table-api-scala-bridge：支持使用 Scala 语言，带有 DataStream/DataSet API 的 Table＆SQL API（推荐使用）
  * flink-sql-parser：SQL 语句解析层，主要依赖 calcite
  * flink-table-planner：Table 程序的 planner 和 runtime
  * flink-table-uber：将上诉模块打成一个 fat jar，在 lib 目录下
  * flink-table-planner-blink：Blink 的 Table 程序的 planner（阿里开源的版本）
  * flink-table-runtime-blink：Blink 的 Table 程序的 runtime（阿里开源的版本）
  * flink-table-uber-blink：将 Blink 版本的 planner 和 runtime 与前面模块（除 flink-table-planner 模块）打成一个 fat jar，在 lib 目录下 

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-11-02-164352.png)

  * flink-sql-client：SQL 客户端

### 两种 planner 之间的区别

上面讲了两种不同的 planner 之间包含的模块有点区别，但是具体有什么区别如下所示：

  * Blink planner 将批处理作业视为流的一种特殊情况。因此不支持 Table 和 DataSet 之间的转换，批处理作业会转换成 DataStream 程序，而不会转换成 DataSet 程序，流作业还是转换成 DataStream 程序。
  * Blink planner 不支持 BatchTableSource，而是使用有界的（bounded） StreamTableSource 代替它。
  * Blink planner 仅支持全新的 Catalog，不支持已经废弃的 ExternalCatalog。
  * 以前的 planner 中 FilterableTableSource 的实现与现在的 Blink planner 有冲突，在以前的 planner 中是叠加 PlannerExpressions（在未来的版本中会移除），而在 Blink planner 中是 Expressions。
  * 基于字符串的 KV 键值配置选项仅可以在 Blink planner 中使用。
  * PlannerConfig 的实现（CalciteConfig）在两种 planner 中不同。
  * Blink planner 会将多个 sink 优化在同一个 DAG 中（只在 TableEnvironment 中支持，StreamTableEnvironment 中不支持），而以前的 planner 是每个 sink 都有一个 DAG 中，相互独立的。
  * 以前的 planner 不支持 catalog 统计，而 Blink planner 支持。

在了解到了两种 planner 的区别后，接下来开始 Flink Table API&SQL 之旅。

### 添加项目依赖

因为在 Flink 1.9 版本中有两个 planner，所以得根据你使用的 planner 来选择对应的依赖，假设你选择的是最新的 Blink
版本，那么添加下面的依赖：

    
    
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-table-planner-blink_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
    </dependency>
    

如果是以前的 planner，则使用下面这个依赖：

    
    
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-table-planner_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
    </dependency>
    

如果要自定义 format 格式或者自定义 function，则需要添加 flink-table-common 依赖：

    
    
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-table-common</artifactId>
      <version>${flink.version}</version>
    </dependency>
    

### 创建一个 TableEnvironment

TableEnvironment 是 Table API 和 SQL 的统称，它负责的内容有：

  * 在内部的 catalog 注册 Table
  * 注册一个外部的 catalog
  * 执行 SQL 查询
  * 注册用户自定义的 function
  * 将 DataStream 或者 DataSet 转换成 Table
  * 保持对 ExecutionEnvironment 和 StreamExecutionEnvironment 的引用

Table 总是会绑定在一个指定的 TableEnvironment，不能在同一个查询中组合不同 TableEnvironment 的 Table，比如
join 或 union 操作。你可以使用下面的几种静态方法创建 TableEnvironment。

    
    
    //创建 StreamTableEnvironment
    static StreamTableEnvironment create(StreamExecutionEnvironment executionEnvironment) {
        return create(executionEnvironment, EnvironmentSettings.newInstance().build());
    }
    
    static StreamTableEnvironment create(StreamExecutionEnvironment executionEnvironment, EnvironmentSettings settings) {
        return StreamTableEnvironmentImpl.create(executionEnvironment, settings, new TableConfig());
    }
    
    /** @deprecated */
    @Deprecated
    static StreamTableEnvironment create(StreamExecutionEnvironment executionEnvironment, TableConfig tableConfig) {
        return StreamTableEnvironmentImpl.create(executionEnvironment, EnvironmentSettings.newInstance().build(), tableConfig);
    }
    
    //创建 BatchTableEnvironment
    static BatchTableEnvironment create(ExecutionEnvironment executionEnvironment) {
        return create(executionEnvironment, new TableConfig());
    }
    
    static BatchTableEnvironment create(ExecutionEnvironment executionEnvironment, TableConfig tableConfig) {
        //
    }
    

你需要根据你的程序来使用对应的 TableEnvironment，是 BatchTableEnvironment 还是
StreamTableEnvironment。默认两个 planner 都是在 Flink 的安装目录下 lib
文件夹中存在的，所以应该在你的程序中指定使用哪种 planner。

    
    
    // Flink Streaming query
    import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
    import org.apache.flink.table.api.EnvironmentSettings;
    import org.apache.flink.table.api.java.StreamTableEnvironment;
    EnvironmentSettings fsSettings = EnvironmentSettings.newInstance().useOldPlanner().inStreamingMode().build();
    StreamExecutionEnvironment fsEnv = StreamExecutionEnvironment.getExecutionEnvironment();
    StreamTableEnvironment fsTableEnv = StreamTableEnvironment.create(fsEnv, fsSettings);
    //或者 TableEnvironment fsTableEnv = TableEnvironment.create(fsSettings);
    
    // Flink Batch query
    import org.apache.flink.api.java.ExecutionEnvironment;
    import org.apache.flink.table.api.java.BatchTableEnvironment;
    ExecutionEnvironment fbEnv = ExecutionEnvironment.getExecutionEnvironment();
    BatchTableEnvironment fbTableEnv = BatchTableEnvironment.create(fbEnv);
    
    // Blink Streaming query
    import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
    import org.apache.flink.table.api.EnvironmentSettings;
    import org.apache.flink.table.api.java.StreamTableEnvironment;
    StreamExecutionEnvironment bsEnv = StreamExecutionEnvironment.getExecutionEnvironment();
    EnvironmentSettings bsSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
    StreamTableEnvironment bsTableEnv = StreamTableEnvironment.create(bsEnv, bsSettings);
    //或者 TableEnvironment bsTableEnv = TableEnvironment.create(bsSettings);
    
    // Blink Batch query
    import org.apache.flink.table.api.EnvironmentSettings;
    import org.apache.flink.table.api.TableEnvironment;
    EnvironmentSettings bbSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inBatchMode().build();
    TableEnvironment bbTableEnv = TableEnvironment.create(bbSettings);
    

如果在 lib 目录下只存在一个 planner，则可以使用 useAnyPlanner 来创建指定的 EnvironmentSettings。

### Table API&SQL 应用程序的结构

批处理和流处理的 Table API&SQL 作业都有相同的模式，它们的代码结构如下：

    
    
    //根据前面内容创建一个 TableEnvironment，指定是批作业还是流作业
    TableEnvironment tableEnv = ...; 
    
    //用下面的其中一种方式注册一个 Table
    tableEnv.registerTable("table1", ...)          
    tableEnv.registerTableSource("table2", ...); 
    tableEnv.registerExternalCatalog("extCat", ...);
    
    //注册一个 TableSink
    tableEnv.registerTableSink("outputTable", ...);
    
    //根据一个 Table API 查询创建一个 Table
    Table tapiResult = tableEnv.scan("table1").select(...);
    //根据一个 SQL 查询创建一个 Table
    Table sqlResult  = tableEnv.sqlQuery("SELECT ... FROM table2 ... ");
    
    //将 Table API 或者 SQL 的结果发送给 TableSink
    tapiResult.insertInto("outputTable");
    
    //运行
    tableEnv.execute("java_job");
    

### Catalog 中注册 Table

Table 有两种类型，输入表和输出表，可以在 Table API&SQL 查询中引用输入表并提供输入数据，输出表可以用于将 Table API&SQL
的查询结果发送到外部系统。输出表可以通过 TableSink 来注册，输入表可以从各种数据源进行注册：

  * 已经存在的 Table 对象，通过是 Table API 或 SQL 查询的结果
  * 连接了外部系统的 TableSource，比如文件、数据库、MQ
  * 从 DataStream 或 DataSet 程序中返回的 DataStream 和 DataSet

#### 注册 Table

在 TableEnvironment 中可以像下面这样注册一个 Table：

    
    
    //创建一个 TableEnvironment
    TableEnvironment tableEnv = ...; // see "Create a TableEnvironment" section
    
    //projTable 是一个简单查询的结果
    Table projTable = tableEnv.scan("X").select(...);
    
    //将 projTable 表注册为 projectedTable 表
    tableEnv.registerTable("projectedTable", projTable);
    

#### 注册 TableSource

TableSource 让你可以访问存储系统（数据库 MySQL、HBase 等）、编码文件（CSV、Parquet、Avro 等）或
MQ（Kafka、RabbitMQ） 中的数据。Flink 为常用组件都提供了 TableSource，另外还提供自定义 TableSource。在
TableEnvironment 中可以像下面这样注册 TableSource：

    
    
    TableEnvironment tableEnv = ...;
    
    //创建 TableSource
    TableSource csvSource = new CsvTableSource("/Users/zhisheng/file", ...);
    
    //将 csvSource 注册为表
    tableEnv.registerTableSource("CsvTable", csvSource);
    

注意：用于 Blink planner 的 TableEnvironment 只能接受
StreamTableSource、LookupableTableSource 和 InputFormatTableSource，用于 Blink
planner 批处理的 StreamTableSource 必须是有界的。

#### 注册 TableSink

TableSink 可以将 Table API&SQL 查询的结果发送到外部的存储系统去，比如数据库、KV 存储、文件（CSV、Parquet 等）或 MQ
等。Flink 为常用等数据存储系统和文件格式都提供了 TableSink，另外还支持自定义 TableSink。在 TableEnvironment
中可以像下面这样注册 TableSink：

    
    
    TableEnvironment tableEnv = ...;
    
    //创建 TableSink
    TableSink csvSink = new CsvTableSink("/Users/zhisheng/file", ...);
    
    //定义属性名和类型
    String[] fieldNames = {"a", "b", "c"};
    TypeInformation[] fieldTypes = {Types.INT, Types.STRING, Types.LONG};
    
    //将 csvSink 注册为表 CsvSinkTable
    tableEnv.registerTableSink("CsvSinkTable", fieldNames, fieldTypes, csvSink);
    

### 注册外部的 Catalog

外部的 Catalog 可以提供外部的数据库和表的信息，例如它们的名称、schema、统计信息以及如何访问存储在外部数据库、表、文件中的数据。可以通过实现
ExternalCatalog 接口来创建外部的 Catalog，并像下面这样注册外部的 Catalog：

    
    
    TableEnvironment tableEnv = ...;
    
    //创建外部的 catalog
    ExternalCatalog catalog = new InMemoryExternalCatalog();
    //注册 ExternalCatalog
    tableEnv.registerExternalCatalog("InMemCatalog", catalog);//该方法已经标记过期，可以使用 Catalog
    
    //使用下面这种
    Catalog catalog = new GenericInMemoryCatalog("zhisheng");
    tableEnv.registerCatalog("InMemCatalog", catalog);
    

在注册后，ExternalCatalog 中的表数据信息可以通过 Table API&SQL 查询获取到。Flink 提供了 Catalog 的一种实现类
GenericInMemoryCatalog 用于样例和测试。

### 查询 Table

#### Table API

先来演示使用 Table API 来完成一个简单聚合查询：

    
    
    TableEnvironment tableEnv = ...;
    
    //注册 Orders 表
    
    //查询注册的 Orders 表
    Table orders = tableEnv.scan("Orders");
    //计算来自中国的顾客的收入
    Table revenue = orders
      .filter("cCountry === 'China'")
      .groupBy("cID, cName")
      .select("cID, cName, revenue.sum AS revSum");
    
    //转换或者提交该结果表
    //运行该查询语句
    

你可以使用 Java 或者 Scala 语言来利用 Table API 开发，而 SQL 却不是这样的。

#### SQL

上面使用 Table API 的聚合查询样例使用 SQL 来完成就如下面这样：

    
    
    TableEnvironment tableEnv = ...;
    
    //注册 Orders 表
    
    //计算来自中国的顾客的收入
    Table revenue = tableEnv.sqlQuery(
        "SELECT cID, cName, SUM(revenue) AS revSum " +
        "FROM Orders " +
        "WHERE cCountry = 'FRANCE' " +
        "GROUP BY cID, cName"
      );
    
    //转换或者提交该结果表
    //运行该查询语句
    

Flink 的 SQL 是基于实现 SQL 标准的 Apache Calcite，SQL 的查询语句就是全部为字符串，上面这条 SQL
就说明了该如何指定查询并返回结果表，下面演示如何更新。

    
    
    tableEnv.sqlUpdate(
        "INSERT INTO RevenueFrance " +
        "SELECT cID, cName, SUM(revenue) AS revSum " +
        "FROM Orders " +
        "WHERE cCountry = 'FRANCE' " +
        "GROUP BY cID, cName"
      );
    

#### Table API&SQL

Table API 和 SQL 之间可以相互结合，因为它们最后都是返回的 Table 对象，比如你可以在 SQL 查询返回的对象上定义 Table API
的查询，也可以在 Table API 查询结果返回的对象上定义 SQL 查询。

### 提交 Table

在前面讲解了注册 TableSink，那么将表的结果提交就是将 Table 写入 TableSink，批处理的 Table 只能写入到
BatchTableSink，而流处理的 Table 可以写入进
AppendStreamTableSink、RetractStreamTableSink、UpsertStreamTableSink。使用
Table.insertInto(String tableName) 方法就可以将 Table 写入进已注册的 TableSink，它会根据名字去
catalog 中查找，并对比两者的 schema 是否相同。

### 翻译并执行查询

对于两种不同的 planner，翻译和执行查询的行为是不同的。

  * 之前的 planner：根据 Table API&SQL 查询的输入是流还是批，然后先优化执行计划，接着对应转换成 DataStream 和 DataSet 程序，当 Table.insertInto() 和 TableEnvironment.sqlUpdate() 方法被调用、Table 转换成 DataStream 或 DataSet 时就会开始将 Table API 和 SQL 进行翻译，一旦翻译翻译完成后，也是和普通作业一样要执行 execute 方法后才开始运行。
  * Blink planner：不管 Table API 的输入是批还是流，都会转换成 DataStream 程序，对于 TableEnvironment 和 StreamTableEnvironment 的查询翻译是不一样的，对于 TableEnvironment，是在 TableEnvironment.execute() 调用的时候就会翻译 Table API&SQL，因为 TableEnvironment 会将多个 Sink 优化在同一个 DAG 中，而 StreamTableEnvironment 和之前的 planner 是类似的。

### 小结与反思

本节介绍了 Flink 新的 planner，然后详细地和之前的 planner 做了对比，然后对 Table API&SQL
中的概念做了介绍，还通过样例去介绍了它们的通用 API。

