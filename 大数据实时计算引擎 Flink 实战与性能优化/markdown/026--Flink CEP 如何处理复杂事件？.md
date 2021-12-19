6.1 节中介绍 Flink CEP 和其使用场景，本节将详细介绍 Flink CEP 的 API，教会大家如何去使用 Flink CEP。

### 准备依赖

要开发 Flink CEP 应用程序，首先你得在项目的 `pom.xml` 中添加依赖。

    
    
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-cep_${scala.binary.version}</artifactId>
        <version>${flink.version}</version>
    </dependency>
    

这个依赖有两种，一个是 Java 版本的，一个是 Scala 版本，你可以根据项目的开发语言自行选择。

### Flink CEP 应用入门

准备好依赖后，我们开始第一个 Flink CEP 应用程序，这里我们只做一个简单的数据流匹配，当匹配成功后将匹配的两条数据打印出来。首先定义实体类
Event 如下：

    
    
    public class Event {
        private Integer id;
        private String name;
    }
    

然后构造读取 Socket 数据流将数据进行转换成 Event，代码如下：

    
    
    SingleOutputStreamOperator<Event> eventDataStream = env.socketTextStream("127.0.0.1", 9200)
        .flatMap(new FlatMapFunction<String, Event>() {
            @Override
            public void flatMap(String s, Collector<Event> collector) throws Exception {
                if (StringUtil.isNotEmpty(s)) {
                    String[] split = s.split(",");
                    if (split.length == 2) {
                        collector.collect(new Event(Integer.valueOf(split[0]), split[1]));
                    }
                }
            }
        });
    

接着就是定义 CEP 中的匹配规则了，下面的规则表示第一个事件的 id 为 42，紧接着的第二个事件 id 要大于
10，满足这样的连续两个事件才会将这两条数据进行打印出来。

    
    
    Pattern<Event, ?> pattern = Pattern.<Event>begin("start").where(
            new SimpleCondition<Event>() {
                @Override
                public boolean filter(Event event) {
                    log.info("start {}", event.getId());
                    return event.getId() == 42;
                }
            }
    ).next("middle").where(
            new SimpleCondition<Event>() {
                @Override
                public boolean filter(Event event) {
                    log.info("middle {}", event.getId());
                    return event.getId() >= 10;
                }
            }
    );
    
    CEP.pattern(eventDataStream, pattern).select(new PatternSelectFunction<Event, String>() {
        @Override
        public String select(Map<String, List<Event>> p) throws Exception {
            StringBuilder builder = new StringBuilder();
            log.info("p = {}", p);
            builder.append(p.get("start").get(0).getId()).append(",").append(p.get("start").get(0).getName()).append("\n")
                    .append(p.get("middle").get(0).getId()).append(",").append(p.get("middle").get(0).getName());
            return builder.toString();
        }
    }).print();//打印结果
    

然后笔者在终端开启 Socket，输入的两条数据如下：

    
    
    42,zhisheng
    20,zhisheng
    

作业打印出来的日志如下图：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-29-072247.png)

整个作业 print 出来的结果如下图：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-29-072320.png)

好了，一个完整的 Flink CEP 应用程序如上，相信你也能大概理解上面的代码，接着来详细的讲解一下 Flink CEP 中的 Pattern API。

### Pattern API

你可以通过 Pattern API 去定义从流数据中匹配事件的 Pattern，每个复杂 Pattern 都是由多个简单的 Pattern
组成的，拿前面入门的应用来讲，它就是由 `start` 和 `middle` 两个简单的 Pattern 组成的，在其每个 Pattern
中都只是简单的处理了流数据。在处理的过程中需要标示该 Pattern 的名称，以便后续可以使用该名称来获取匹配到的数据，如
`p.get("start").get(0)` 它就可以获取到 Pattern 中匹配的第一个事件。接下来我们先来看下简单的 Pattern 。

#### 单个 Pattern

##### 数量

