通过前面的介绍，我们可以看到 MyBatis 的一级缓存是一个粗粒度的缓存，设计的比较简单。本质上它就是一个 HashMap，MyBatis 没有对
HashMap 的大小进行管理，也没有缓存更新和过期的概念。这是因为一级缓存的生命周期很短，不会存活多长时间。接下来让我们继续研究 MyBatis
中的另一种缓存表现形式，即二级缓存。相较一级缓存，MyBatis 的二级缓存使用方法有所不同，代码结构和类层关系也要更加复杂一些。

### 二级缓存的基本概念

与一级缓存不同，MyBatis 的二级缓存默认是不启用的，如果需要启动，则应该在配置文件中添加如下配置项：

    
    
    <setting name="cacheEnabled" value="true"/>
    

上述配置方法是全局级别的，我们也可以在特定的查询级别使用二级缓存。MyBatis 专门提供了一个 `<cache/>`
配置节点用于实现这一目标，这个配置节点可以定义缓存回收策略（如 FIFO 和 LRU，回想装饰器模式中所介绍的各种缓存实现类）、最多缓存对象数量等参数。

下图展示了 MyBatis 中二级缓存的生效范围，请注意，二级缓存是与命名空间（namespace）强关联的，即如果在不同的命名空间下存在相同的查询
SQL，这两者之间也是不共享缓存数据的。当然，MyBatis 考虑到这种需求，可以通过 `<cache-ref/>`
配置节点来引用命名空间，这样这两个命名空间就可以共用一个缓存。但为了引起不必要的问题，通常也不大建议大家这么做。我们知道，在 MyBatis
中，Configuration 对象管理着所有配置信息，所以所有的二级缓存相当于全部位于 Configuration 之内，如下所示：

![ymGU0k](https://images.gitbook.cn/2020-05-25-052611.png)

### CacheBuilder

回到代码，我们在 org.apache.ibatis.cache
包下试图寻找关于二级缓存的相关实现。但是，我们什么也没有找到。显然，二级缓存的实现应该位于其他包中。联系上图中的这层关系，我们可以想象二级缓存实际上是跟
SQL Mapper 紧密相关的，因为命名空间位于 Mapper 中。所以，相关代码是否是与 Mapper 实现放在一起呢？答案是肯定的，我们在
org.apache.ibatis.mapping 包下找到了这么一个类：CacheBuilder。

从命名上看，CacheBuilder 类的作用就是构建一个 Cache，这个过程通过它的 build 方法实现，该方法代码如下所示：

    
    
    public Cache build() {
        setDefaultImplementations();
        Cache cache = newBaseCacheInstance(implementation, id);
        setCacheProperties(cache);
        if (PerpetualCache.class.equals(cache.getClass())) {
          for (Class<? extends Cache> decorator : decorators) {
            cache = newCacheDecoratorInstance(decorator, cache);
            setCacheProperties(cache);
          }
          cache = setStandardDecorators(cache);
        } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
          cache = new LoggingCache(cache);
        }
        return cache;
    }
    

我们先来看该方法的第一句，即调用 setDefaultImplementations()
方法，该方法设置默认的缓存实现，代码如下。可以看到，这里设置缓存实现为 PerpetualCache 类，同时设置装饰类是 LruCache 类。

    
    
    private void setDefaultImplementations() {
        if (implementation == null) {
          implementation = PerpetualCache.class;
          if (decorators.isEmpty()) {
            decorators.add(LruCache.class);
          }
        }
    }
    

然后，build 方法根据当前命名空间（即 id）以及这个 PerpetualCache 来创建一个 Cache 类的实例。通过反射获取到
Constructor 并调用 newInstance 方法。然后，遍历装饰类接口，对 PerpetualCache
进行各种装饰类的包装，setStandardDecorators 方法如下所示：

    
    
    private Cache setStandardDecorators(Cache cache) {
        try {
          MetaObject metaCache = SystemMetaObject.forObject(cache);
          if (size != null && metaCache.hasSetter("size")) {
            metaCache.setValue("size", size);
          }
          if (clearInterval != null) {
            cache = new ScheduledCache(cache);
            ((ScheduledCache) cache).setClearInterval(clearInterval);
          }
          if (readWrite) {
            cache = new SerializedCache(cache);
          }
          cache = new LoggingCache(cache);
          cache = new SynchronizedCache(cache);
          if (blocking) {
            cache = new BlockingCache(cache);
          }
          return cache;
        } catch (Exception e) {
          throw new CacheException("Error building standard cache decorators.  Cause: " + e, e);
        }
    }
    

