这是设计模式系列的最后一篇，我们关注于行为型模式的基本结构及其在 MyBatis 中的应用。

### 模板方法模式

模板方法模式非常简单，其实就是类的继承机制，但它却是一个应用非常广泛的模式。该模式在实现上往往与抽象类一起使用，在 MyBatis
中大量应用了模板方法来实现基类和子类之间的职责分离和协作。

#### **模板方法模式简介**

当我们要完成一个过程或一系列步骤时，这些过程或步骤在某一细节层次保持一致，但其个别步骤在更详细的层次上的实现可能不同时，可以考虑用模板方法模式来处理。模板方法模式的结构示意图如下所示：

![27.01](https://images.gitbook.cn/2020-05-25-052650.png)

抽象模板类 AbstractClass 定义了一套算法框架或流程，而具体实现类 ConcreteClass 对算法框架或流程中的某些步骤进行了实现。

#### **模板方法模式在 MyBatis 中的应用与实现**

在 MyBatis 中，我们知道 SqlSession 的 SQL 执行实际上都是委托给 Executor 实现的，Executor
包含如下图所示中的类层结构，其中的 BaseExecutor 就采用了模板方法模式，它实现了大部分的 SQL
执行逻辑，然后把以下几个方法交给子类定制化完成。这几个子类的具体实现使用了不同的策略，体现了模板方法模式基于继承的代码复用技术。

![27.02](https://images.gitbook.cn/2020-05-25-052651.png)

Executor 是 MyBatis 的核心接口之一，其中定义了数据库操作的基本方法，该接口的代码如下：

    
    
    public interface Executor {
    
      ResultHandler NO_RESULT_HANDLER = null;
    
      //执行 update、insert、delete 三种类型的 SQL 语句
      int update(MappedStatement ms, Object parameter) throws SQLException;
    
      //执行 selete 类型的 SQL 语句，返回值分为结果对象列表或游标对象
      <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;
      <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;
      <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;
    
      //批量执行 SQL 语句
      List<BatchResult> flushStatements() throws SQLException;
    
      //提交事务
      void commit(boolean required) throws SQLException;
    
      //回滚事务
      void rollback(boolean required) throws SQLException;
    
      //创建缓存中用到的 CacheKey 对象
      CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql);
    
      //根据 CacheKey 对象查找缓存
      boolean isCached(MappedStatement ms, CacheKey key);
    
      //清空一级缓存
      void clearLocalCache();
    
      //延迟加载一级缓存中的数据
      void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType);
    
      //获取事务对象
      Transaction getTransaction();
    
      //关闭 Executor 对象
      void close(boolean forceRollback);
    
      //检测 Executor 是否已关闭
      boolean isClosed();
    
      void setExecutorWrapper(Executor executor);
    }
    

作为模板方法模式的具体体现，BaseExecutor
是一个抽象类，主要提供了缓存管理和事务管理的基本功能，核心代码如下所示（为了只关注于模板方法相关内容，对代码的结构做了一定调整）：

    
    
    public abstract class BaseExecutor implements Executor {
    
        //省略参数和构造函数 
    
        //抽象方法定义 
      protected abstract int doUpdate(MappedStatement ms, Object parameter)
          throws SQLException;
    
      protected abstract List<BatchResult> doFlushStatements(boolean isRollback)
          throws SQLException;
    
      protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
          throws SQLException;
    
      protected abstract <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql)
          throws SQLException;
    
    
        //使用抽象方法的方法定义
      @Override
      public int update(MappedStatement ms, Object parameter) throws SQLException {
        ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
        if (closed) {
          throw new ExecutorException("Executor was closed.");
        }
        clearLocalCache();
        return doUpdate(ms, parameter);
      }
    
      public List<BatchResult> flushStatements(boolean isRollBack) throws SQLException {
        if (closed) {
          throw new ExecutorException("Executor was closed.");
        }
        return doFlushStatements(isRollBack);
      }
    
      @Override
      public <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException {
        BoundSql boundSql = ms.getBoundSql(parameter);
        return doQueryCursor(ms, parameter, rowBounds, boundSql);
      }
    
    private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        List<E> list;
        localCache.putObject(key, EXECUTION_PLACEHOLDER);
        try {
          list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
        } finally {
          localCache.removeObject(key);
        }
        localCache.putObject(key, list);
        if (ms.getStatementType() == StatementType.CALLABLE) {
          localOutputParameterCache.putObject(key, parameter);
        }
        return list;
      }
    
        // 省略其他方法
    }
    

