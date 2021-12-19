缓存（Cache）技术在互联网系统的开发过程中应用非常广泛。在业务关键链路，当性能问题逐渐成为系统运行的瓶颈时，很多场合需要我们分析系统实现方案，并使用以缓存为代表的架构设计手段重构业务关键链路。缓存的作用在于减少数据的访问时间和计算时间，表现为把来自持久化或其它外部系统的数据转变为一系列可以直接从内存获取的数据结构的过程。

### 如何理解缓存的设计策略？

想要理解缓存的设计策略，我们一般都会基于一定的缓存架构体系展开讨论。本篇内容将从经典的缓存分层架构出发探讨如何应用缓存的方法，但是这是一个非常大的主题，这里仅做简要介绍。而我们的课程关注与如何理解
MyBatis 等应用层开源框架中的缓存实现策略，所以我们也会对应用层的缓存设计方法进行展开。

#### 经典缓存分层架构

在学习互联网系统中的主流架构时，我们经常会看到类似如下所示的架构图，这张图可以理解为是缓存应用的分层架构图。

![dDTvu2](https://images.gitbook.cn/2020-05-25-052609.png)

在上图中，我们可以看到清晰的三层缓存架构。当用户请求过来时会首先经过 Nginx 层，Nginx 本身带有缓存机制可用作数据缓存。然后，如果请求在
Nginx 中找不到数据，则直接请求到后台服务系统，这时候后台服务系统会到以 Redis 为代表的第三方分布式缓存系统中去查询数据。更进一步，如果在
Redis 中也查询不到数据，则从应用程序缓存中查询数据，这里的应用程序泛指诸如 Tomcat 等应用程序容器，也同时包括像
Spring、Dubbo、MyBatis
等的开源框架，以及我们自己开发的业务系统。最后，如果在应用程序中也查询不到数据，则直接请求数据库，然后将数据缓存到应用程序缓存、Redis 缓存和
Nginx 缓存中。

上述的三级缓存架构比较适用于数据变动不是很大，但请求并发量比较大的场景。我们也要考虑数据库与缓存中数据一致性的问题。

#### **应用层缓存的分级模式**

本课程关注的是上图中位于应用程序层的缓存。如果对其进行进一步分析，我们发现应用程序层的缓存同样存在分级模式，这种分级模式通常包括两级，即一级缓存和二级缓存。

简单讲，所谓的一级缓存就是指请求级别的或者说会话级别的缓存。针对每次查询操作，一级缓存会把数据放在会话中，如果再次执行查询，就会直接从会话的缓存中获取数据，而不会去查询数据库。二级缓存则范围更大一点，它是一种全局作用域的缓存，在整个应用开启运行状态期间，所有请求和会话都可以使用。

我们接下去将基于 MyBatis 框架来分析它的一级缓存和二级缓存。

### MyBatis 一级缓存

在 MyBatis 中，一级缓存对应的接口是 Cache，同时还提供了针对该接口的众多实现类。我们在《经典设计模式：结构型模式及其在 MyBatis
中的应用（下）》一文中讨论装饰器设计模式在 MyBatis 的应用时已经看到过 Cache 的类层结构。我们知道除了 PerpetualCache
类之外，其他的实现类都是 Cache 的装饰器。

#### **PerpetualCache**

PerpetualCache 是 MyBatis 中默认使用的缓存类型，其代码如下所示：

    
    
    public class PerpetualCache implements Cache {
    
      private final String id;
    
      private Map<Object, Object> cache = new HashMap<>();
    
      //省略构造函数、getId 和 getSize 方法
    
      @Override
      public void putObject(Object key, Object value) {
        cache.put(key, value);
      }
    
      @Override
      public Object getObject(Object key) {
        return cache.get(key);
      }
    
      @Override
      public Object removeObject(Object key) {
        return cache.remove(key);
      }
    
      @Override
      public void clear() {
        cache.clear();
      }
    
      @Override
      public boolean equals(Object o) {
        if (getId() == null) {
          throw new CacheException("Cache instances require an ID.");
        }
        if (this == o) {
          return true;
        }
        if (!(o instanceof Cache)) {
          return false;
        }
    
        Cache otherCache = (Cache) o;
        return getId().equals(otherCache.getId());
      }
    
      @Override
      public int hashCode() {
        if (getId() == null) {
          throw new CacheException("Cache instances require an ID.");
        }
        return getId().hashCode();
      }
    }
    

可以看到，整个 PerpetualCache 类的代码结构非常简单，除了一个 id 变量之外，保存缓存数据的 cache 变量也只是一个
HashMap，是一种典型的基于内存的缓存实现方案。这里的几个方法也比较简单，所有对缓存的操作实际上就是对 HashMap 的操作。

我们可以想象 HashMap 中 Key-Value 对中的 Value 就是要缓存的数据，那么如何定义它的 Key 呢？MyBatis 提供了专门的
CacheKey 类。相比 PerpetualCache，CacheKey 反而更加复杂一点。

CacheKey 的目的就是判断查询是否相同，所以它是根据每次查询操作的特征值抽象而成的类。在 Java 世界中，判断两个对象是否相等，我们可以使用
equals 方法，而 equals 方法相等的前提是 hashcode 要相等，CacheKey 在实现上采用的是经典的 hashcode 算法。在
CacheKey 的实现代码中包含了若干个属性，其中以下属性都与这一算法相关（在构造函数中，分别把 DEFAULT_MULTIPLYER 和
DEFAULT_HASHCODE 赋值给了 multiplier 和 hashcode）：

    
    
    private static final int DEFAULT_MULTIPLYER = 37;
    private static final int DEFAULT_HASHCODE = 17;
    private final int multiplier;
    private int hashcode;
    private long checksum;
    private int count;
    private List<Object> updateList;
    

然后我们再来看它的 update 方法，CacheKey 通过 update 方法加入一个特征值对象：

    
    
    public void update(Object object) {
        int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);
    
        count++;
        checksum += baseHashCode;
        baseHashCode *= count;
    
        hashcode = multiplier * hashcode + baseHashCode;
    
        updateList.add(object);
    }
    