在这里，我们终于看到了各种缓存装饰类的创建过程（根据是否定时刷新、是否为只读等属性）。最终构建了

>
> PerpetualCache→LruCache→ScheduledCache→SerializedCache→LoggingCache→SynchronizedCache→BlockingCache

这样一个装饰链，装饰器模式在这里得到了完美的应用。

### MapperBuilderAssistant

各种缓存装饰类的创建方法有了，那么谁来用呢？让我们跟踪 CacheBuilder 类的使用者，MyBatis 中，只有一个地方调用了
CacheBuilder 类，即 MapperBuilderAssistant 类。在该类中存在一个 useNewCache 方法，如下所示：

    
    
    public Cache useNewCache(Class<? extends Cache> typeClass,
          Class<? extends Cache> evictionClass,
          Long flushInterval,
          Integer size,
          boolean readWrite,
          boolean blocking,
          Properties props) {
        Cache cache = new CacheBuilder(currentNamespace)
            .implementation(valueOrDefault(typeClass, PerpetualCache.class))
            .addDecorator(valueOrDefault(evictionClass, LruCache.class))
            .clearInterval(flushInterval)
            .size(size)
            .readWrite(readWrite)
            .blocking(blocking)
            .properties(props)
            .build();
        configuration.addCache(cache);
        currentCache = cache;
        return cache;
    }
    

这里通过 CacheBuilder 创建了一个新的缓存对象，并把它添加到了 configuration 对象中。同时，我们注意到该类中还存在一个
useCacheRef 方法用于缓存引用，该方法如下所示：

    
    
    public Cache useCacheRef(String namespace) {
        if (namespace == null) {
          throw new BuilderException("cache-ref element requires a namespace attribute.");
        }
        try {
          unresolvedCacheRef = true;
          Cache cache = configuration.getCache(namespace);
          if (cache == null) {
            throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.");
          }
          currentCache = cache;
          unresolvedCacheRef = false;
          return cache;
        } catch (IllegalArgumentException e) {
          throw new IncompleteElementException("No cache for namespace '" + namespace + "' could be found.", e);
        }
    }
    

反过来，这里也是根据 configuration 对象中获取缓存实例。请注意，configuration 对象中的缓存对象都是基于命名空间进行引用。

### 二级缓存与 CachingExecutor

现在，二级缓存已经创建完毕并保存在 configuration 对象中，接下来让我们看下在查询过程中如何使用它们。

首先明确一点，在 MyBatis 中，如果开启了二级缓存，不管配置的是哪种类型 Executor，都会将该 Executor 嵌套到
CachingExecutor 类当中。顾名思义，CachingExecutor 是带有缓存机制的 Executor，我们对 Executor
已经有了足够的了解，所以直接来看它的查询方法，如下所示：

    
    
    @Override
      public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
          throws SQLException {
        Cache cache = ms.getCache();
        if (cache != null) {
          flushCacheIfRequired(ms);
          if (ms.isUseCache() && resultHandler == null) {
            ensureNoOutParams(ms, boundSql);
            @SuppressWarnings("unchecked")
            List<E> list = (List<E>) tcm.getObject(cache, key);
            if (list == null) {
              list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
              tcm.putObject(cache, key, list); 
            }
            return list;
          }
        }
        return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }
    

在该方法中，我们通过 `Cache cache = ms.getCache()` 语句从 MappedStatement 当中获取 Cache 对象，而从
MappedStatement 中获取到的 Cache 对象就是保存在 Configuration 中的 Cache 对象。

然后，我们注意到 CachingExecutor 中持有一个新的类 TransactionalCacheManager。当我们执行上述 query
方法时，我们首先会通过 TransactionalCacheManager 获取到 Cache 对象，如果获取到的 Cache
对象为空，那么就执行查询操作，并把查询得到的结果放入 TransactionalCacheManager 中。

我们跳转代码到 TransactionalCacheManager，来研究一下它的 getObject 和 putObject 方法。我们发现在
TransactionCacheManager 对象内部持有一个 HashMap，该 HashMap 的 key 是一个 Cache 对象，value
是一个 TransactionalCache 对象。TransactionalCacheManager 的结构很简单，它的 getObject 和
putObject 方法实际上是调用了 TransactionalCache 类的对应方法。让我们直接进入到 TranscationalCache 中：

    
    
    @Override
    public Object getObject(Object key) {
        Object object = delegate.getObject(key);
        if (object == null) {
          entriesMissedInCache.add(key);
        }
        if (clearOnCommit) {
          return null;
        } else {
          return object;
        }
    }
    
    @Override
    public void putObject(Object key, Object object) {
        entriesToAddOnCommit.put(key, object);
    }
    