上述代码对 BaseExecutor 类做了代码的裁剪和组织顺序上的调整，我们首先看到四个抽象方法，分别是 doUpdate() 方法、doQuery()
方法、doQueryCursor() 方法、doFlushStatement() 方法。然后，我们列举了直接使用这四个抽象方法的方法，这些方法有些直接实现了
Executor 中接口中对应的方法，而有些则作为中间方法供接口实现时使用。

我们再来看一下 BaseExecutor 中最复杂也最具代表性的 query 方法，代码如下所示：

    
    
    @Override
      public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
        if (closed) {
          throw new ExecutorException("Executor was closed.");
        }
        if (queryStack == 0 && ms.isFlushCacheRequired()) {
          clearLocalCache();
        }
        List<E> list;
        try {
          queryStack++;
          list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
          if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
          } else {
            // 这里调用了前文中提到的 queryFromDatabase 方法
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
          }
        } finally {
          queryStack--;
        }
        if (queryStack == 0) {
          for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
          }      
          deferredLoads.clear();
          if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            clearLocalCache();
          }
        }
        return list;
      }
    

可以看到，其中的 query() 方法首先会创建 CacheKey 对象，并根据 CacheKey
对象查找一级缓存，如果缓存命中则返回缓存中记录的结果对象，如果未命中则通过 queryFromDatabase
方法查询数据库得到结果集，之后将结果集映射成结果对象并保存到一级缓存中，同时返回结果对象。而在 queryFromDatabase
方法中，我们调用了供子类实现的 doQuery 方法。显然，由于这里使用了模板方法模式，一级缓存等固定不变的操作都封装到了 BaseExecutor
中，因此子类就不必再关心一级缓存等操作，只需要专注实现 doQuery 方法即可。

BaseExecutor 的子类有四个分别是
SimpleExecotor、ReuseExecutor、BatchExecutor、ClosedExecutor，这里以 SimpleExecutor
为例，介绍其 doQuery 方法的实现，代码如下：

    
    
    @Override
      public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        Statement stmt = null;
        try {
          Configuration configuration = ms.getConfiguration();
          StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
          stmt = prepareStatement(handler, ms.getStatementLog());
          return handler.query(stmt, resultHandler);
        } finally {
          closeStatement(stmt);
        }
      }
    

显然，该 doQuery 方法的流程非常简单，就是通过 StatementHandler 完成对 prepareStatement 中 SQL
语句的执行，没有任何涉及缓存等复杂操作，因为这些操作已经在模板方法类 BaseExecutor 得以实现。

### 策略模式

关于 Executor 类层结构的讨论，实际上还可以引申出另一种设计模式，即策略模式。

#### **策略模式简介**

所谓策略（Strategy），本质上是定义了一组算法，然后将每个算法都封装起来，并且使它们之间可以互换。策略模式的结构示意图如下所示：

![27.03](https://images.gitbook.cn/2020-05-25-052652.png)

上图中，Context 是一个上下文，维护一个对 Strategy 对象的引用。Strategy 是一个策略接口，用于定义所有支持算法的公共接口。而两个
ConcreteStrategy 是具体策略类，封装了具体的算法或行为。

#### **策略模式在 MyBatis 中的应用与实现**

对于 MyBatis 而言，上图中的 Context 实际上就是 DefaultSqlSession，DefaultSqlSession 内部包含了对
Executor 的引用。让我们再次回到 Executor 类层结构的讨论，我们知道 Executor 接口的实现类中，除 ClosedExecutor
只是某个类的一个内部类之外，其他三种 Executor 的简要介绍如下所示：

  * SimpleExecutor：普通执行器，是 MyBatis 执行 Mapper 语句时默认使用的 Executor，提供最基本的 Mapper 语句执行功能
  * ReuseExecutor：预处理语句重用执行器，提供了 Statement 重用的功能，对使用过的 Statement 对象进行重用，可以减少 SQL 预编译以及创建和销毁 Statement 对象的开销，从而提高性能
  * BatchExecutor：批处理执行器，实现了批处理多条 SQL 语句的功能，在客户端缓存多条 SQL 并在合适的时机将多条 SQL 打包发送给数据库执行，从而减少网络方面的开销，提升系统的性能。

在讨论模板方法模式时，我们关注的是 Executor 接口的抽象实现类 BaseExecutor，该抽象实现各个子类提供了以 doXXX
方式命名的几个模版方法，从而确保例如缓存处理等公共功能不需要各个子类重复实现。而在这里，我们关注点在于各个 Executor 子类的生成策略。

回顾我们在使用 MyBatis 时的配置文件，其中存在如下所示的一个配置项，该配置项用于设置 Executor 的默认类型。而该配置项的默认值就是
SIMPLE，也就是指向 Executor 的实现类 SimpleExecutor。

    
    
    <setting name="defaultExecutorType" value="SIMPLE" />
    

前面提到过的 MyBatis 核心类 Configuration 类似于策略模式中的
Context，根据传入的策略对象类型生成相应的策略对象，Configuration 类中的相关代码如下所示：

    
    
    protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
    
    public Executor newExecutor(Transaction transaction) {
        return newExecutor(transaction, defaultExecutorType);
    }
    
    public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
        executorType = executorType == null ? defaultExecutorType : executorType;
        executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
        Executor executor;
        if (ExecutorType.BATCH == executorType) {
          executor = new BatchExecutor(this, transaction);
        } else if (ExecutorType.REUSE == executorType) {
          executor = new ReuseExecutor(this, transaction);
        } else {
          executor = new SimpleExecutor(this, transaction);
        }
        if (cacheEnabled) {
          executor = new CachingExecutor(executor);
        }
        executor = (Executor) interceptorChain.pluginAll(executor);
        return executor;
    }
    

