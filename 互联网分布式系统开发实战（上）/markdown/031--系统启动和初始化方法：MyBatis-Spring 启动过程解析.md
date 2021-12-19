在《基于核心执行流程分析代码结构：如何抓住主流程对框架进行分层剖析？》一篇中，我们已经基于核心主流程对 MyBatis
框架的整体执行过程做了分析，这里面就包括它的启动流程。所以，今天我们讨论 MyBatis 主要关注不是 MyBatis
作为独立框架时的启动过程，而是重点介绍 MyBatis 与 Spring 之间的集成，即 MyBatis-Spring 框架，版本为 1.3.x。

前面几篇文章介绍了 Spring 扩展点及其在 Dubbo 框架中的应用，在此基础之上，我们理解 MyBatis-Spring 框架会简单很多。打开
MyBatis-Spring 代码工程，我们快速寻找实现了 Spring 扩展点的类，这个类并不难找，它就是 SqlSessionFactoryBean
类。

### SqlSessionFactoryBean

很典型的，SqlSessionFactoryBean 实现了 FactoryBean、InitializingBean 和
ApplicationListener 这三个扩展方法，其定义如下所示（列举了部分重要的变量）：

    
    
    public class SqlSessionFactoryBean implements FactoryBean<SqlSessionFactory>, InitializingBean, ApplicationListener<ApplicationEvent> {
          ...
          private Configuration configuration;
          private SqlSessionFactoryBuilder sqlSessionFactoryBuilder = new SqlSessionFactoryBuilder();
          private SqlSessionFactory sqlSessionFactory;
          private String environment = SqlSessionFactoryBean.class.getSimpleName();
        ...
    }
    

我们还是按照既有的思路，先来看看这些扩展点如何完成 Spring 与 MyBatis 框架之前的整合。首先，我们来看 InitializingBean
接口的实现，即如下所示的 afterPropertiesSet 方法（代码做了裁剪）：

    
    
      @Override
      public void afterPropertiesSet() throws Exception {
        ...
        this.sqlSessionFactory = buildSqlSessionFactory();
      }
    

显然，对于 SqlSessionFactoryBean 而言，主要职责就是完成 SqlSessionFactory
的构建，这也是这个类的类名的由来，而完成这个操作的最合适的阶段就是生命周期中的 InitializingBean 阶段。我们来看一下这里的
buildSqlSessionFactory
方法的具体实现过程，这个方法非常长，但代码结构比较简单。我们抛开大量的代码细节，使用如下所示的代码框架来展示这个方法的结构：

    
    
    protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
    
        Configuration configuration;
    
        XMLConfigBuilder xmlConfigBuilder = null;
        if (this.configuration != null) {
          //如果当前的 configuration 不为空，这直接使用该对象
          configuration = this.configuration;
          ...
        } else if (this.configLocation != null) {
          //如果配置文件地址 configLocation 不为空，则通过    XMLConfigBuilder 进行解析并创建 configuration
          xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
          configuration = xmlConfigBuilder.getConfiguration();
        } else {
          //如果以上两种情况都不满足，则创建一个新的 configuration 对象并进行参数赋值
          configuration = new Configuration();
          ...
        }
    
        //设置 objectFactory 等各种 MyBatis 运行时所需的配置信息
        ...
    
        //基于 configuration 通过 SqlSessionFactoryBuilder 构建 SqlSessionFactory
        return this.sqlSessionFactoryBuilder.build(configuration);
    }
    

关于 SqlSessionFactoryBuilder 的内容我们已经在《经典设计模式：创建型模式及其在 MyBatis
中的应用》一文中进行了介绍，我们知道该类会创建一个 DefaultSqlSessionFactory。我们可以通过
DefaultSqlSessionFactory 进而获取 SqlSession 对象的实例。

然后，我们来关注 SqlSessionFactoryBean 实现的 FactoryBean<SqlSessionFactory>
接口，从接口的泛型定义上，我们明白它的 getObject 方法应该返回的是一个 SqlSessionFactory 对象，如下所示：

    
    
      @Override
      public SqlSessionFactory getObject() throws Exception {
        if (this.sqlSessionFactory == null) {
          afterPropertiesSet();
        }
    
        return this.sqlSessionFactory;
      }
    

可以看到，这里的实现过程非常简单，如果目标 sqlSessionFactory 还没有被创建，就直接调用了前面介绍的 afterPropertiesSet
方法完成该对象的创建并返回。

最后，我们需要关注的是如下所示的 onApplicationEvent 方法，这是 ApplicationListener 接口的具体实现，用来对
Spring 中所生成的 ApplicationEvent 进行响应：

    
    
      @Override
      public void onApplicationEvent(ApplicationEvent event) {
        if (failFast && event instanceof ContextRefreshedEvent) {
          this.sqlSessionFactory.getConfiguration().getMappedStatementNames();
        }
      }
    

