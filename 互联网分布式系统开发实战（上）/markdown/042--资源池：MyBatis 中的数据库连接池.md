与介绍缓存时采用的自下而上（即从底层实现类 PerpetualCache 出发向上游逐层分析直到系统的访问入口
SqlSession）的分析方法不同，我们将采用自上而下的策略来介绍 MyBatis 中的数据库连接池，即从如何获取 Connection
对象开始逐步分析其背后的池化操作。

### Connection 对象获取过程

我们在介绍缓存时，已经明确了 DefaultSqlSession 会调用 CachingExecutor，然后 CachingExecutor 是
BaseExecutor 的实现类，而 BaseExecutor 又实现了 Executor 接口，该接口提供了多种用于数据 CRUD
的方法。有了这层调用链，我们可以想象为了获取与数据库访问相关的 Connection，入口应该是在 BaseExecutor
中的几个与数据库访问直接相关的方法中。关于 BaseExecutor 的类层结构我们已经在《经典设计模式：行为型模式及其在 MyBatis
中的应用》一文中介绍模板方法时做了介绍，这里不再重复，所以我们直接来到了 BaseExecutor 的子类 SimpleExecutor 中的
doQuery 方法。在该方法中我们看到了一个 prepareStatement 方法，进一步猜想获取 Connection
应该是在这个阶段，让我们来看一下 prepareStatement 方法的定义：

    
    
    private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
        Statement stmt;
        Connection connection = getConnection(statementLog);
        stmt = handler.prepare(connection, transaction.getTimeout());
        handler.parameterize(stmt);
        return stmt;
    }
    

果然，在这里我们看到了获取 Connection 对象的 getConnection 方法，该方法如下所示：

    
    
    protected Connection getConnection(Log statementLog) throws SQLException {
        Connection connection = transaction.getConnection();
        if (statementLog.isDebugEnabled()) {
          return ConnectionLogger.newInstance(connection, statementLog, queryStack);
        } else {
          return connection;
        }
    }
    

我们发现，getConnection 实际上是 Transaction 接口的一个方法。除了 getConnection 方法之外，Transaction
作为事务类还包含 commit 和 rollback 等事务处理相关接口。在 MyBatis 中，Transaction
接口的类层接口如下所示，可以看到两个实现类，即 JdbcTransaction 和 ManagedTransaction。