这段代码我们已经很熟悉了，应该在讨论管道-过滤器模式时已经分析过其中的结构。在讨论管道-过滤器模式时，我们关注的是 interceptorChain
的应用，而在本篇中，我们关注根据 ExecutorType 生成相应 executor 对象的过程。

Executor 的使用者是 SqlSessionFactory 接口的的默认实现类 DefaultSqlSessionFactory，该类用于生成
SqlSession。而在 SqlSession 生成过程中，需要指定 ExecutorType。这时就会调用 Configuration 对象的上述
newExecutor 方法。相关代码在介绍建造者模式时也已经给出，这里不再展开，读者可自行阅读 DefaultSqlSessionFactory 类。

### 迭代器模式

在日常开发过程中，我们可能很少会自己去实现一个迭代器（Iterator），但几乎每天都在使用 JDK 中自带的各种迭代器。在 MyBatis 中，针对
SQL 中配置项语句的解析，设计并实现了一套迭代器组件。

#### **迭代器模式简介**

通过设计一个迭代器，这个迭代器提供一种方法，可以顺序访问聚合对象中的各个元素，但又不暴露该对象的内部表示。迭代器模式的基本结构如下图所示：

![27.04](https://images.gitbook.cn/2020-05-25-052654.png)

上图中的 Aggregate 相当于是一个容器，致力于提供符合 Iterator 实现的数据格式。当我们访问容器时，则是使用 Iterator
提供的数据遍历方法进行数据访问，这样处理容器数据的逻辑就和容器本身的实现了解耦，因为我们只需要使用 Iterator
接口就行了，完全不用关心容器怎么实现、底层数据如何访问之类的问题。而且更换容器的时候也不需要修改数据处理逻辑。

#### **迭代器模式在 MyBatis 中的应用与实现**

MyBatis 中存在两个类，通过了对迭代器模式的具体实现，分别是 PropertyTokenizer 和 CursorIterator。我们先来看
PropertyTokenizer 的实现方法。

**1\. PropertyTokenizer**

在 MyBatis 的 org.apache.ibatis.reflection.property 包中，存在一个非常常用的工具类
PropertyTokenizer，该类主要用于解析诸如 `order[0].address.contactinfo.name`
类型的属性表达式，在这个例子中，我们可以看到系统是在处理订单实体的地址信息，MyBatis
支持使用这种形式的表达式来获取最终的“name”属性。我们可以想象一下，当我们想要解析
`order[0].address.contactinfo.name` 字符串时，我们势必需要先对其进行分段处理以分别获取各个层级的对象属性名称，如果遇到
`[]` 符号表示说明要处理的是一个对象数组。这种分层级的处理方式可以认为是一种迭代处理方式，作为迭代器模式的实现，PropertyTokenizer
对这种处理方式提供了支持，该类代码如下所示：

    
    
    public class PropertyTokenizer implements Iterator<PropertyTokenizer> {
      private String name;
      private final String indexedName;
      private String index;
      private final String children;
    
      public PropertyTokenizer(String fullname) {
        int delim = fullname.indexOf('.');
        if (delim > -1) {
          name = fullname.substring(0, delim);
          children = fullname.substring(delim + 1);
        } else {
          name = fullname;
          children = null;
        }
        indexedName = name;
        delim = name.indexOf('[');
        if (delim > -1) {
          index = name.substring(delim + 1, name.length() - 1);
          name = name.substring(0, delim);
        }
      }
    
      public String getName() {
        return name;
      }
    
      public String getIndex() {
        return index;
      }
    
      public String getIndexedName() {
        return indexedName;
      }
    
      public String getChildren() {
        return children;
      }
    
      @Override
      public boolean hasNext() {
        return children != null;
      }
    
      @Override
      public PropertyTokenizer next() {
        return new PropertyTokenizer(children);
      }
    
      @Override
      public void remove() {
        throw new UnsupportedOperationException("Remove is not supported, as it has no meaning in the context of properties.");
      }
    }
    

针对 `order[0].address.contactinfo.name` 字符串，当启动解析时，PropertyTokenizer 类的 name
字段指的就是 `order`，indexedName 字段指的就是 `order[0]`，index 字段指的就是 `0`，而 children
字段指的就是 `address.contactinfo.name`。在构造函数中，当对传入的字符串进行处理时，通过 `.`
分隔符将其分作两部分。然后在对获取的 name 字段提取 `[`，把中括号里的数字给解析出来，如果 name 段子你中包含 `[]` 的话，分别获取
index 字段并更新 name 字段。

通过构造函数对输入字符串进行处理之后，PropertyTokenizer 的 next() 方法非常简单，直接再通过 children 字段再来创建一个新的
PropertyTokenizer 实例即可。而经常使用的 hasNext() 方法实现也很简单，就是判断 children 属性是否为空。

我们再来看 PropertyTokenizer 类的使用方法，我们在 org.apache.ibatis.reflection 包的 MetaObject
类中找到了它的一种常见使用方法，代码如下所示。

    
    
    public Object getValue(String name) {
        PropertyTokenizer prop = new PropertyTokenizer(name);
        if (prop.hasNext()) {
          MetaObject metaValue = metaObjectForProperty(prop.getIndexedName());
          if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
            return null;
          } else {
            return metaValue.getValue(prop.getChildren());
          }
        } else {
          return objectWrapper.get(prop);
        }
      }
    