从以上代码中，实际上看不出什么内容来，我们只看到了一些针对中间变量的处理。但是，MyBatis 中对于变量的命名是有讲究的，这里的
entriesMissedInCache 和 entriesToAddOnCommit 这两个变量有没有让你想到什么？如何没有想到，再联系一下
TransactionalCache 这个类的名称呢？显然，关键点在于 Transaction（事务）这个概念上。

我们换一个角度，从 TransactionalCache 的 commit 方法入手试试。该方法如下所示：

    
    
     public void commit() {
        if (clearOnCommit) {
          delegate.clear();
        }
        flushPendingEntries();
        reset();
    }
    

首先，我们明确这里的 delegate 就是一个 Cache 对象，MyBatis 中大量使用了
delegate（委托）这个概念来进行类层之间的职责分离。接下来需要关注这里的 flushPendingEntries 方法，如下所示：

    
    
     private void flushPendingEntries() {
        for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
          delegate.putObject(entry.getKey(), entry.getValue());
        }
        for (Object entry : entriesMissedInCache) {
          if (!entriesToAddOnCommit.containsKey(entry)) {
            delegate.putObject(entry, null);
          }
        }
    }
    

该方法调用了 delegate 的 putObject 方法完成了真正意义上的缓存更新。完成这个更新之后，reset() 方法的作用就是清除
entriesToAddOnCommit、entriesMissedInCache 等中间变量的值。

到此，我们终于明白了 TransactionalCache 只是将查询到的数据暂时保存起来，直到 commit
方法执行时，才真正把之前从数据库当中查询到的数据放入到缓存当中。

明确了这一点，我们可以从 TransactionalCache 往前推演整个调用流程。TransactionalCache 中 commit 方法的调用者是
TransactionalCacheManager，TransactionalCacheManager 中的 commit 方法如下所示，所做的事情只是对
TransactionalCache 方法的直接调用：

    
    
    public void commit() {
        for (TransactionalCache txCache : transactionalCaches.values()) {
          txCache.commit();
        }
    }
    

然后 TransactionalCacheManager 的 commit 方法由谁调用呢？显然应该是 CachingExecutor：

    
    
    @Override
    public void commit(boolean required) throws SQLException {
        delegate.commit(required);
        tcm.commit();
    }
    

继续往前推演，我们进一步发现 CachingExecutor 的 commit 是由 DefaultSqlSession
调用的，DefaultSqlSession 也是通过它自己内部的 commit 方法完成了提交。

至此，整个二级缓存的使用过程得到了详细的解释，以 commit 流程为例，整个过程的处理流程如下图所示：

![kxH8I3](https://images.gitbook.cn/2020-05-25-052612.png)

### 面试题分析

#### **简要描述 MyBatis 二级缓存的实现机制？**

**考点分析：**

本题考查 MyBatis
中的二级缓存，相较一级缓存，二级缓存更为复杂，因为需要更多关注于缓存对象的生命周期以及具体的缓存策略。有时候，这个考点也可以与《经典设计模式：结构型模式及其在
MyBatis 中的应用（下）》中介绍的装饰器模式结合在一起进行考查。

**解题思路：**

同样，针对这道面试题，我们也是从应用方法上进行切入。二级缓存的启动需要我们通过配置项进行控制，而它的作用范围取决于 MyBatis
的命名空间定义。MyBatis 也专门提供了 `<cache/>` 和 `<cache-ref/>`
标签来供开发人员有效的管理二级缓存。从实现原理上，二级缓存的回答内容中需要包含三个要点。第一个要点是通过装饰器模式来构建具体种类的缓存对象，这块我们需要结合设计模式相关内容进行展开；第二个要点是它与
CachingExecutor 之间的交互关系。最后一点在于与事务相关的处理方式，位于 TransactionalCache
类中。只有把这些要点都解释清楚，才能很好的完整这个面试题。

**本文内容与建议回答：**

本文同样对解题思路中提到的各个要点都给出理详细的说明和解释。同时，本文最后部分还专门提供了一个流程图来描述二级缓存的整个执行过程，方便大家进行记忆。

### 小结与预告

结合前一篇内容，我们通过两篇文章分别介绍了 MyBatis
框架中的一级缓存和二级缓存。作为整个课程中《剖析通用机制与组件》模块的最后一部分内容，我们也将通过两篇文章来介绍 MyBatis
中的连接池实现原理，下一篇将先讨论资源池模式与数据库连接池的基本概念。

