### State Processor API 介绍

能够从外部访问 Flink 作业的状态一直用户迫切需要的功能之一，在 Apache Flink 1.9.0 中新引入了 State Processor
API，该 API 让用户可以通过 Flink DataSet 作业来灵活读取、写入和修改 Flink 的 Savepoint 和 Checkpoint。

### 在 Flink 1.9 之前是如何处理状态的？

一般来说，大多数的 Flink
作业都是有状态的，并且随着作业运行的时间越来越久，就会累积越多越多的状态，如果因为故障导致作业崩溃可能会导致作业的状态都丢失，那么对于比较重要的状态来说，损失就会很大。为了保证作业状态的一致性和持久性，Flink
从一开始使用的就是 Checkpoint 和 Savepoint 来保存状态，并且可以从 Savepoint 中恢复状态。在 Flink 的每个新
Release 版本中，Flink 社区添加了越来越多与状态相关的功能以提高 Checkpoint 的速度和恢复速度。

有的时候，用户可能会有这些需求场景，比如从第三方外部系统访问作业的状态、将作业的状态信息迁移到另一个应用程序等，目前现有支持查询作业状态的功能
Queryable State，但是在 Flink 中目前该功能只支持根据 Key
查找，并且不能保证返回值的一致性。另外该功能不支持添加和修改作业的状态，所以适用的场景还是比较有限。

### 使用 State Processor API 读写作业状态

在 1.9 版本中的 State Processor API，它完全和之前不一致，该功能使用 InputFormat 和 OutputFormat 扩展了
DataSet API 以读取和写入 Checkpoint 和 Savepoint 数据。由于 DataSet 和 Table API
的互通性，所以也可以使用 Table 或者 SQL API 查询和分析状态的数据。例如，再获取到正在运行的流作业状态的 Checkpoint 后，可以使用
DataSet 批处理程序对其进行分析，以验证该流作业的运行是否正确。另外 State Processor API
还可以修复不一致的状态信息，它提供了很多方法来开发有状态的应用程序，这些方法在以前的版本中因为设计的问题导致作业在启动后不能再修改，否则状态可能会丢失。现在，你可以任意修改状态的数据类型、调整算子的最大并行度、拆分或合并算子的状态、重新分配算子的
uid 等。

### 使用 DataSet 读取作业状态

State Processor API 将作业的状态映射到一个或多个可以单独处理的数据集，为了能够使用该
API，需要先了解这个映射的工作方式，首先来看下有状态的 Flink 作业是什么样子的。Flink
作业是由很多算子组成，通常是一个或多个数据源（Source）、一些实际处理数据的算子（比如 Map／Filter／FlatMap 等）和一个或者多个
Sink。每个算子会在一个或者多个任务中并行运行（取决于并行度），并且可以使用不同类型的状态，算子可能会有零个、一个或多个 Operator
State，这些状态会组成一个以算子任务为范围的列表。如果是算子应用在 KeyedStream，它还有零个、一个或者多个 Keyed
State，它们的作用域范围是从每个已处理数据中提取 Key，可以将 Keyed State 看作是一个分布式的 Map。

State Processor API 现在提供了读取、新增和修改 Savepoint 数据的方法，比如从已加载的 Savepoint
中读取数据集，然后将数据集转换为状态并将其保存到 Savepoint。下面分别讲解下这三种方法该如何使用。

#### 读取现有的 Savepoint

读取状态首先需要指定一个 Savepoint（或者 Checkpoint） 的路径和状态后端存储的类型。

    
    
    ExecutionEnvironment bEnv   = ExecutionEnvironment.getExecutionEnvironment();
    ExistingSavepoint savepoint = Savepoint.load(bEnv, "hdfs://path/", new RocksDBStateBackend());
    

读取 Operator State 时，只需指定算子的 uid、状态名称和类型信息。

    
    
    DataSet<Integer> listState  = savepoint.readListState("zhisheng-uid", "list-state", Types.INT);
    
    DataSet<Integer> unionState = savepoint.readUnionState("zhisheng-uid", "union-state", Types.INT);
    
    DataSet<Tuple2<Integer, Integer>> broadcastState = savepoint.readBroadcastState("zhisheng-uid", "broadcast-state", Types.INT, Types.INT);
    

如果在状态描述符（StateDescriptor）中使用了自定义类型序列化器 TypeSerializer，也可以指定它：

    
    
    DataSet<Integer> listState = savepoint.readListState(
        "zhisheng-uid", "list-state", 
        Types.INT, new MyCustomIntSerializer());
    