![OmR8th](https://images.gitbook.cn/2020-05-25-052559.png)

JdbcTransaction 使用 JDBC 的事务管理机制，即利用 java.sql.Connection 对象完成对事务的提交。而
ManagedTransaction 使用 MANAGED 的事务管理机制，在这种机制下 MyBatis 自身不会去实现事务管理，而是让 Tomcat
等容器来实现对事务的管理。无论哪种方案，我们在各自的 getConnection 方法中都能看到以下语句，即通过 DataSource 来获取
Connection 对象。

    
    
    this.connection = this.dataSource.getConnection();
    

以执行查询操作的 query 方法为例，整体获取 Connection 对象的工作流程如下图所示：

![iKdlg9](https://images.gitbook.cn/2020-05-25-052600.png)

### PooledDataSource

DataSource 是 JDBC 中定义的接口，该接口只包含两个重载的 getConnection() 方法。MyBatis
实现了该接口，并提供了三种实现方案，即基于 JNDI 的工厂类 JndiDataSourceFactory（用于从配置信息中获取 DataSource
实例）、普通的非池化类 UnpooledDataSource 以及池化类 PooledDataSource。本课程的重点是介绍
PooledDataSource，但也会涉及一部分 UnpooledDataSource 的介绍，因为 PooledDataSource 是构建在
UnpooledDataSource 的基础之上。

MyBatis 中与 DataSource 接口相关的工厂类和实例类类层关系如下图所示：

![5UoGSS](https://images.gitbook.cn/2020-05-25-052602.png)

我们看到 MyBatis 使用了工厂模式来创建各种 DataSource 实例，有了《经典设计模式：创建型模式及其在 MyBatis
中的应用》中工厂模式的介绍，上面的类层结构图就显得非常简单，我们不做展开，而是直接切入 PooledDataSource
类。在该类中，首先我们看到的是一组连接池的参数，如下所示。可以看到诸如最大空闲连接数（poolMaximumIdleConnections）、连接池最大持有连接数（poolMaximumActiveConnections），最大存在时间（poolMaximumCheckoutTime）等核心参数，我们在上一篇中介绍连接池的核心要素时介绍过这里的部分参数。

    
    
    protected int poolMaximumActiveConnections = 10;
    protected int poolMaximumIdleConnections = 5;
    protected int poolMaximumCheckoutTime = 20000;
    protected int poolTimeToWait = 20000;
    protected int poolMaximumLocalBadConnectionTolerance = 3;
    protected String poolPingQuery = "NO PING QUERY SET";
    protected boolean poolPingEnabled;
    protected int poolPingConnectionsNotUsedFor;
    

#### **popConnection 获取连接**

然后，我们来看 PooledDataSource 的 getConnection() 方法，如下所示：

    
    
    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return popConnection(username, password).getProxyConnection();
    }
    

继续跟进，来到 popConnection
方法，该方法非常长，但结构比较清晰。基于上一篇中介绍的关于连接池的管理方法，理解这段代码难度并不大。为了更好的把握代码结构，我们对该方法的代码进行裁剪，只关注于主要的分支和流程，如下所示：

    
    
    private PooledConnection popConnection(String username, String password) throws SQLException {        
            while (conn == null) {
              synchronized (state) {
                if (!state.idleConnections.isEmpty()) {
                  // 如果 idle 列表不为空，表示有可用连接，直接选取第一个元素
                  conn = state.idleConnections.remove(0);            
                } else {
                  // 连接池没有可用连接的场景
                  if (state.activeConnections.size() < poolMaximumActiveConnections) {
                    // 如果 active 列表没有满，直接创建新连接
                    conn = new PooledConnection(dataSource.getConnection(), this);          
                  } else {// active 已经满了
                    // 获得使用最久的连接，判断是否已经超时
                    PooledConnection oldestActiveConnection = state.activeConnections.get(0);
                    long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
                    if (longestCheckoutTime > poolMaximumCheckoutTime) {
                      // 已经超时，将原连接废弃并建立新连接                
                      conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);                  
                    } else {
                      // 如果没有超时，则进行等待，并计算时间累加
                      state.wait(poolTimeToWait);
                    }
                  }
                }
                if (conn != null) {
                  // 如果获取到了连接，则验证该连接是否有效
                  if (conn.isValid()) {
                    // 如果连接有效，更新该链接相关参数和状态
                  } else {
                    // 如果连接无效，且无效连接数量超过上限则抛异常               
                  }
                }
              }
            }
    
            if (conn == null) {
                // 如果最终无法获取有效连接，则同样抛异常  
            }
    
            return conn;
    }
    

在上述代码中，我们添加了很多注释来解释用于获取 Connection 的 popConnection 方法的整体流程，而不关心其具体细节。在整体流程上，当
MyBatis 执行查询时会首先从 idleConnections 列表中申请一个空闲的连接，只有当 idleConnections
列表为空时才会创建新连接。当然 PooledDataSource 并不允许无限建立新连接，当连接池中连接数目达到一定数量时，即使
idleConnections 列表为空，也不会建立新连接。而是从 activeConnections
列表中找出使用最久的一个连接，判断其是否超时。如果超时，则将该连接废弃并建立新连接，否则线程等待直到连接池中有新的可用连接。注意到这里有一个
state.wait(poolTimeToWait) 语句执行了线程等待，关于该语句的作用后面会有进一步说明。

但是，我们同样注意到这里有一个 PooledConnection 类，PooledDataSource 中创建的就是这个类。该类是 MyBatis
自己设计的一个 Connection 类，让我们来看一下。

在 PooledConnection 中我们首先应该注意到的是它的两个变量，即 realConnection 和
proxyConnection，它们的类型都是 Connection。从命名上看，其中的 realConnection 是真正通过 JDBC
驱动建立的连接，而 proxyConnection 是一个代理连接，也是 PooledDataSource 返回的连接。PooledConnection
本身就实现了 JDK 的 InvocationHandler 接口，并提供了如下所示的 invoke 方法实现：

    
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {
          dataSource.pushConnection(this);
          return null;
        }
        try {
          if (!Object.class.equals(method.getDeclaringClass())) {
            checkConnection();
          }
          return method.invoke(realConnection, args);
        } catch (Throwable t) {
          throw ExceptionUtil.unwrapThrowable(t);
        }
    }
    