这里可以明显看到通过 PropertyTokenizer 的 prop.hasNext() 方法进行递归调用的代码处理流程。

**2\. CursorIterator**

其实，迭代器模式有时还被称为是游标（Cursor）模式，所以通常可以使用该模式构建一个基于游标机制的组件。我们数据库访问领域中恰恰就有一个游标的概念，当查询数据库返回大量的数据项时可以使用游标
Cursor，利用其中的迭代器可以懒加载数据，避免因为一次性加载所有数据导致内存奔溃。而 MyBatis
又是一个数据库访问框架，那么在这个框架中是否存在一个基于迭代器模式的游标组件呢？答案是肯定的，让我们来看一下。

MyBatis 提供了 Cursor 接口用于表示游标操作，该接口位于 org.apache.ibatis.cursor 包中，定义如下所示：

    
    
    public interface Cursor<T> extends Closeable, Iterable<T> {
      boolean isOpen();
      boolean isConsumed();
      int getCurrentIndex();
    }
    

同时，MyBatis 为 Cursor 接口提供了一个默认实现类 DefaultCursor（位于
org.apache.ibatis.cursor.defaults 包中），核心代码如下：

    
    
    public class DefaultCursor<T> implements Cursor<T> {
      private final CursorIterator cursorIterator = new CursorIterator();
    
      @Override
      public boolean isOpen() {
        return status == CursorStatus.OPEN;
      }
    
      @Override
      public boolean isConsumed() {
        return status == CursorStatus.CONSUMED;
      }
    
      @Override
      public int getCurrentIndex() {
        return rowBounds.getOffset() + cursorIterator.iteratorIndex;
      }
    
        // 省略其他方法    
    }
    

我们看到这里引用了 CursorIterator，从命名上就可以看出这是一个迭代器，其代码如下所示：

    
    
    private class CursorIterator implements Iterator<T> {
        T object;
        int iteratorIndex = -1;
    
        @Override
        public boolean hasNext() {
          if (object == null) {
            object = fetchNextUsingRowBound();
          }
          return object != null;
        }
    
        @Override
        public T next() {
          // Fill next with object fetched from hasNext()
          T next = object;
    
          if (next == null) {
            next = fetchNextUsingRowBound();
          }
    
          if (next != null) {
            object = null;
            iteratorIndex++;
            return next;
          }
          throw new NoSuchElementException();
        }
    
        @Override
        public void remove() {
          throw new UnsupportedOperationException("Cannot remove element from Cursor");
        }
    }
    