当读取 Keyed State 时，用户可以指定 KeyedStateReaderFunction 来读取任意列和复杂的状态类型，例如
ListState，MapState 和 AggregatingState。这意味着如果算子包含了有状态的处理函数，例如：

    
    
    public class StatefulFunctionWithTime extends KeyedProcessFunction<Integer, Integer, Void> {
    
       ValueState<Integer> state;
    
       @Override
       public void open(Configuration parameters) {
          ValueStateDescriptor<Integer> stateDescriptor = new ValueStateDescriptor<>("state", Types.INT);
          state = getRuntimeContext().getState(stateDescriptor);
       }
    
       @Override
       public void processElement(Integer value, Context ctx, Collector<Void> out) throws Exception {
          state.update(value + 1);
       }
    }
    

然后可以通过定义输出类型和相应的 KeyedStateReaderFunction 进行读取上面的状态。

    
    
    class KeyedState {
      Integer key;
      Integer value;
    }
    
    class ReaderFunction extends KeyedStateReaderFunction<Integer, KeyedState> {
      ValueState<Integer> state;
    
      @Override
      public void open(Configuration parameters) {
         ValueStateDescriptor<Integer> stateDescriptor = new ValueStateDescriptor<>("state", Types.INT);
         state = getRuntimeContext().getState(stateDescriptor);
      }
    
      @Override
      public void readKey(Integer key, Context ctx, Collector<KeyedState> out) throws Exception {
         KeyedState data = new KeyedState();
         data.key    = key;
         data.value  = state.value();
         out.collect(data);
      }
    }
    
    DataSet<KeyedState> keyedState = savepoint.readKeyedState("zhisheng-uid", new ReaderFunction());
    

注意：使用 KeyedStateReaderFunction 时，状态描述器（StateDescriptor）必须在 open 方法中注册，否则
RuntimeContext#getState，RuntimeContext#getListState 或
RuntimeContext#getMapState 将导致 RuntimeException。

#### 写入新的 Savepoint

写入新的 Savepoint 主要是基于下面三个接口：

  * StateBootstrapFunction：用于写入未分区的 Operator State
  * BroadcastStateBootstrapFunction：用于写入 Broadcast State
  * KeyedStateBootstrapFunction：用于写入 Keyed State

    
    
    public  class Account {
        public int id;
    
        public double amount;    
    
        public long timestamp;
    }
    
    public class AccountBootstrapper extends KeyedStateBootstrapFunction<Integer, Account> {
        ValueState<Double> state;
    
        @Override
        public void open(Configuration parameters) {
            ValueStateDescriptor<Double> descriptor = new ValueStateDescriptor<>("total",Types.DOUBLE);
            state = getRuntimeContext().getState(descriptor);
        }
    
        @Override
        public void processElement(Account value, Context ctx) throws Exception {
            state.update(value.amount);
        }
    }
    
    ExecutionEnvironment bEnv = ExecutionEnvironment.getExecutionEnvironment();
    
    DataSet<Account> accountDataSet = bEnv.fromCollection(accounts);
    
    BootstrapTransformation<Account> transformation = OperatorTransformation
        .bootstrapWith(accountDataSet)
        .keyBy(acc -> acc.id)
        .transform(new AccountBootstrapper());
    

该 KeyedStateBootstrapFunction 函数支持设置事件时间和处理时间的定时器，定时器不会在该函数中触发，只有在 DataStream
作业中还原后才会激活，如果设置了处理时间的定时器，但是该处理时间已经过期了，那么在恢复作业的时候会立即触发。一旦创建了一个或者多个算子，可以将它们合并为一个
Savepoint。

    
    
    Savepoint
        .create(backend, 128)
        .withOperator("uid1", transformation1)
        .withOperator("uid2", transformation2)
        .write(savepointPath);
    

#### 修改现有的 Savepoint

除了可以从头开始创建 Savepoint 之外，还可以基于现有的 Savepoint，例如在为现有作业添加新的算子。

    
    
    Savepoint
        .load(backend, oldPath)
        .withOperator("uid", transformation)
        .write(newPath);
    

删除或者覆盖现有 Savepoint 中的算子状态，并将其写入。

    
    
    Savepoint
        .removeOperator(oldOperatorUid)
        .withOperator(oldOperatorUid, transformation)
        .write(path)
    

### 为什么要使用 DataSet API？

社区一直在想将批和流统一，所以在未来 DataSet API 可能会废弃，那么为啥 State Processor API 还要基于 DataSet API
开发呢？这是因为社区在设计这个功能的时候，对 DataStream API 和 Table API 做了评估对比，但没有一个能满足需求的，而又因为
State Processor API 功能对于 Flink API 的进一步发展有至关重要的作用，因此社区决定在 DataSet API 构建 State
Processor API 功能，但是尽可能的降低了对 DataSet API 的依赖性，方便后续迁移到其他的 API 中。

### 小结与反思

本节讲了 Flink 1.9 中的 State Processor API 的概念和如何使用，以及该功能的设计背景及需求。有关更多详细信息，请参见
[FLIP-43](https://cwiki.apache.org/confluence/display/FLINK/FLIP-43%3A+State+Processor+API)。对于使用
DataSet API 来完成该功能，你有什么更好的解决方案吗？

