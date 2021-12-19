从本篇开始，我们将基于设计模式这个主题来剖析开源框架中的代码结构，关注于类层之间的调用和协作关系。

设计模式种类很多，可以说所有的开源框架中或多或少都使用到了一定数量的设计模式。我们无意对所有框架中的设计模式进行详细展开，而是选择目前非常主流的开源 ORM
框架 MyBatis 作为切入。

  * 一方面，MyBatis 中对创建型、结构型和行为型这三大类设计模式都有所介绍，且应用场景都比较典型，对于理解设计模式在其他开源框架中的应用具有参考价值；
  * 另一方面，MyBatis 的代码风格比较简单易懂，适合我们从设计模式出发来梳理代码类层结构，而避免陷入框架本身的复杂设计中。

MyBatis
中包含的设计模式包括建造者模式、工厂模式、单例模式、外观模式、装饰者模式、组合模式、模版方法模式、迭代器模式、策略模式、代理模式、责任链模式等。

其中的责任链模式我们已经在介绍管道—过滤器模式时做过介绍，这里不再重复介绍。而代理模式是 MyBatis 以及 Dubbo
框架的核心模式，其重要性可以作为单独的一个主题进行讲解，在后续的《剖析通用机制与组件》模块中会有专题，这里也不做展开。

其他的几种涉及模式我们将按照类型进行分类讲解，首先让我们来看一下创建型设计模式及其实现。在 MyBatis
中，创建型设计模式主要包括建造者模式、工厂模式和单例模式。

### 建造者模式

建造者模式在 MyBatis
框架中应用非常广泛，是最具代表性的设计模式之一。我们在前面的课程中已经看到了很多与这一设计模式相关的内容，这里来做系统化的阐述。

#### **建造者模式简介**

建造者模式（Builder
Pattern）的目的是将类的内部逻辑与类的生成过程分割开来，从而可以使用一个建造过程生成具有不同的内部逻辑的复杂对象。也就是说，它将一个复杂的构建与其逻辑相分离，使得同样的构建过程可以创建不同的逻辑。建造者模式的组成结构相对比较简单，如下图所示：