上述游标迭代器 CursorIterator 实现了 java.util.Iterator 迭代器接口，这里的迭代器模式实现方法实际上跟 ArrayList
中的迭代器几乎一样。

### 面试题分析

#### **1\. 面向对象语言中模版方法的常见实现方式是什么？MyBatis 中有哪些模板方法的应用场景？**

**考点分析**

这道题考查了模板方法，出题的方式有点特殊，前面的半个问题先让面试者有点意外，因为一般考查设计模式时不大会与具体的语言体系相关联。而后半部分则也偏发散，也没有明确指出
MyBatis 中使用模板方法的具体场景。所以需要面试者有一定的应变能力。

**解题思路**

该题的上半部分看似有点意外，实际上完全是送分题，因为面向对象语言中存在抽象类和抽象方法的概念，在各种开源框架中，实现模板方法最常见也最直接的方法就是使用抽象方法。因此，宽泛地讲，只要存在抽象方法的场景都可以简单认为是模板方法的应用场景。显然，MyBatis
中存在不少这种场景。

**本文内容与建议回答**

本文基于 MyBatis 中的 BaseExecutor 讲解了模板方法的具体实现过程。BaseExecutor 是一个抽象类，存在一批以 doXXX
命名的抽象方法，是模板方法非常典型的一种应用。在 MyBatis 中，BaseExecutor 存在
SimpleExecotor、ReuseExecutor、BatchExecutor 和 ClosedExecutor
四个子类，这些子类分别实现了父类中的模板方法。

#### **2\. MyBatis 中 Executor 的实现有哪几种策略？**

**考点分析**

从问题上讲，我们目前考点是针对 Executor。而从问法上讲，我们同样需要明确该问题考查的是面试者对策略模式的理解程度。

**解题思路**

MyBatis 中针对 Executor 的设计同时使用了模板方法模式和策略模式，实际上这两个模式往往也是结合在一起使用。针对 SQL
执行过程，MyBatis 分别提供了普通执行器 SimpleExecutor、预处理语句重用执行器 ReuseExecutor 以及批处理执行器
BatchExecutor 这三种不同的实现策略。

**本文内容与建议回答**

策略模式本身结构比较简单，基于 Executor 的应用场景也很明确，本文对这两点都做了详细介绍。在关于 MyBatis
设计模式的面试题中，策略模式应该算是比较简单的考查方式。

#### **3\. 如何理解迭代器模式及其在 MyBatis 中的应用？**

**考点分析**

迭代器是一种常见的工具类，也是面试过程中常见的一个考查点。而迭代器模式作为一种经典的设计模式，同样在 MyBatis
中得到了应用。和其他设计模式一样，如果只是从应用角度看，开发人员不会关注迭代器模式与 MyBatis 两者之间的关联关系，因此这道题的难点同样在于对
MyBatis 中实现原理的深入理解。

**解题思路**

针对这道题，我们在回答上最好先列举迭代器模式在 MyBatis 中的应用场景，典型的场景就是用于解析类似
`order[0].address.contactinfo.name` 的属性表达式。而 MyBatis 设计并实现了 PropertyTokenizer
这一工具类来完成对上述表达式的解析过程，该工具类同样实现了 JDK 中的 Iterator 接口。因此，我们在回答时，可以将迭代器设计模式与
Iterator 结合起来一起进行阐述。

**本文内容与建议回答**

本文集中介绍了 MyBatis 中如何基于 PropertyTokenizer 类完成属性表达式解析的整个，重点关于与该类中 hasNext() 和
next() 等迭代器方法。部分内容还是需要进行一定的记忆工作，同样建议根据 MyBatis 中属性表达式的场景进行记忆和回答。

### 日常开发技巧

与上一篇一样，本文同样也介绍了一种在日常开发过程中也可能遇到的场景，即处理类似 `order[0].address.contactinfo.name`
字符串的解析过程。这种场景在处理上同样具有挑战性，设计并实现优雅的解决暗杆并不容易。本文中介绍的迭代器模式是应对这类场景的有效手段，能够从这类问题的本质出发抽象出统一的处理模式。通过本课程的学习，当碰到类似问题时，同样确保能够联想到今天所介绍的内容。

### 小结与预告

本文是经典设计模式系列的最后一篇，我们同样以 MyBatis 这款流行的 ORM
框架为例，讨论了模版方法模式、策略模式和迭代器模式等三种具有代表性的行为型设计模式在该框架中的应用。从下一篇开始，我们将进入一个全新的模块，即本课程的第三个模块《剖析通用机制与组件》。