从场景而言，这个方法的作用是在接收到代表容器刷新的 ContextRefreshedEvent 事件时，重新获取各种
MappedStatement。这里通过调用 Configuration 的 getMappedStatementNames 完成这一操作：

    
    
      public Collection<MappedStatement> getMappedStatements() {
        buildAllStatements();
        return mappedStatements.values();
      }
    

再往下看，就到了 MyBatis 内部的实现细节了，我们不再展开。至此，SqlSessionFactoryBean 中与 Spring
整合的相关内容就介绍到这里。通过 SqlSessionFactoryBean，我们就可以获取 SqlSessionFactory 对象，这是 MyBatis
框架启动过程的主要目标。

### MapperFactoryBean

然后我们在 MyBatis-Spring 工程的 org.mybatis.spring.mapper 包中找到了另一个 FactoryBean，即
MapperFactoryBean。从命名上看，这个类用于生成 MapperFactory，而 MapperFactory 的作用显然是获取
Mapper。MapperFactoryBean 的类定义如下所示：

    
    
    public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
      private Class<T> mapperInterface;
      ...
    }
    

可以看到 MapperFactoryBean 实现了 FactoryBean 接口，它的 getObject 方法如下所示：

    
    
      @Override
      public T getObject() throws Exception {
        return getSqlSession().getMapper(this.mapperInterface);
      }
    

显然，我们通过 SqlSession 的 getMapper 方法获取 Mapper 对象，这个 MyBatis 自身所提供的合作功能。那么这个
SqlSession 是怎么来的呢？我们发现 MapperFactoryBean 在实现了 FactoryBean 接口的同时，还扩展了
SqlSessionDaoSupport 类。

SqlSessionDaoSuppor 是一个抽象类，扩展了 Spring 中的 DaoSupport 抽象类，并提供了如下方法：

    
    
    public abstract class SqlSessionDaoSupport extends DaoSupport {
    
      private SqlSession sqlSession;
    
      private boolean externalSqlSession;
    
      public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
        if (!this.externalSqlSession) {
          this.sqlSession = new SqlSessionTemplate(sqlSessionFactory);
        }
      }
    
      public void setSqlSessionTemplate(SqlSessionTemplate sqlSessionTemplate) {
        this.sqlSession = sqlSessionTemplate;
        this.externalSqlSession = true;
      }
    
      public SqlSession getSqlSession() {
        return this.sqlSession;
      }
    
      @Override
      protected void checkDaoConfig() {
        notNull(this.sqlSession, "Property 'sqlSessionFactory' or 'sqlSessionTemplate' are required");
      }
    }
    

我们看到这里定义了一个 SqlSession 对象用于对外暴露访问入口，但在内部，这个 SqlSession 并不是来自 MyBatis 的
DefaultSqlSession，而是构建了一个同样实现了 SqlSession 接口的 SqlSessionTemplate
类。接下去我们就来一起讨论一下这个 SqlSessionTemplate，这是 MyBatis-Spring 中的核心类。

### SqlSessionTemplate

SqlSessionTemplate 实现了 SqlSession 接口，并提供了如下所示的变量和构造函数：

    
    
    public class SqlSessionTemplate implements SqlSession, DisposableBean {
    
      private final SqlSessionFactory sqlSessionFactory;
      private final ExecutorType executorType;
      private final SqlSession sqlSessionProxy;
      private final PersistenceExceptionTranslator exceptionTranslator;
    
      public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
    
        this.sqlSessionFactory = sqlSessionFactory;
        this.executorType = executorType;
        this.exceptionTranslator = exceptionTranslator;
        this.sqlSessionProxy = (SqlSession) newProxyInstance(
            SqlSessionFactory.class.getClassLoader(),
            new Class[] { SqlSession.class },
            new SqlSessionInterceptor());
      }
      ...
    }
    

要理解 SqlSessionTemplate 这个类的设计思想，我们就要理解 MyBatis 中自带的 DefaultSqlSession
所存在的问题。我们知道 DefaultSqlSession 本身是线程不安全的，所以每当我们想要使用 SqlSession 时，常规做法就是从
SqlSessionFactory 中获取一个新的 SqlSession。但这种做法显然效率低下，造成资源的浪费。更好的实现方法应该是全局存在一个唯一的
SqlSession 实例来完成 DefaultSqlSession 的工作。而 SqlSessionTemplate 就是这个全局唯一的
SqlSession 实例。

明确了 SqlSessionTemplate 这个类的定位之后，那么问题又来了？当我们通过 Web 线程访问同一个
SqlSessionTemplate，也就是同一个 SqlSession 时，它是如何确保线程安全的呢？这就需要我们来分析上面给出
SqlSessionTemplate 类的构造函数。

