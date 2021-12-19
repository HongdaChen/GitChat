上一篇中我们通过一个简单的示例，解释了装饰器模式这一代表性结构型设计模式，本文继续讨论该模式在 MyBatis 中的实际使用方式和技巧。

### 装饰者模式

#### **装饰者模式在 MyBatis 中的应用与实现**

装饰器模式在 MyBatis 主要应用在对缓存（Cache）的处理上。在 MyBatis 中，缓存的功能由根接口 Cache 定义，该接口位于
org.apache.ibatis.cache 包中，如下所示。

    
    
    public interface Cache {  
      String getId();
      void putObject(Object key, Object value);
      Object getObject(Object key);
      Object removeObject(Object key);
      void clear();
      int getSize();
      default ReadWriteLock getReadWriteLock() {
        return null;
      }
    }
    

围绕 Cache 接口的类层结构如下图所示。在该图中，Cache 接口代表一种抽象，而位于图中央的 PerpetualCache
代表该接口的具体实现类（位于 org.apache.ibatis.cache.impl 包中），而其他所有以 Cache 结尾的类都是装饰器类（位于
org.apache.ibatis.cache.decorators 包中）。

![26.01](https://images.gitbook.cn/2020-05-25-052629.png)

在上图中，整个缓存体系采用装饰器设计模式，数据存储和缓存的基本功能由 PerpetualCache
类实现，该类采用永久的（Perpetual）缓存实现，实际上采用的就是一种基于 Map 的简单策略。然后通过一系列的装饰器来对 PerpetualCache
永久缓存进行缓存策略等方面的控制。用于装饰 PerpetualCache 的标准装饰器包括
BlockingCache、FifoCache、LoggingCache、LruCache、ScheduledCache、SerializedCache、SoftCache、SynchronizedCache、TransactionalCache、WeakCache
等共计 8 个，通过名称可以判断出装饰类所要装饰的功能。

本专栏无意对所有这些装饰类做全面，我们只挑选其中一个来说明装饰器模式的应用方式，这里我们选择 FifoCache，该缓存类提供了 FIFO（First
Input First Output，先进先出）的缓存数据管理策略，代码如下所示：

    
    
    public class FifoCache implements Cache {
    
      private final Cache delegate;
      private final Deque<Object> keyList;
      private int size;
    
      public FifoCache(Cache delegate) {
        this.delegate = delegate;
        this.keyList = new LinkedList<>();
        this.size = 1024;
      }
    
      @Override
      public String getId() {
        return delegate.getId();
      }
    
      @Override
      public int getSize() {
        return delegate.getSize();
      }
    
      public void setSize(int size) {
        this.size = size;
      }
    
      @Override
      public void putObject(Object key, Object value) {
        cycleKeyList(key);
        delegate.putObject(key, value);
      }
    
      @Override
      public Object getObject(Object key) {
        return delegate.getObject(key);
      }
    
      @Override
      public Object removeObject(Object key) {
        return delegate.removeObject(key);
      }
    
      @Override
      public void clear() {
        delegate.clear();
        keyList.clear();
      }
    
      private void cycleKeyList(Object key) {
        keyList.addLast(key);
        if (keyList.size() > size) {
          Object oldestKey = keyList.removeFirst();
          delegate.removeObject(oldestKey);
        }
      }
    }
    

以上代码虽然比较冗长，但却简单明了。关键点在于我们引用了 Cache
接口，并在具体对缓存的各个操作中调用了该接口中的缓存管理方法。因为，这里实现的是一个先进先出的策略，所有，我们通过使用一个 Deque
对象来达到这种效果，这也让我们间接掌握了实现 FIFO 机制的一种实现方案。

当我们像使用这个各种缓存类时，可以通过类似 `Cache cache = new XXXCache(new
PerpetualCache("cacheid"))` 的方式实现装饰。这里把 XXXCache 替换成 FifoCache 代表着这个新创建的 cache
对象具备了 FIFO 功能。其他缓存装饰器类的使用方法也是一样。

关于 MyBatis
中缓存的设计和实现还有很多值得深入研究的地方（如一级缓存和二级缓存的区别以及各自的实现方式），本专栏后续在《剖析通用机制与组件》会有专题来进行展开。

### 组合模式

组合（Composite）模式想要表示的是一种对象的部分—整体层次结构。在 MyBatis 中，动态 SQL 机制的实现过程中应用到了组合模式。

#### **组合模式简介**

组合模式将对象组合成树形结构以表示“部分—整体”的层次结构，组合模式使得用户对单个对象和组合对象的使用具有一致性，其基本的组成结构如下所示：

![26.03](https://images.gitbook.cn/2020-05-25-052630.png)

上图中的 Component 是组合中的对象声明接口，Leaf 在组合中表示叶子结点对象，叶子结点没有子结点。而 Composite
定义有子节点行为，用来存储子 Component，并在在 Component 接口中实现与子 Component 相关的新增和删除等相关操作。

#### **组合模式在 MyBatis 中的应用与实现**

MyBatis 的一个强大特性是它的动态 SQL。相信使用过 MyBatis 的同学都知道我们可以使用它所提供的各种标签对 SQL
语句进行动态组合，这些标签常见的包含 if、choose、when、otherwise、trim、where、set、foreach 等。虽然并不建议在
SQL 级别添加过于复杂的逻辑判断，但开发人员通过这些标签确实可以组合成非常灵活的 SQL 语句，从而提高效率。

一个采用 MyBatis 动态 SQL 的示例如下所示，可以看到这里使用了 <where> 和 <if> 这两个标签，同时在 <if> 标签中使用 test
断言进行非空校验：

    
    
    <select id="findActiveBlogLike"  resultType="Blog">
      SELECT * FROM BLOG 
      <where> 
        <if test="state != null">
             state = #{state}
        </if> 
        <if test="title != null">
            AND title like #{title}
        </if>
        <if test="author != null and author.name != null">
            AND author_name like #{author.name}
        </if>
      </where>
    </select>
    

我们可以想象一下，如何实现对这段 SQL
的解析，显然，解析过程并不简单。如果对每个标签我们都进行硬编码处理，那边处理流程和代码逻辑会很混乱且不易维护。MyBatis 在处理动态 SQL
节点时，就应用到了组合设计模式。MyBatis 会将映射配置文件中定义的动态 SQL 节点、文本节点等解析成对应的一个被称为 SqlNode
的实现，并形成树形结构。

SqlNode 接口就是组合模式中的抽象组件，定义如下：

    
    
    public interface SqlNode {
      boolean apply(DynamicContext context);
    }
    

apply 是 SqlNode 接口中定义的唯一方法，该方法会根据用户传入的实参，参数解析该 SQLNode 所记录的动态 SQL 节点，当 SQL
节点下所有的 SqlNode 完成解析后，我们就可以从 DynamicContext 中获取一条动态生产的、完整的 SQL 语句。这里，我们需要先了解
DynamicContext 类的作用，该类主要用于记录解析动态 SQL 语句之后产生的 SQL
语句片段。从命名上看这是一个上下文组件，可以认为它是一个用于记录动态 SQL 语句解析结果的容器。DynamicContext 类内部具备一个
sqlBuilder 对象，类型为 JDK 中的 StringJoiner。当调用 DynamicContext.appendSql()
方法时，会将解析后的 SQL 片段追加到 DynamicContext.sqlBuilder 中保存。

作为抽象组件，SqlNode 拥有丰富的类层结构，如下图所示。这些类都与 SQLNode 接口一样位于
org.apache.ibatis.scripting.xmltags 包中。

![26.04](https://images.gitbook.cn/2020-05-25-52631.png)

上图中，我们先来看一下 MixedSqlNode 类的代码，如下所示：

    
    
    public class MixedSqlNode implements SqlNode {
      private final List<SqlNode> contents;
    
      public MixedSqlNode(List<SqlNode> contents) {
        this.contents = contents;
      }
    
      @Override
      public boolean apply(DynamicContext context) {
        contents.forEach(node -> node.apply(context));
        return true;
      }
    }
    

MixedSqlNode 维护了一个 `List<SqlNode>` 类型的列表，用于存储 SqlNode 对象，apply 方法通过 for 循环遍历
SqlNode 列表并调用其中对象的 apply 方法。很明显，MixedSqlNode 扮演了组合模式中容器组件的角色。

然后我们再来找一个典型的 SqlNode 实现，这里选择 IfSqlNode，代码如下：

    
    
    public class IfSqlNode implements SqlNode {
      private final ExpressionEvaluator evaluator;
      private final String test;
      private final SqlNode contents;
    
      public IfSqlNode(SqlNode contents, String test) {
        this.test = test;
        this.contents = contents;
        this.evaluator = new ExpressionEvaluator();
      }
    
      @Override
      public boolean apply(DynamicContext context) {
        if (evaluator.evaluateBoolean(test, context.getBindings())) {
          contents.apply(context);
          return true;
        }
        return false;
      }
    }
    

IfSqlNode 对应的是动态 SQL 节点中的 <if> 标签，其 apply 方法首先通过
ExpressionEvaluator.evaluateBoolean() 方法检测其 test 表达式是否为 true，然后根据 test
表达式的结果，决定是否执行其子节点的 apply() 方法。

找到 ExpressionEvaluator 类的 evaluateBoolean 方法，发现解析表达式使用的是 OGNL（Object Graph
Navigation Language，对象导航图语言），代码如下所示：

    
    
    public class ExpressionEvaluator {
    
      public boolean evaluateBoolean(String expression, Object parameterObject) {
        Object value = OgnlCache.getValue(expression, parameterObject);
        if (value instanceof Boolean) {
          return (Boolean) value;
        }    
        if (value instanceof Number) {
          return new BigDecimal(String.valueOf(value)).compareTo(BigDecimal.ZERO) != 0;
        }
        return value != null;
    }
    

OGNL 是应用于 Java 中的一个开源表达式语言。通过 `OgnlCache.getValue(expression,
parameterObject)` 可以看到表达式的值是从缓存中获取的，MyBatis
这样做也是为了提高性能。然后根据获取的表达式值判断其类型，然后根据不同的类型（Boolean、Number 和
String）将表达式的值与传入的值进行比较，从而实现 <if> 标签的语义。

其他 SqlNode 子类的结构与 IfSqlNode 类似，结合日常的使用方法，各自功能也比较明确。例如 TextSqlNode 表示包含 `${}`
占位符的动态 SQL 节点，其 apply 方法会使用 GenericTokenParser 解析 `${}`
占位符，并直接替换成用户给定的实际参数值；TrimSqlNode 会根据子节点的解析结果，添加或删除相应的前缀或后缀；ForeachSqlNode 对应
<foreach> 标签，对集合进行迭代等。

最后我们来看一下 DynamicSqlSource 类，在组合模式中，该类扮演了客户端的角色，代码如下：

    
    
    public class DynamicSqlSource implements SqlSource {
    
      private final Configuration configuration;
      private final SqlNode rootSqlNode;
    
      public DynamicSqlSource(Configuration configuration, SqlNode rootSqlNode) {
        this.configuration = configuration;
        this.rootSqlNode = rootSqlNode;
      }
    
      @Override
      public BoundSql getBoundSql(Object parameterObject) {
        DynamicContext context = new DynamicContext(configuration, parameterObject);
        rootSqlNode.apply(context);
        SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
        Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
        SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
        BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
        context.getBindings().forEach(boundSql::setAdditionalParameter);
        return boundSql;
      }
    }
    

可以看到，DynamicSqlSource 依赖于 SqlNode，通过构建 DynamicContext 并应用 SqlNode 的 apply
方法完成动态 SQL 的解析过程。

综上，SqlNode 接口有多个实现类，每个实现类对应一个动态 SQL 节点，其中 SqlNode 扮演抽象组件的角色，MixedSqlNode
扮演容器构件角色，而其它 SqlNode 的子类则是叶子组件角色，最后 DynamicSqlSource
则是整个模式的客户端组件。整个组合模式的类结构如下所示：

![26.05](https://images.gitbook.cn/2020-05-25-052632.png)

### 面试题分析

#### **1\. MyBatis 中缓存相关的类层结构如何？使用到了哪种设计模式？**

**考点分析**

缓存的设计和实现是常见的一道 MyBatis
面试题，而该题的切入点是考查缓存的类层设计结构，也就是考查这种类层结构背后的设计模式。所以这道题的两个问题实际上是一个问题，就是看面试者对 MyBatis
中缓存设计和实现的理解程度。我认为有一定难度，因为我们在使用缓存时更多的使用的是配置，并不会涉及这部分的源代码。

**解题思路**

我们首先需要对 MyBatis 中的 Cache 接口有一定的了解，并重点关注于该接口的一个实现类 PerpetualCache，这是 MyBatis
中缓存结构的基础，也是需要记忆的部分。然后，我们明确其他所有的缓存类都是对 PerpetualCache
的一种装饰，使用到了装饰器模式。在回答时，我们可以列举一两个具体的装饰器类来说明装饰器模式的应用方式。

**本文内容与建议回答**

我们在上一篇中介绍了装饰器模式的基本结构，在本文中给出了 MyBatis 缓存相关的类层结构以及 FifoCache 这一特定装饰器 Cache
类的实现细节。从内容完整性上讲，足以应对这道面试题的细节要求。另一方面，我们在后续的内容中还会针对 MyBatis
缓存做详细的展开，也可以在学习完后续内容之后再结合今天的内容做跟深入的回答。

#### **2\. MyBatis 如何使用组合模式来实现动态 SQL？**

**考点分析**

这个问题的问法是比较直接的，点出了组合模式。如果问题的问法变为“MyBatis 如何实现动态
SQL？”，那么我们也应该基于组合模式来回答这个问题。总体而言，这道题难度较大，因为很多面试者并不了解 MyBatis 中动态 SQL
的实现原理，也就更加无法明确组合模式在其中起到的作用。

**解题思路**

MyBatis 中的动态 SQL 实现涉及较多的知识点，包括对 SqlNode
接口的抽象、DynamicContext、ExpressionEvaluator 工具类以及第三方的 OGNL
表达式语言。如果无法完全记住这些知识点，我们应该尽量往这些点上去靠，确保面试官能够得到关于这些要点的反馈。至于组合模式，在 MyBatis
中的应用方式该模式的基本结构一致，并不是很复杂。

**本文内容与建议回答**

本文给出了组合模式的基础结构，并基于该模式详细介绍了 MyBatis 中实现动态 SQL 的细节。动态 SQL 的实现也是关于 MyBatis
的一个常见问题，建议大家重点进行理解和掌握。

### 日常开发技巧

本文中介绍了一种在日常开发过程中也可能遇到的场景，即类似 MyBatis 中动态 SQL
的实现过程。当一个应用场景时包含多个属于同一类型的对象、且这些对象之间存在复杂的依赖关系时，我们就可以把它抽象成类似动态 SQL
的处理方式。考虑到代码的扩展性和维护性，这种应用场景的设计和实现实际上是有难度的。组合模式为应对这种应用场景提供了一个非常好的解决方案。如果开发过程中遇到了类型场景，确保能够联想到本文中介绍的关于组合模式的相关内容。

### 小结与预告

通过两篇文章的我们结束了对 MyBatis 中的结构型设计模式的讨论，包括外观模式、装饰器模式和组合模式。在下一篇中，我们也将介绍行为型设计模式的特点以及在
MyBatis 中的应用，涉及模板方法模式、策略模式和迭代器模式这三种设计模式。