这里的逻辑主要是 DataSource 的 pushConnection 方法，我们来看一下。

#### **pushConnection 释放连接**

我们采用与 popConnection 方法相同的方式来介绍 pushConnection 方法，如下所示：

    
    
    protected void pushConnection(PooledConnection conn) throws SQLException {
            synchronized (state) {
              // 从 active 列表中移除该连接
              state.activeConnections.remove(conn);
              // 如果是有效连接
              if (conn.isValid()) {
                // 如果满足 idle 列表没有填满且类型符合期望
                if (state.idleConnections.size() < poolMaximumIdleConnections && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
                  state.accumulatedCheckoutTime += conn.getCheckoutTime();            
                  // 则创建一个新的连接并放入 idle 列表中
                  PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
                  // 唤醒线程
                  state.notifyAll();
                } else {              
                  // 如果不满足判断条件，则说明该连接已经不需要存储，直接关闭真正连接
                  conn.getRealConnection().close();          
                }
              } else {
                // 如果是无效连接，则记录
              }
            }
    }
    

结合 popConnection 方法，pushConnection 方法的整体流程也在意料之中。这里唯一需要注意一点在于
state.notifyAll() 语句，该语句与 popConnection 方法中用于等待线程的 state.wait(poolTimeToWait)
语句相对应，从而唤醒了线程。这样，当 SQL 执行完成时，PooledDataSource 也不会直接关闭线程，而是将其加入到
idleConnections 中并唤醒所有等待线程。

这里用到了一个新的类 PoolState，该类管理所有连接的状态，其包含的参数如下所示：

    
    
    protected PooledDataSource dataSource;
    
    protected final List<PooledConnection> idleConnections = new ArrayList<>();
    protected final List<PooledConnection> activeConnections = new ArrayList<>();
    protected long requestCount = 0;
    protected long accumulatedRequestTime = 0;
    protected long accumulatedCheckoutTime = 0;
    protected long claimedOverdueConnectionCount = 0;
    protected long accumulatedCheckoutTimeOfOverdueConnections = 0;
    protected long accumulatedWaitTime = 0;
    protected long hadToWaitCount = 0;
    protected long badConnectionCount = 0;
    

这里我们找到了 idleConnections 和 activeConnections 列表以及执行过程中的
accumulatedCheckoutTime、badConnectionCount 等在 popConnection 和 pushConnection
方法中用到的数据累积变量。

我们用一张围绕 popConnection 和 pushConnection 方法的全流程图来结束对 PooledDataSource
的讨论，如下所示，这张图包含了流程中的各个主要步骤，其中 popConnection 和 pushConnection
通过线程的等待和唤醒完成相互之间的协作：