单个 Pattern 后追加的 Pattern 如果都是相同的，那如果要都重新再写一遍，换做任何人都会比较痛苦，所以就提供了 times(n)
来表示期望出现的次数，该 times() 方法还有很多写法，如下所示：

    
    
     //期望符合的事件出现 4 次
     start.times(4);
    
     //期望符合的事件不出现或者出现 4 次
     start.times(4).optional();
    
      //期望符合的事件出现 2 次或者 3 次或者 4 次
     start.times(2, 4);
    
     //期望出现 2 次、3 次或 4 次，并尽可能多地重复
     start.times(2, 4).greedy();
    
    //期望出现 2 次、3 次、4 次或者不出现
     start.times(2, 4).optional();
    
     //期望出现 0、2、3 或 4 次并尽可能多地重复
     start.times(2, 4).optional().greedy();
    
     //期望出现一个或多个事件
     start.oneOrMore();
    
     //期望出现一个或多个事件，并尽可能多地重复这些事件
     start.oneOrMore().greedy();
    
     //期望出现一个或多个事件或者不出现
     start.oneOrMore().optional();
    
     //期望出现更多次，并尽可能多地重复或者不出现
     start.oneOrMore().optional().greedy();
    
     //期望出现两个或多个事件
     start.timesOrMore(2);
    
     //期望出现 2 次或 2 次以上，并尽可能多地重复
     start.timesOrMore(2).greedy();
    
     //期望出现 2 次或更多的事件，并尽可能多地重复或者不出现
     start.timesOrMore(2).optional().greedy();
    

##### 条件

可以通过 `pattern.where()`、`pattern.or()` 或 `pattern.until()` 方法指定事件属性的条件。条件可以是
`IterativeConditions` 或`SimpleConditions`。比如 SimpleCondition 可以像下面这样使用：

    
    
    start.where(new SimpleCondition<Event>() {
        @Override
        public boolean filter(Event value) {
            return "zhisheng".equals(value.getName());
        }
    });
    

#### 组合 Pattern

前面已经对单个 Pattern 做了详细对讲解，接下来讲解如何将多个 Pattern 进行组合来完成一些需求。在完成组合 Pattern 之前需要定义第一个
Pattern，然后在第一个的基础上继续添加新的 Pattern。比如定义了第一个 Pattern 如下：

    
    
    Pattern<Event, ?> start = Pattern.<Event>begin("start");
    

接下来，可以为此指定更多的 Pattern，通过指定的不同的连接条件。比如：

  * next()：要求比较严格，该事件一定要紧跟着前一个事件。

  * followedBy()：该事件在前一个事件后面就行，两个事件之间可能会有其他的事件。

  * followedByAny()：该事件在前一个事件后面的就满足条件，两个事件之间可能会有其他的事件，返回值比上一个多。

  * notNext()：不希望前一个事件后面紧跟着该事件出现。

  * notFollowedBy()：不希望后面出现该事件。

具体怎么写呢，可以看下样例：

    
    
    Pattern<Event, ?> strict = start.next("middle").where(...);
    
    Pattern<Event, ?> relaxed = start.followedBy("middle").where(...);
    
    Pattern<Event, ?> nonDetermin = start.followedByAny("middle").where(...);
    
    Pattern<Event, ?> strictNot = start.notNext("not").where(...);
    
    Pattern<Event, ?> relaxedNot = start.notFollowedBy("not").where(...);
    

可能概念讲了很多，但是还是不太清楚，这里举个例子说明一下，假设有个 Pattern 是 `a b`，给定的数据输入顺序是 `a c b
b`，对于上面那种不同的连接条件可能最后返回的值不一样。

  1. a 和 b 之间使用 next() 连接，那么则返回 {}，即没有匹配到数据
  2. a 和 b 之间使用 followedBy() 连接，那么则返回 {a, b}
  3. a 和 b 之间使用 followedByAny() 连接，那么则返回 {a, b}, {a, b}

相信通过上面的这个例子讲解你就知道了它们的区别，尤其是 followedBy() 和
followedByAny()，笔者一开始也是毕竟懵，后面也是通过代码测试才搞明白它们之间的区别的。除此之外，还可以为 Pattern
定义时间约束。例如，可以通过 `pattern.within(Time.seconds(10))` 方法定义此 Pattern 应该 10 秒内完成匹配。
该时间不仅支持处理时间还支持事件时间。另外还可以与 consecutive()、allowCombinations() 等组合，更多的请看下图中
Pattern 类的方法。

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-29-164118.png)

#### Group Pattern