我们看到这里创建了一个名为 sqlSessionProxy 的 SqlSession，但创建过程使用了 JDK 中经典的动态代理实现方案，即通过
Proxy.newProxyInstance 静态方法诸如一个 InvocationHandler 的实例。在上述代码中，这个实例就是
SqlSessionInterceptor 类。SqlSessionInterceptor 类是 SqlSessionTemplate
中的一个内部类，实现了 InvocationHandler 接口，其 invoke 方法实现如下：

    
    
    private class SqlSessionInterceptor implements InvocationHandler {
    
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    
          //获取 SqlSession 实例，该实例是线程不安全的
          SqlSession sqlSession = getSqlSession(
              SqlSessionTemplate.this.sqlSessionFactory,
              SqlSessionTemplate.this.executorType,
              SqlSessionTemplate.this.exceptionTranslator);
    
          try {
            //调用真实 SqlSession 的方法
            Object result = method.invoke(sqlSession, args);
    
            //判断当前的 sqlSession 是否被 Spring 托管，如果未被 Spring 托管则自动 commit
            if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
              sqlSession.commit(true);
            }
            return result;
          } catch (Throwable t) {
            //省略异常处理
          } finally {
            if (sqlSession != null) {
              //关闭 sqlSession
              closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
            }
          }
        }
      }
    

这里的核心逻辑其实是在 getSqlSession 方法（位于 SqlSessionUtils 类中）中，该方法完成了线程安全性的处理，如下所示（去掉了部分
log 处理的相关逻辑）：

    
    
      public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
    
        //根据 sqlSessionFactory 从当前线程对应的资源 Map 中获取 SqlSessionHolder
        SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
    
        //从 SqlSessionHolder 中获取 sqlSession 实例
        SqlSession session = sessionHolder(executorType, holder);
        if (session != null) {
          return session;
        }
    
        //如果获取不到 sqlSession 实例，则根据 executorType 创建一个新的 sqlSession 
        session = sessionFactory.openSession(executorType);
    
        //将 sessionFactory 和 session 注册到线程安全的资源 Map
        registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);
    
        return session;
      }
    

可以看到这里使用了 Spring 中的一个工具类 TransactionSynchronizationManager，在该类中用于存储传入的
SessionHolder，上述 registerSessionHolder 方法中的核心代码如下所示：

    
    
    holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
    TransactionSynchronizationManager.bindResource(sessionFactory, holder);
    TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
    

而 TransactionSynchronizationManager 中存储 SqlSessionSynchronization 用的是如下所示的
synchronizations 变量：

    
    
    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
        new NamedThreadLocal<Set<TransactionSynchronization>>("Transaction synchronizations");
    

显然，这里用到了熟悉的 ThreadLocal，确保了线程的安全性。

### 面试题分析

#### **简要描述 MyBatis 中 SqlSessionTemplate 的实现原理？**

**考点分析：**

这道题有一定的深度，适合面向比较高阶的开发人员。SqlSessionTemplate 在使用上非常简单，但其设计思想和实现原理非常值得我们学习。事实上，在
Spring 中同样存在一批以 Template 结尾的模板类，这些模板类的实现机制与 SqlSessionTemplate
类同。因此，这道题的考点可以比较灵活，经常会延伸到关于 Spring 中很多内容的考查。

**解题思路：**

要回答这道题，存在几个要点。第一个要点在于我们要明确 SqlSessionTemplate 的设计初衷是因为 MyBatis 中默认的
DefaultSqlSession 是线程不安全的，每次创建新对象会导致效率低下，所以 SqlSessionTemplate
起到的是一个全局唯一访问实例的作用。第二个要点是这里用到了动态代理机制，在代理类中完成了线程安全性的处理。这个处理过程还是有点复杂的，需要结合 Spring
中的一个工具类 TransactionSynchronizationManager 进行说明。最终的线程安全还是通过经典的 ThreadLocal
进行实现。

**本文内容与建议回答：**

本文对 MyBatis 与 Spring 的整合框架 MyBatis-Spring 进行了讲解，除了
FactoryBean、InitializingBean 和 ApplicationListener 这三个扩展点之外，我们重点介绍的是
SqlSessionTemplate 模板类的具体实现方法。这个模板类的实现方法具有一定的代表性，建议进行熟练掌握。在回答这个问题时，我们也需要站在
MyBatis 与 Spring 的整合过程中进行思考和联系。

### 日常开发技巧

在本文中，我们接触到了一种新的线程安全性的处理方式，即通过 Spring 中的工具类
TransactionSynchronizationManager。TransactionSynchronizationManager 提供了
TransactionSynchronization 完成外部对象与 Spring 之间的集成。而在 Spring
中内部，TransactionSynchronizationManager 也是通过对 ThreadLocal 的封装确保并发访问下的线程安全。

### 小结与预告

本文基于 MyBatis 框架讲解了 Spring 中多种扩展点的应用方式，从而完成这 MyBatis-Spring
这个专用于两者之间的集成框架。下一篇中，我们将开始讨论 Spring 家族中目前最流程的 Spring Boot
框架，来看看该框架为我们提供了哪些扩展点机制。