![](https://images.gitbook.cn/58c5f7b0-d4f0-11ea-baf7-95cdfa1a7574)

#### **UnpooledDataSource**

接下来我们来讨论 UnpooledDataSource。在 MyBatis 中，PooledDataSource 实际上也是依赖
UnpooledDataSource 来创建真正的连接，这点通过 PooledDataSource 类的构造函数可以得到印证：

    
    
    public PooledDataSource(String driver, String url, String username, String password) {
        dataSource = new UnpooledDataSource(driver, url, username, password);
        expectedConnectionTypeCode = assembleConnectionTypeCode(dataSource.getUrl(), dataSource.getUsername(), dataSource.getPassword());
    }
    

所以，PooledDataSource 中的 dataSource 变量实际上就是一个 UnpooledDataSource 对象。在
PooledDataSource 中的 popConnection 方法中，我们看到有如下语句：

    
    
    conn = new PooledConnection(dataSource.getConnection(), this);  
    

这里通过 UnpooledDataSource 获取了 Connection 并构建 PooledConnection 对象，所以前面提到的
PooledConnection 中的 realConnection 对象实际就是来自 UnpooledDataSource。

我们有必要来分析一下 UnpooledDataSource 中获取 Connection 的过程。我们跟踪 UnpooledDataSource 的
getConnection 方法，找到如下所示的 doGetConnection 方法。

    
    
     private Connection doGetConnection(Properties properties) throws SQLException {
        initializeDriver();
        Connection connection = DriverManager.getConnection(url, properties);
        configureConnection(connection);
        return connection;
      }
    

显然，这里回归到了熟悉的 JDBC，实际上就是基于 JDBC 中的 DriverManager 工具类来获取 Connection。

### 面试题分析

#### **简要描述 MyBatis 中对数据库连接对象 Connection 的管理过程？**

**考点分析：**

这道题问的是 Connection 的管理过程，本质上还是在考查大家对连接池的理解程度，因为 Connection
的管理就是通过连接池来完成的。对于普通的应用场景而言，Connection 的管理对于开发人员是透明的，所以该题的考点有一定难度，需要面试者对
MyBatis 内部实现原理有深入的理解。

**解题思路：**

在解答思路上，笔者认为这道题的主要挑战是梳理 Connection 在使用上的整个流程，该流程涉及到两大方面，即获取 Connection 和释放
Connection。为此，MyBatis 中分别提供了 popConnection 和 pushConnection
方法。这两个方法是需要明确点到的，这是一个要点。另一方面，从连接池的管理上，获取和释放 Connection
并不是很复杂，无非就是要做很多判断和异常处理，但如何确保并发访问下新连接的创建和访问是一个难题，需要指出 MyBatis
在多线程处理上的实现方式，这是该问题的第二个要点。

**本文内容与建议回答：**

本文详细介绍了 MyBatis 中 PooledConnection 类是设计思想和实现方法，同时也给出了 popConnection 和
pushConnection
这两个核心方法的流程细节。为了大家进行高效的理解和记忆，本文最后还提供了一张流程图来详细总结这两个方法中的各个步骤，以及两者之间通过多线程技术进行交互的过程。

### 日常开发技巧

通过本文的学习，笔者认为 MyBatis 中关于 PooledConnection 的设计思想和实现过程值得我们做深入的分析，并应用到日常的开发过程中。在
PooledConnection 中存在 realConnection 和 proxyConnection 这两个变量，它们的类型都是
Connection。而 PooledConnection 本身就实现了 JDK 的 InvocationHandler 接口，实现了代理机制。

在日常开发过程中，我们经常会需要对一些来自第三方的对象进程处理，但又没有办法直接对这些对象进行改造（例如本文中的 Connection 对象来自 JDBC
规范，我们只能使用而无法修改器源代码），这时候就可以采用类似 PooledConnection 中所采用的方法来实现一个
proxyConnection，一方面该类添加了自定义的一些操作，另一方面它也符合原有的 JDBC
规范，实现了外部使用上的兼容性。事实上，这种设计方法在很多开源框架中都能找到应用场景，最典型的就是分库分表中间件 ShardingSphere
中提供了一套完全兼容 JDBC 规范但又实现了数据分片和主从分离的解决方案。

### 小结与预告

本文是《剖析通用机制与组件》模块最后一部分内容，介绍了 MyBatis
中连接池的设计理念以及获取数据库访问连接的详细过程。至此，《剖析通用机制与组件》模块的内容全部介绍完毕。