业务需求比较复杂的场景，如果要使用 Pattern 来定义的话，可能这个 Pattern 会很长并且还会嵌套，比如由
begin、followedBy、followedByAny、next 组成和嵌套，另外还可以再和
oneOrMore()、times(#ofTimes)、times(#fromTimes,
#toTimes)、optional()、consecutive()、allowCombinations() 等结合使用。效果如下面这种：

    
    
    Pattern<Event, ?> start = Pattern.begin(
        Pattern.<Event>begin("start").where(...).followedBy("start_middle").where(...)
    );
    
    //next 表示连续
    Pattern<Event, ?> strict = start.next(
        Pattern.<Event>begin("next_start").where(...).followedBy("next_middle").where(...)
    ).times(3);
    
    //followedBy 代表在后面就行
    Pattern<Event, ?> relaxed = start.followedBy(
        Pattern.<Event>begin("followedby_start").where(...).followedBy("followedby_middle").where(...)
    ).oneOrMore();
    
    //followedByAny
    Pattern<Event, ?> nonDetermin = start.followedByAny(
        Pattern.<Event>begin("followedbyany_start").where(...).followedBy("followedbyany_middle").where(...)
    ).optional();
    

关于上面这些 Pattern 操作的更详细的解释可以查看[官网](https://ci.apache.org/projects/flink/flink-
docs-release-1.9/dev/libs/cep.html#groups-of-patterns)。

#### 事件匹配跳过策略

对于给定组合的复杂 Pattern，有的事件可能会匹配到多个 Pattern，如果要控制将事件的匹配数，需要指定跳过策略。在 Flink CEP
中跳过策略有四种类型，如下所示：

  * NO_SKIP：不跳过，将发出所有可能的匹配事件。

  * SKIP_TO_FIRST：丢弃包含 PatternName 第一个之前匹配事件的每个部分匹配。

  * SKIP_TO_LAST：丢弃包含 PatternName 最后一个匹配事件之前的每个部分匹配。

  * SKIP_PAST_LAST_EVENT：丢弃包含匹配事件的每个部分匹配。

  * SKIP_TO_NEXT：丢弃以同一事件开始的所有部分匹配。

这几种策略都是根据 AfterMatchSkipStrategy 来实现的，可以看下它们的类结构图，如下所示：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-29-133737.png)

关于这几种跳过策略的具体区别可以查看[官网](https://ci.apache.org/projects/flink/flink-docs-
release-1.9/dev/libs/cep.html#after-match-skip-strategy)，至于如何使用跳过策略，其实
AfterMatchSkipStrategy 抽象类中已经提供了 5 种静态方法可以直接使用，方法如下：

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-29-135526.png)

使用方法如下：

    
    
    AfterMatchSkipStrategy skipStrategy = ...; // 使用 AfterMatchSkipStrategy 调用不同的静态方法
    Pattern.begin("start", skipStrategy);
    

### 检测 Pattern

编写好了 Pattern 之后，你需要的是将其应用在流数据中去做匹配。这时要做的就是构造一个 PatternStream，它可以通过
`CEP.pattern(eventDataStream, pattern)` 来获取一个 PatternStream 对象，在
`CEP.pattern()` 方法中，你可以选择传入两个参数（DataStream 和 Pattern），也可以选择传入三个参数
（DataStream、Pattern 和 EventComparator），因为 CEP 类中它有两个不同参数数量的 pattern 方法。

    
    
    public class CEP {
    
        public static <T> PatternStream<T> pattern(DataStream<T> input, Pattern<T, ?> pattern) {
            return new PatternStream(input, pattern);
        }
    
        public static <T> PatternStream<T> pattern(DataStream<T> input, Pattern<T, ?> pattern, EventComparator<T> comparator) {
            PatternStream<T> stream = new PatternStream(input, pattern);
            return stream.withComparator(comparator);
        }
    }
    

#### 选择 Pattern

在获取到 PatternStream 后，你可以通过 select 或 flatSelect 方法从匹配到的事件流中查询。如果使用的是 select
方法，则需要实现传入一个 PatternSelectFunction 的实现作为参数，PatternSelectFunction 具有为每个匹配事件调用的
select 方法，该方法的参数是 `Map<String, List<Event>>`，这个 Map 的 key 是 Pattern
的名字，在前面入门案例中设置的 `start` 和 `middle` 在这时就起作用了，你可以通过类似 `get("start")` 方法的形式来获取匹配到
`start` 的所有事件。如果使用的是 flatSelect 方法，则需要实现传入一个 PatternFlatSelectFunction
的实现作为参数，这个和 PatternSelectFunction 不一致地方在于它可以返回多个结果，因为这个接口中的 flatSelect 方法含有一个
Collector，它可以返回多个数据到下游去。两者的样例如下：

    
    
    CEP.pattern(eventDataStream, pattern).select(new PatternSelectFunction<Event, String>() {
        @Override
        public String select(Map<String, List<Event>> p) throws Exception {
            StringBuilder builder = new StringBuilder();
            builder.append(p.get("start").get(0).getId()).append(",").append(p.get("start").get(0).getName()).append("\n")
                    .append(p.get("middle").get(0).getId()).append(",").append(p.get("middle").get(0).getName());
            return builder.toString();
        }
    }).print();
    
    CEP.pattern(eventDataStream, pattern).flatSelect(new PatternFlatSelectFunction<Event, String>() {
        @Override
        public void flatSelect(Map<String, List<Event>> map, Collector<String> collector) throws Exception {
            for (Map.Entry<String, List<Event>> entry : map.entrySet()) {
                collector.collect(entry.getKey() + " " + entry.getValue().get(0).getId() + "," + entry.getValue().get(0).getName());
            }
        }
    }).print();
    

关于 PatternStream 中的 select 或 flatSelect 方法其实可以传入不同的参数，比如传入 OutputTag 和
PatternTimeoutFunction 去处理延迟的数据，具体查看下图。

![](http://zhisheng-blog.oss-cn-
hangzhou.aliyuncs.com/img/2019-10-29-125416.png)

如果使用的 Flink CEP 版本是大于等于 1.8 的话，还可以使用 process 方法，在上图中也可以看到在 PatternStream
类中包含了该方法。要使用 process 的话，得传入一个 PatternProcessFunction 的实现作为参数，在该实现中需要重写
processMatch 方法。使用 PatternProcessFunction 比使用 PatternSelectFunction 和
PatternFlatSelectFunction 更好的是，它支持获取应用的的上下文，那么也就意味着它可以访问时间（因为 Context 接口继承自
TimeContext 接口）。另外如果要处理延迟的数据可以与 TimedOutPartialMatchHandler 接口的实现类一起使用。

### CEP 时间属性

#### 根据事件时间处理延迟数据

在 CEP
中，元素处理的顺序很重要，当时间策略设置为事件时间时，为了确保能够按照事件时间的顺序来处理元素，先来的事件会暂存在缓冲区域中，然后对缓冲区域中的这些事件按照事件时间进行排序，当水印到达时，比水印时间小的事件会按照顺序依次处理的。这意味着水印之间的元素是按照事件时间顺序处理的。

注意：当作业设置的时间属性是事件时间是，CEP
中会认为收到的水印时间是正确的，会严格按照水印的时间来处理元素，从而保证能顺序的处理元素。另外对于这种延迟的数据（和 3.5 节中的延迟数据类似），CEP
中也是支持通过 side output 设置 OutputTag 标签来将其收集。使用方式如下：

    
    
    PatternStream<Event> patternStream = CEP.pattern(inputDataStream, pattern);
    
    OutputTag<String> lateDataOutputTag = new OutputTag<String>("late-data"){};
    
    SingleOutputStreamOperator<ComplexEvent> result = patternStream
        .sideOutputLateData(lateDataOutputTag)
        .select(
            new PatternSelectFunction<Event, ComplexEvent>() {...}
        );
    
    DataStream<String> lateData = result.getSideOutput(lateDataOutputTag);
    

#### 时间上下文

在 PatternProcessFunction 和 IterativeCondition 中可以通过 TimeContext
访问当前正在处理的事件的时间（Event Time）和此时机器上的时间（Processing Time）。你可以查看到这两个类中都包含了
Context，而这个 Context 继承自 TimeContext，在 TimeContext 接口中定义了获取事件时间和处理时间的方法。

    
    
    public interface TimeContext {
    
        long timestamp();
    
        long currentProcessingTime();
    }
    

### 小结与反思

本节开始通过一个 Flink CEP 案例教大家上手，后面通过讲解 Flink CEP 的 Pattern
API，更多详细的还是得去看官网文档，其实也建议大家好好的跟着官网的文档过一遍所有的
API，并跟着敲一些样例来实现，这样在开发需求的时候才能够及时的想到什么场景下该使用哪种 API，接着教了大家如何将 Pattern
与数据流结合起来匹配并获取匹配的数据，最后讲了下 CEP 中的时间概念。

你公司有使用 Flink CEP 吗？通常使用哪些 API 居多？

本节涉及代码地址：https://github.com/zhisheng17/flink-learning/tree/master/flink-
learning-libraries/flink-learning-libraries-cep

