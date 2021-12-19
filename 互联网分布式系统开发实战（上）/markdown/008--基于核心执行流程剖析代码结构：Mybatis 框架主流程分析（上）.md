任何一个软件系统都存在核心执行流程，对多种框架类软件系统而言，这种核心主流程往往只有一条，所以只要抓住这条主流程，我们就能把握框架的整体代码结构。本文将从这一主题出发探讨如何剖析框架代码。

### 如何抓住主流程对框架进行分层剖析？

如果想要跳出源码阅读的困境，不想浪费大量时间阅读代码细节，不要始终只能捕捉到对系统的片段化认识，我们就必须转换到另一种观点来看待系统。就“框架和结构”这个主题而言，我们进行源码解读的主要目标是明确代码的结构，前面的课程中我们已经学习了从设计原则、架构演进等不同角度出发实现这个目标。事实上，还有一个比较容易理解和把握的方法可以帮忙我们梳理代码结构，这就是代码的执行流程。任何系统的行为都可以认为是流程的组合，看似复杂的代码结构通过分析一般都能梳理出一条贯穿全局的主流程。当我们拿到一个框架的源代码时，不妨先来问自己一个问题：

**_如何抓住主流程对框架进行分层剖析？_**

提出一个问题，我们就要解决一个问题，而解决一个问题的方法通常是继续抛出其他问题。那么，对于一个框架而言，什么才是它的主流程呢？这个问题看上去很难回答，但实时上并非如此。当我们已经着手去分析一个框架的代码结构时，意味着至少我们已经使用过这个框架。基于这个这点，采自上而下的方式，通常都能明确所谓的主流程。举例来说，对于前面提到的
Dubbo 框架，一次 RPC 请求就是它的主流程。而对于 Mybatis 而言，显然，我们应该关注它的一次 SQL 执行过程。

在软件建模领域，可以通过一些工具和手段对代码执行的流程进行可视化。以 UML 为例，我们可以使用活动图（Activity
Diagrams）实现对业务过程、工作流的建模，本质上就是一种流程图。而时序图（Sequence
Diagrams）描述了对象之间消息发送的先后顺序，强调时间顺序。同时，序列图更有效地描述如何分配各个类的职责以及各类具有相应职责的原因。关于活动图和时序图的介绍不是我们的重点，大家可以参考这一领域的相关资料进行学习。但在本篇中，我们将大量使用这些图形化工具进行流程的梳理和表述。

### Mybatis 中的主流程

本节内容我们将回到 Mybatis 这个 ORM 框架，尝试从主流程角度对该框架进行代码结构的解读。首先，我们要明确对于 Mybatis
而言，什么是它的主流程？这个问题前面已经有了答案：一次 SQL 执行过程。

在日常开发过程中，通常都是将 Mybatis 和 Spring
整合在一起使用，而这种整合在方便开发人员使用的同时也屏蔽了部分流程上的细节，一定程度上导致我们对 Mybatis
的执行过程缺少了全局的理解。因此，我们还是直接从 Mybatis 的原始使用方法入手分析 SQL 执行的主流程，代码如下。

    
    
    InputStream is = Resource.getResourceAsStream("mybatis_config.xml");
    
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(is);
    
    SqlSession sqlSession = sqlSessionFactory.openSession();
    
    TestMapper testMapper = sqlSession.getMapper(TestMapper.class);
    
    TestObject testObject = testMapper.getObject("testCode1");
    
    System.out.println(testObject.getCode());
    

以上几行代码是开发人员使用 Mybatis 这个框架的流程，这一流程取决于 Mybatis 为我们开放的接口。作为一个框架，Mybatis
同样屏蔽了底层实现上的细节。但正因为有这些接口的存在，为我们深入框架内核提供了入口。显然，上述代码中我们提供了两个核心的入口，一个是
SqlSession，另一个是 Mapper，我们将从这两个入口开始切入。

### Mybatis SqlSession

本小节讨论一个核心问题，即我们是如何获取 SqlSession？下图通过时序图展示了整个流程。

![](https://images.gitbook.cn/2020-05-25-052742.png)

上图中涉及 Mybatis 中的两个核心对象，即 SqlSessionFactory 和 SqlSession，我们将分别展开讨论。

#### SqlSessionFactory

首先，SqlSessionFactoryBuilder 去读取 mybatis 的配置文件，然后构建出一个
DefaultSqlSessionFactory，源码如下。

    
    
    public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
        try {
          XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
          return build(parser.parse());
        } catch (Exception e) {
          throw ExceptionFactory.wrapException("Error building SqlSession.", e);
        } finally {
          ErrorContext.instance().reset();
          try {
            reader.close();
          } catch (IOException e) {
            // Intentionally ignore. Prefer previous error.
          }
        }
    }
    

