上一篇中我们基于核心执行流程分析了 Mybatis 中 SQL 执行的部分内容，今天我们继续来讨论剩下的流程，并尝试将整个流程中所涉及的各个组件进行整合。

### Mybatis Mapper

我们回到主流程，现在我们已经获取了 SqlSession，接下来就要讨论 Mapper 部分的内容，获取 Mapper 的过程如下图所示。

![](https://images.gitbook.cn/2020-05-25-052732.png)

从上图中，我们看到 Mybatis 通过 MapperProxy 动态代理 Mapper。也就是说当执行自定义 Mapper 中的方法时，其实是对应的
MapperProxy 在代理进行执行。获取 MapperProxy 对象的方式是通过 SqlSession 从 Configuration
中进行获取，以下代码位于 org.apache.ibatis.session.defaults.DefaultSqlSession 中。

    
    
    @Override
    public <T> T getMapper(Class<T> type) {
        return configuration.getMapper(type, this);
    }
    

可以看到 SqlSession 把包袱甩给了 Configuration, 接下来就看看 Configuration 中的相关操作，代码如下。

    
    
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        return mapperRegistry.getMapper(type, sqlSession);
    }
    

跳转到 mapperRegistry 中的 getMapper()方法，我们就看到了上图中的 MapperProxyFactory 类，代码如下。

    
    
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
        if (mapperProxyFactory == null) {
          throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
        }
        try {
          return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception e) {
          throw new BindingException("Error getting mapper instance. Cause: " + e, e);
        }
    }
    

现在 MapperProxyFactory
是一个代理工厂类，再往下走就是动态代理的内容了。但是，今天我们的主题是关注主流程，动态代理的部分这里不展开，我们将在本课程的剖析通用机制与组件模块中详细讨论代理模式及其应用。在主流程上，大家只需要知道最终各个
Mapper 是通过 MapperRegistry 进行获取。最终的效果就是最初的代码示例。

    
    
    TestMapper testMapper = sqlSession.getMapper(TestMapper.class);
    
    TestObject testObject = testMapper.getObject("testCode1");
    

### Mybatis Executor

事实上，各个通过代理获取的 Mapper，在执行过程中还是会回到 SqlSession。本节内容将关注 SqlSession 的 CRUD
操作。而在执行过程中，这些操作实际上是交给 Executor 去处理。我们挑选 SqlSession 中的 selectList()来做演示，代码如下。

    
    
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
    

显然，这里使用 executor 的 query()方法来执行操作。请记住，Mybatis 中 Executor 接口有两个实现类，一个是
CachingExecutor，另一个是
BaseExecutor。前者会使用二级缓存，而后者使用的是一级缓存，也就是本地缓存。关于缓存部分的介绍也不是本节内容的重点，在《剖析通用机制与组件》模块中同样会专门来介绍这部分内容。从流程上，这时候的
executor 是一个 CachingExecutor，先从二级缓存中获取结果，以下代码位于
org.apache.ibatis.executor.CachingExecutor 中。

    
    
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
    

如果二级缓存中没有数据，那么代码就会执行到上面的 delegate.query ()方法，该方法会执行 BaseExecutor 中的 query
方法，也就是会使用一级缓存。如果一级缓存也没有数据，则走 queryFromDatabase 方法查数据库。以下代码就是 BaseExecutor
中的执行逻辑

    
    
    if (list != null) {
         handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
         list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
    

queryFromDatabase()方法会从数据库查到数据，然后放入到一级缓存中，如下所示。

    
    
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
    

然后，通过一层一层的调用，最终会来到 doQuery 方法， 我们来看一下 SimpleExecutor 中该方法的实现过程，代码如下。

    
    
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
    

上面代码中 StatementHandler 存在几个实现类，以最常用的封装了 PreparedStatement 的
PreparedStatementHandler 为例，我们来看看它的 query()方法。

    
    
    @Override
    public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
        PreparedStatement ps = (PreparedStatement) statement;
        ps.execute();
        return resultSetHandler.handleResultSets(ps);
    }
    

终于，我们看到了经典的 PreparedStatement，一切又回到了 JDBC。至此整个执行流程执行完毕。