上述代码中，count 用于表示特征值数量，checksum 表示所有特征值的 hashcode 之和。而每次更新之后新的 hashcode = 原来的
hashcode * 扩展因子（37） + 新特征值的 hashcode。这样做可以使不同 CacheKey 对象的 hashcode
的碰撞率控制在一个极小的概率上。

了解了如何计算 hashcode，接下来我们再来看它的 equals 方法，如下所示：

    
    
      @Override
      public boolean equals(Object object) {
        if (this == object) {
          return true;
        }
        if (!(object instanceof CacheKey)) {
          return false;
        }
    
        final CacheKey cacheKey = (CacheKey) object;
    
        if (hashcode != cacheKey.hashcode) {
          return false;
        }
        if (checksum != cacheKey.checksum) {
          return false;
        }
        if (count != cacheKey.count) {
          return false;
        }
    
        for (int i = 0; i < updateList.size(); i++) {
          Object thisObject = updateList.get(i);
          Object thatObject = cacheKey.updateList.get(i);
          if (!ArrayUtil.equals(thisObject, thatObject)) {
            return false;
          }
        }
        return true;
    }
    

可以看到，整个 equals
方法经历了多层卫语句判断，而判断特征集集合的相等性只是最后一步，如果两个对象本身是不等的，那么需要对特征值集合进行比较的可能性就非常之小。这种设计思想和实现方法对我们日常的开发工作也有一定的参考价值。

#### **一级缓存与 BaseExecutor**

MyBatis 中存在一个配置项，用于指定一级缓存默认开启的级别，如下所示：

    
    
    <setting name="localCacheScope" value="SESSION"/>
    

在 MyBatis 中一级缓存存在两个级别，即 SESSION 级和 STATEMENT 级，默认采用的是 SESSION
级，也就是会话级别。如果将其设置为 STATEMENT 级，可以理解为缓存只对当前 SQL 语句有效，SESSION
当中的缓存数据在每次查询之后就会被清空。而如果是 SESSION 级，则查询结果一直会位于该 Session 中。但是，要注意由于一级缓存是独立存在于每个
Session 内部的，因此，如果我们创建了不同的 Session，那么他们之间会使用不同的缓存。例如，完全一样的操作，如果在两个不同的 Session
中执行，那就意味着存在两份一样的缓存数据，但分别位于两个 Session 中，彼此之间不会共享。

是时候回到 SqlSession 了，我们已经在《基于核心执行流程剖析代码结构：MyBatis 框架主流程分析》中分析了 SqlSession
的整体执行流程。让我们回顾一下 DefaultSqlSession 中最具代表性的 selectList 方法，代码如下所示：

    
    
    @Override
    public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
        try {
          MappedStatement ms = configuration.getMappedStatement(statement);
          return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
        } catch (Exception e) {
          throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
        } finally {
          ErrorContext.instance().reset();
        }
    }
    