我们发现里面有一个 XMLconfigBuilder 对象，该对象就是用来解析 XML 文件的一个 Builder（显然是构建者模式的应用），通过它的
parse()方法解析 mybatis 配置文件，代码如下。

    
    
    public Configuration parse() {
        if (parsed) {
          throw new BuilderException("Each XMLConfigBuilder can only be used once.");
        }
        parsed = true;
        parseConfiguration(parser.evalNode("/configuration"));
        return configuration;
    }
    

顺着代码，我们继续来看 parseConfiguration()方法，该方法解析“/configuration”节点下的子节点：

    
    
     private void parseConfiguration(XNode root) {
        try {
          propertiesElement(root.evalNode("properties"));
          Properties settings = settingsAsProperties(root.evalNode("settings"));
          loadCustomVfs(settings);
          loadCustomLogImpl(settings);
          typeAliasesElement(root.evalNode("typeAliases"));
          pluginElement(root.evalNode("plugins"));
          objectFactoryElement(root.evalNode("objectFactory"));
          objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
          reflectorFactoryElement(root.evalNode("reflectorFactory"));
          settingsElement(settings);            environmentsElement(root.evalNode("environments"));
          databaseIdProviderElement(root.evalNode("databaseIdProvider"));
          typeHandlerElement(root.evalNode("typeHandlers"));
          mapperElement(root.evalNode("mappers"));
        } catch (Exception e) {
          throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
        }
    }
    

然后我们发现 parse()解析完成后返回了一个 Configuration 对象，解析的配置相关信息都会封装在这个对象。再次回到一个
build()方法，可以看到把这个 Configuration 对象作为参数进行传入，并返回了一个 DefaultSqlSessionFactory
对象，代码如下。

    
    
    public SqlSessionFactory build(Configuration config) {
        return new DefaultSqlSessionFactory(config);
    }
    

这个 DefaultSqlSessionFactory 就是 SqlSessionFactory 的实现类，用来生成 SqlSession 对象。

#### SqlSession

当我们获取到 SqlSessionFactory 之后，就可以通过 SqlSessionFactory 去获取 SqlSession
对象。SqlSessionFactory 包含了一系列 openSession()重载方法，这些方法的背后都引用了
openSessionFromDataSource()方法，代码如下。

    
    
    private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
        Transaction tx = null;
        try {
          final Environment environment = configuration.getEnvironment();
          final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
          tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
          final Executor executor = configuration.newExecutor(tx, execType);
          return new DefaultSqlSession(configuration, executor, autoCommit);
        } catch (Exception e) {
          closeTransaction(tx); 
          throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
        } finally {
          ErrorContext.instance().reset();
        }
    }
    

我们可以看到，这段代码首先通过 Confuguration 对象去获取 Mybatis 的 Environment 配置信息，Environment
对象包含了数据源和事务的配置。然后再创建了一个 Executor 对象，该对象是对原生 Statement 的封装，我们在后续
内容中还会具体展开。最后，从头传入 Confuguration 对象和 Executor 对象终于创建了一个 DefaultSqlSession 对象。

### 面试题分析

#### 简要阐述 Mybatis 的一次 SQL 执行的整体过程？

  * 考点分析

Mybatis 作为一个 ORM 框架，本身对 JDBC 规范做了封装，提供了自身的一套执行流程。这道题考查对 Mybatis
的理解，属于宏观类问题，而不是面向细节，相对比较简单。

  * 解题思路

笔者认为这是一道送分题，我们只需要把握 Mybatis 中所涉及到的几个核心步骤即可，包括通过配置文件创建 SqlSessionFactory，再通过
SqlSessionFactory 创建 SqlSession。然后我们可以从 SqlSession 中获取某一个特定的 Mapper，然后 Mapper
内部会通过调用 Executor 完成对原生 JDBC 的执行过程。

  * 本文内容与建议回答

本文给出了 Mybatis
执行上的几个核心步骤，并会结合下一篇文章把每个步骤中的要点进行展开，从而形成一个完成的执行流程。在回答上，通常只要能说清楚这些步骤即可。

### 日常开发技巧

今天的开发技巧同样来自于对 UML
的合理利用，我们需要强调时序图在梳理代码执行流程时的关键作用。事实上，以笔者自身的经历而言，很多时候我们可以省略类图等一些设计上的元素，但不能没有时序图。在设计环节，时序图和数据模型是整个软件从设计到开发阶段过渡的最重要产物。尤其在目前广泛流行的分布式架构或微服务架构中，通常会涉及到多个服务之间的交互和协作，时序图的作用更为突出。强烈建议大家平时都画画时序图，有时候一图胜千言。

### 小结与预告

本篇从“基于核心执行流程剖析代码结构”角度出发探讨了阅读开源框架代码的另一种方法，并基于 Mybatis 这一特定框架讨论了它的核心执行过程。本篇给出了关于
Mybatis 中 Session 的创建过程，下一篇我们将继续讨论后续的 Mapper 和 Executor 组件，并尝试将这些组件整合在一起。