### 整合：流程执行的结果

在本节的最后，我们试图把前面介绍个各个核心类整合在一起完整描述流程执行的过程，结果如下所示。

![](https://images.gitbook.cn/2020-05-25-052733.png)

从上图中可以看到，我们首先是 Configuration 对象的获取是通过对 Mybatis 配置文件的解析。然后 SqlSession
的获取使用了工厂方法。而用户自定义 Mapper 中方法的执行则是通过 Java
中的代理模式来实现的，对执行结果的处理也分别使用使用了二级缓存和一级缓存。本节内容的讨论还是关注于整体的执行流程，上述过程中所涉及的设计模式、代理机制、缓存机制等知识点我们会在本课程其他章节中再做展开。

### 面试题分析

#### Mybatis 中 Executor 的类型以及与缓存之间的关系？

  * 考点分析：

这个问题考查了大家对 Mybatis 内部结构理解，因为 Executor 这个对象并没有通过 API 的方式开放给开发人员。

  * 解题思路：

在 Mybatis 中，存在一个 BaseExecutor 和一个 CacheExecutor，其中后者是对前者的一种封装。而关于 Executor
的内容由于与缓存机制密切相关，需要整合在一起进行回答。

  * 本文内容与建议回答：

本文给出了 CachingExecutor，会使用二级缓存。而 CachingExecutor 又依赖于使用一级缓存的
BaseExecutor。这部分内容已经可以用来回答这个问题，如果要对一级缓存和二级缓存做进一步展开，可以参考《剖析通用机制与组件》模块中关于
Mybatis 缓存机制的介绍。

#### Mybatis 如何与底层的 JDBC 进行集成？

  * 考点分析

这是一道比较经典的问题，考查 Mybatis 与 JDBC 之间的关系。我们需要先明确 Mybatis 作为一种半自动化的 ORM 框架，只是对 JDBC
做了一层封装，所以最终的 SQL 执行还是要通过 JDBC，并对最终的结果做不同程度的封装。

  * 解题思路

在 Mybatis 中提供了一些类的 Handler 机制类处理 Statement/PreparedStatement 以及
ResultSet。在面试过程中，能够列举具体 1 到 2 个具体的 Handler 的实现细节，然后把 JDBC 的 SQL 的执行过程、结果嵌入到
Mybatis 的整体流程中应该就可以了。

  * 本文内容与建议回答

本文中以 StatementHandler 为例，给出了 Handler 机制的一种应用场景，即 query
方法如何处理查询类操作。回答这个问题时，可以以这个示例为基础做一定的发散。

#### 为什么 Mybatis 的 Mapper 只是接口没有实现类却能完成实现类的功能？

  * 考点分析

以笔者的经历，很多面试者回答不上来这道题的原因是不知道具体的考点是什么。实际上，这道题背后考查的是大家对代理机制的理解程度。

  * 解题思路

明确了考点实际上是在问代理机制之后，我们进一步明确代理机制的实现原理，这类问题也基本属于具有固定回答模式的问题，比较简单。很多时候，只要能说出代理机制这个概念，这道题可能就能通过。实际上，这类问题也比较常见，不光只是限于
Mybatis，在 Spring Cloud、Spring Data 等主流开源框架中都有类似的实现。

  * 本文内容与建议回答

本文并没有对代理机制做详细展开，但给出了 MapperProxyFactory
的概念以及接口定义。代理机制是一个非常重要且有用的技术，我们在课程的《剖析通用机制与组件》模块中会有专题讨论，并会给出 Mybatis
中基于代理机制实现数据访问的底层原理。

### 小结与预告

前面我们同样通过两篇文章从“基于核心执行流程剖析代码结构”这一主题出发对 Mybatis 的核心执行过程做了分析。现在，我们已经能够从宏观角度把握
Mybatis 框架中的各个核心步骤。这里面提到的很多核心组件我们都将在课程的后续内容中做全面展开。从下一篇开始，我们将回到 Dubbo
框架，并使用比较多的篇幅来讨论下一个分析代码结构的主题，即"基于基础架构组织剖析代码结构"。