![24.01](https://images.gitbook.cn/2020-05-25-052641.png)

上图中的 Director 负责对外直接和客户端系统进行交互。抽象的 Builder
描述具体建造者的公共接口，一般用来定义建造细节的方法，并不涉及具体的对象创建。ConcreteBuilder 实现抽象建造者的公共接口，而最终的
Product 描述一个由一系列组件组成的较为复杂的对象。

上述结构是经典的建设者模式组成结构，在实际应用过程中，根据所需要构建的最终 Product 的复杂程度，不一定完全按照上述结构使用这种模式。以
MyBatis 为例，其使用过程就涉及到在一个 Builder 中嵌套另一个 Builder 的情况，我们来看一下。

#### **建造者模式在 MyBatis 中的应用与实现**

在 MyBatis 中存在一批以 Builder 结尾的类，这些类基本都是建造者模式的具体体现。最典型的就是
SqlSessionFactoryBuilder，该类的目的就是构建 SqlSessionFactory，在 MyBatis 环境的初始化过程中，会调用
XMLConfigBuilder 读取所有的 MybatisMapConfig.xml 和所有的 Mapper.xml 文件，构建 MyBatis
运行的核心对象 Configuration 对象，然后将该 Configuration 对象作为参数构建一个 SqlSessionFactory 对象。而
XMLConfigBuilder 在构建 Configuration 对象时，也会调用 XMLMapperBuilder 用于读取各种 Mapper
文件，而 XMLMapperBuilder 会使用 XMLStatementBuilder 来读取和构建所有的 SQL 语句。

这些过程我们已经在《基于核心执行流程剖析代码结构：MyBatis 框架主流程分析》这个主题中做过详细分析。在这些过程中，有一个相似的特点，就是这些
Builder 会读取文件或者配置，然后做大量的 XpathParser
解析、配置或语法的解析、反射生成对象、存入结果缓存等步骤，这些工作都不适合简单的通过构造函数来提供初始化服务，因此大量采用了建造者模式来解决。接下来，我们就以
SqlSessionFactoryBuilder 为例，介绍建造者模式在 MyBatis 中的应用。

SqlSessionFactoryBuilder 位于 org.apache.ibatis.session 包下，该类内部有很多重载的 build
方法，这些方法返回值都是 SqlSessionFactory，如下图所示。

![24.02](https://images.gitbook.cn/2020-05-25-052642.png)

上图中虽然方法名都是 build，但可以分成三组，其中前四个是一组，第五个到第八个是一组，最后一个是一组。而前面的两组方法最终都是调用最后一个 build
方法，即：

    
    
    public SqlSessionFactory build(Configuration config) {
        return new DefaultSqlSessionFactory(config);
    }
    

显然这里通过传入一个 Configuration 对象构建出了 DefaultSqlSessionFactory，这是 MyBatis 默认的
SqlSessionFactory。这么一来，接下来要搞清楚的就是这个 Configuration 是怎么来的，看一下如下所示的 build
方法，该方法为前面的各种重载方法提供了基础：

    
    
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
    

可以看到该方法的核心内容就是通过 XMLConfigBuilder 类构建出 Configuration 对象，该类中最重要的就是 parse
方法，代码如下：

    
    
    public Configuration parse() {
        if (parsed) {
          throw new BuilderException("Each XMLConfigBuilder can only be used once.");
        }
        parsed = true;
        parseConfiguration(parser.evalNode("/configuration"));
        return configuration;
    }
    
    public class XMLConfigBuilder extends BaseBuilder {
    
        // 省略参数定义和构造函数
    
        private void parseConfiguration(XNode root) {
            try {
              //issue #117 read properties first
              propertiesElement(root.evalNode("properties"));
              Properties settings = settingsAsProperties(root.evalNode("settings"));
              loadCustomVfs(settings);
              loadCustomLogImpl(settings);
              typeAliasesElement(root.evalNode("typeAliases"));
              pluginElement(root.evalNode("plugins"));
              objectFactoryElement(root.evalNode("objectFactory"));
              objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
              reflectorFactoryElement(root.evalNode("reflectorFactory"));
              settingsElement(settings);
              // read it after objectFactory and objectWrapperFactory issue #631
              environmentsElement(root.evalNode("environments"));
              databaseIdProviderElement(root.evalNode("databaseIdProvider"));
              typeHandlerElement(root.evalNode("typeHandlers"));
              mapperElement(root.evalNode("mappers"));
            } catch (Exception e) {
              throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
            }
          }
    
        // 省略其他方法
    }
    

显然，从命名上，XMLConfigBuilder 也是一个 Builder，该 Builder 中的 parse 方法最终要返回一个
Configuration 对象，构建 Configuration 对象的建造过程都在 parseConfiguration 方法中，这也就是
MyBatis 解析 XML 配置文件来构建 Configuration 对象的主要过程。

作为总结，从建造者设计模式上讲，XMLConfigBuilder 是建造者 SqlSessionFactoryBuilder
中的建造者，它们所构建的复杂对象分别是 SqlSessionFactory 和 Configuration，如下图所示：

![24.03](https://images.gitbook.cn/2020-05-25-52643.png)

在 MyBatis 中，除了本文中提到的 SqlSessionFactoryBuilder 和 XMLConfigBuilder，还有
XMLMapperBuilder、XMLStatementBuilder、CacheBuilder 等一大批以 Builder
结尾的建造者类，读者可以自行进行阅读和理解。

### 工厂模式

在前面介绍的建造者设计模式中，我们通过 SqlSessionFactoryBuilder 最终构建了一个
DefaultSqlSessionFactory。从命名上，我们也可以知道 DefaultSqlSessionFactory
是一个工厂类，本小节就来介绍相关的工厂模式。

#### **工厂模式简介**

一个工厂类处于对负责对象实例化的中心位置上，它知道每一个对象，它决定哪一个对象应当被实例化。这个模式的优点是允许客户端相对独立于对象创建的过程，并且在系统引入新对象的时候无需修改客户端，也就是说，它在某种程度上支持开闭原则（Open
Close Principle，OCP）。另一方面，工厂方法模式中，我们会首先定义一个创建目标对象的工厂接口，将实际创建工作推迟到实现类中。

在软件系统中，经常面临着具体某个对象的创建工作，但这个对象的实现逻辑由于需求变更经常面临着剧烈的变化，但是它却拥有比较稳定的接口，这时候就可以考虑使用工厂方法来实现对象接口与实现类之间的分离。工厂模式的示意图如下所示：

![24.04](https://images.gitbook.cn/2020-05-25-052644.png)

显然，该模式在对象创建管理方式最为简单，因为其仅仅需要的对不同种类对象的创建过程添加一层薄薄的封装。该模式通过向工厂传递类型来指定要创建的对象。

#### **工厂模式在 MyBatis 中的应用与实现**

在 MyBatis 中存在一系列以 Factory 结尾的工厂类，其中最典型的就是 DefaultSqlSessionFactory
类，让我们回到该类。我们再一次从类的命名上可以看出该工厂的作用是返回一个 SqlSession 对象。DefaultSqlSessionFactory
实现了 SqlSessionFactory 接口，该接口定义如下：

    
    
    public interface SqlSessionFactory {
    
      SqlSession openSession();
      SqlSession openSession(boolean autoCommit);
      SqlSession openSession(Connection connection);
      SqlSession openSession(TransactionIsolationLevel level);
      SqlSession openSession(ExecutorType execType);
      SqlSession openSession(ExecutorType execType, boolean autoCommit);
      SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level);
      SqlSession openSession(ExecutorType execType, Connection connection);
      Configuration getConfiguration();
    }
    

我们看到该接口定义了一批 openSession 重载方法，该方法分别支持 autoCommit、Executor、Transaction
等参数的输入来构建核心的 SqlSession 对象。而在它的实现类 DefaultSqlSessionFactory 中，我们以如下所示的
openSessionFromDataSource 方法为例理解获取 SqlSession 的过程。

    
    
    private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
        Transaction tx = null;
        try {
          final Environment environment = configuration.getEnvironment();
          final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
          tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
          final Executor executor = configuration.newExecutor(tx, execType);
          return new DefaultSqlSession(configuration, executor, autoCommit);
        } catch (Exception e) {
          closeTransaction(tx); // may have fetched a connection so lets call close()
          throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
        } finally {
          ErrorContext.instance().reset();
        }
    }
    

可以看到该方法返回一个 DefaultSqlSession 对象，而该对象的构建需要配置对象 configuration、执行器对象 Executor
以及是否为自动提交的 autoCommit 标志位。注意到这里我们又发现了一个工厂接口 TransactionFactory，它的定义如下所示：

    
    
    public interface TransactionFactory {
    
      default void setProperties(Properties props) {
        // NOP
      }
    
      Transaction newTransaction(Connection conn);
    
      Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit);
    }
    

TransactionFactory 有两个实现类 JdbcTransactionFactory 和
ManagedTransactionFactory，分别对应于我们通过 `<transactionManager type="JDBC" />` 配置项配置
MyBatis 时的使用的“JDBC”和“MANAGED”这两种策略。

这里以 JdbcTransactionFactory 为例，可以看到它的实现如下所示，返回的是
JdbcTransaction。ManagedTransactionFactory 的实现大家可以自行参阅 MyBatis 源码。

    
    
    public class JdbcTransactionFactory implements TransactionFactory {
    
      @Override
      public Transaction newTransaction(Connection conn) {
        return new JdbcTransaction(conn);
      }
    
      @Override
      public Transaction newTransaction(DataSource ds, TransactionIsolationLevel level, boolean autoCommit) {
        return new JdbcTransaction(ds, level, autoCommit);
      }
    }
    

综上，对于工厂模式而言，上述代码分别介绍了两种场景，一种是通过 SqlSessionFactory 返回 SqlSession，另一种是通过
TransactionFactory 返回
Transaction。这两种场景所使用的工厂模式都是属于工厂方法，即通过定义一个创建目标对象的工厂接口，然后将实际创建工作推迟到实现类中。

### 单例模式

单例（Singleton）模式在很多开源框架中都得到了应用，MyBatis 也不例外。针对单例模式，在 MyBatis 中有一个地方用到该模式，即
ErrorContext。

#### **单例模式简介**

在单例模式中，一个类仅有一个实例，并提供一个访问它的全局访问点。也就是说，单件模式的要点是某个类只能有一个实例，它必须自行创建这个实例，也必须自行向整个系统提供这个实例。单例模式的结构示意图如下所示：

![24.05](https://images.gitbook.cn/2020-05-25-052647.png)

#### **单例模式在 MyBatis 中的应用与实现**

在 MyBatis 中，ErrorContext 是用在每个线程范围内的单例，用于记录该线程的执行环境错误信息。

以 ErrorContext 为例，其单例模式的实现方法是用 private 修饰符修饰构造函数，然后设置一个 static 的局部 instance
变量和一个获取 instance 变量的方法，。然后在获取实例的方法中，先判断是否为空如果是的话就先创建，然后返回构造好的对象。ErrorContext
类位于 org.apache.ibatis.executor 包中，其构造函数如下所示：

    
    
    private static final ThreadLocal<ErrorContext> LOCAL = new ThreadLocal<>();
    
    public static ErrorContext instance() {
        ErrorContext context = LOCAL.get();
        if (context == null) {
          context = new ErrorContext();
          LOCAL.set(context);
        }
        return context;
    }
    

注意到，这里的 LOCAL 对象是一个 ThreadLocal<ErrorContext> 对象。ErrorContext 类通过 ThreadLocal
管理单例对象，一个线程对应一个 ErrorContext 对象，从而确保线程安全。

### 面试题分析

#### **1\. 描述一下 MyBatis 中 SqlSessionFactory 的构建过程，这里用到了哪种设计模式？**

**考点分析：**

这个问题有些导向性，如果不熟悉这块内容，看到 SqlSessionFactory 容易让人联想到工厂模式。但该问题的考点在于描述
SqlSessionFactory 自身的构建过程，所以用到的是建造者模式。

**解题思路：**

MyBatis 中存在一批以"Builder"结尾的建造者类，SqlSessionFactoryBuilder 就是其中之一，该类负责创建
SqlSessionFactory。在回答该问题时，需要简要把握 SqlSessionFactory 本身的建造过程。而对于 MyBatis
而言，这一场景下建造者模式的特殊之处在于在构建 SqlSessionFactory 过程中还用到了另一个建造者类 XMLConfigBuilder，相当于
XMLConfigBuilder 是 SqlSessionFactoryBuilder 的建造者。

**本文内容与建议回答：**

本文内容上重点把 SqlSessionFactoryBuilder 中 build 方法进行了展开，该方法并不复杂，只是其中涉及到另一个
XMLConfigBuilder 类。而 XMLConfigBuilder 通过对 XML 文件的解析完成 Configuration 类的创建，进而完成
SqlSessionFactory 的创建。整个流程还是比较清晰的，需要对源代码有一定的了解。

#### **2\. 请说出 MyBatis 中使用工厂模式的一处具体场景？**

**考点分析：**

这个题目的考点是很明确的，关注的是工厂模式在 MyBatis 中的应用。MyBatis 中关于工厂模式的应用场景不止一处，因此该问题也比较开放，不难回答。

**解题思路：**

对于这类问题，回答思路同样比较明确，基本也属于送分题。可以先阐述一下工厂模式这一特定设计模式的基本结构和特点，然后引出其在 MyBatis
中的应用场景。然后从实现过程上给出源码级别的细节即可。

**本文内容与建议回答：**

本文给出了工厂模式在 MyBatis 中两个应用场景，即 SqlSessionFactory 返回 SqlSession，以及通过
TransactionFactory 返回
Transaction。实现方式上也比较简单明确，都是通过定义一个创建目标对象的工厂接口，然后将实际创建工作推迟到实现类中。

#### **3\. 单例模式的基本实现方法是什么？MyBatis 中用到单例模式了吗？**

**考点分析：**

这道题考查 MyBatis 中用到的单例模式，单例模式的实现过程比较固化，但在 MyBatis 中应用的不多，所以回答起来有一点难度。

**解题思路：**

就单例模式而言，基本的实现方式就是用 private 修饰符修饰构造函数，然后设置一个 static 的局部变量，并提供一个获取该变量的方法。对于
ErrorContext 而言，很多同学可能不一定知道，需要提升对 MyBatis 的了解程度。

**本文内容与建议回答：**

本文中给出了 ErrorContext 类中构建单例模式的实现方法，该方法与常见的实现过程有所不同。MyBatis 在这里使用了一个 ThreadLocal
来确保线程安全性，而不是使用一个 static 类的静态变量来保存所创建的单例实例。

### 日常开发技巧

对于创建型设计模式而言，日常开发过程我们用的最多的应该是各种工厂模式。工厂模式的使用方法简单明了，相较同属于创建者类型的其他设计模式，在代码中引入工厂模式的成本和难度也多不高，但却能够为代码结构带来不少的提升。但凡涉及到一些核心对象的创建，都可以先考虑是否可以用工厂模式来进行封装。

### 小结与预告

本文是经典设计模式系列的第一篇，我们以 MyBatis 这款流行的 ORM
框架为例，讨论了创建型设计模式在该框架中的应用。下一篇我们将关注另一类设计模式的使用场景和方式，即结构型设计模式。