我们知道，SqlSession 使用了经典的外观模式，它将具体的查询委托给了 Executor。而 Executor 又使用了模板方法模式，所以这里会先调用
BaseExecutor 中的 query 方法，我们来看一下该方法中的具体实现：

    
    
    @Override
    public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
        BoundSql boundSql = ms.getBoundSql(parameter);
        CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
        return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
    

可以看到在 BaseExecutor 中会根据当前查询 SQL、参数等信息生成一个 CacheKey，具体的实现方法为
createCacheKey。我们在该方法中发现，MyBatis 会新建一个 CacheKey 实例，并根据传入的这些信息反复调用 CacheKey 的
update 方法以生成特征值。

当 CacheKey 实例创建完成，BaseExecutor 把该 CacheKey 传入到自身的 query 方法执行查询。这个 query
方法的完整代码如下所示：

    
    
    @Override
    public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
        if (closed) {
          throw new ExecutorException("Executor was closed.");
        }
    
        //如果设置了 isFlushCacheRequired 参数，则清除所有一级缓存
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
            //如果缓存中找不到数据，则查询数据库
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
    
          //如果是 STATEMENT 级，则清空当前的本地缓存
          if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {        
            clearLocalCache();
          }
        }
        return list;
    }
    

上述方法的逻辑有点复杂，首先我们来看最后一个 if 判断，可以看到如果一级缓存被设置为 STATEMENT
级，那么该方法会清空当前的本地缓存。这点跟前面的讨论一致。

然后我们回到方法开头，这里会判断 queryStack 是否为 0，如果该值是 0 且当前的 SQL 语句中对 isFlushCacheRequired
参数进行了设置，那就会把 Session 的 localCache 清空。请注意，MyBatis
一级缓存的启动过程不需要任何配置，它是强制开启的，我们无法取消。但是通过动态 SQL 的 isFlushCacheRequired
参数可以强制清除所有一级缓存。

针对上述代码中所使用到的 localCache 对象，通过查看定义，我们发现它就是一个 PerpetualCache 对象。接着，我们根据 SQL 的
cacheKey 来对 localCache 进行查询看是否存在对应的数据。如果查询到的结果集为空，则去数据库中查询，代码如下：

    
    
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
    

显然，一旦完成数据库查询，会将从数据库中获取的数据保存在 localCache 中。

如果读者查看 Executor 的 update、commit 和 close 方法，会发现这些方法在执行完毕之后都会清空一级缓存。这里不再具体展开。

### 面试题分析

#### **简要描述 MyBatis 一级缓存的实现机制？**

**考点分析：**

关于 MyBatis
中缓存机制的面试题，一般都会考查所谓的一级缓存和二级缓存。从考点上，因为比较容易混淆，我们首先需要明确这两种缓存的区别，可以从缓存的作用范围进行记忆，作用范围小的是一级，反之是二级。同时，我们也要明确，这种面试题往往会考查实现原理而不仅是简单应用，因为在
MyBatis 中缓存的应用更多是被动的，而不是开发人员主动进行控制。

**解题思路：**

针对问题，我们需要先给出一级缓存的具体使用方法，这点比较简单，但要指明一级缓存中存在请求级别和会话级别的两种处理方式，我们可以通过配置项进行控制。然后我们需要进一步明确，MyBatis
中的一级缓存实际上就是一个 HashMap，并没有太多涉及到缓存管理方面的操作。同时，我们需要解释一下一级缓存与 BaseExecutor 之间的关系，在
Executor 执行具体的查询操作时，就会应用到一级缓存。如果缓存中找不到对应数据才会执行数据库查询。作为加分项，这里可以重点解释一下 MyBatis 中
CacheKey 的实现方法，这也是 MyBatis 框架本身在设计上的一大亮点。

**本文内容与建议回答：**

针对上述解题思路中提到的各个回答要点，本文都给出了对应的知识体系。部分内容，确实需要结合源代码才能够理解的更加清晰明白，建议大家还是要阅读
PerpetualCache、BaseExecutor 等核心类中的代码执行主流程

### 日常开发技巧

本文关于 MyBatis 中 CacheKey 的内容可以用于指导日常的开发工作。缓存键的设计和实现是一大难点，MyBatis 中采用了还是经典的
hashcode 算法，但具体的计算过程使用了一系列特征值，我们对此都进行了详细介绍。我们可以基于本文中的内容来设计并实现可用于各种场合的自定义 Key。

### 小结与预告

本文介绍了 MyBatis 中的一级缓存，一级缓存的作用范围较小，只能应用与请求级别或会话级别。在下一篇中，我们将介绍具有更大作用范围的 MyBatis
二级缓存。

