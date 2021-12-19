本文继续讨论设计模式在 MyBatis 中的应用，主要涉及到外观模式以及装饰器模式这两大代表性的结构型设计模式。

### 外观模式

在 MyBatis 中，外观模式应用也比较多，但并不像建造者模式或工厂模式那样从类的命名上就可以直接进行判断，而是需要做一些挖掘。

#### **外观模式简介**

外观（Facade）模式的意图可以描述为为子系统中的一组接口提供一个一致的界面。外观模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。该模式的示意图如下图所示：

![25.01](https://images.gitbook.cn/2020-05-25-052656.png)

从作用上讲，外观模式与微服务架构中的 API 网关比较类似。通常，各个子系统或自服务的 API 粒度与客户端的要求不一定完全匹配，各个子系统一般提供细粒度的
API，这意味着客户端通常需要与多个子系统进行交互。更为重要的是，外观模式能够起到客户端与微服务之间的隔离作用，随着业务需求的变化和时间的演进，外观背后的各个子系统的划分和实现可能要做相应的调整和升级，这种调整和升级需要实现对客户端透明。

外观模式也常被称为门面模式，在本课程中，外观和门面这两种叫法都会用到，没有区别。从类层结构而言，一个外观模式的常见组成方式如下所示：

![25.02](https://images.gitbook.cn/2020-05-25-052657.png)

上图中，外观类 Facade 分别对接了背后的三个子系统，从而为 Client 提供统一的访问入口。

#### **外观模式在 MyBatis 中的应用与实现**

我们已经在上一篇《经典设计模式：创建型模式及其在 MyBatis 中的应用》中看到了 DefaultSqlSession 类，该类是 SqlSession
接口的默认实现。我们也已经不止一次看到 Defaultxxx 这一命名规则（如用于生产 SqlSession 的工厂类
DefaultSqlSessionFactory）。在 MyBatis 中，该命名规则适用于寻找某些其它接口的默认实现。SqlSession
的作用实际上就是封装了底层的
API，使得客户类不需要关注这些底层细节，转而通过外观类所提供的接口，这样做的好处就是避免底层那些功能强大但层次较低的接口使我们的调用更复杂。

**1\. SqlSession**

我们已经知道 DefaultSqlSession 是一个外观类，那么从外观模式角度讲，学习该类的最直接的方法就是查看它的变量。让我们来翻看
DefaultSqlSession 的代码，如下所示：

    
    
    public class DefaultSqlSession implements SqlSession {
    
          private final Configuration configuration;
        private final Executor executor;
    
        // 省略其他代码
    }
    

从上述代码中，我们看到 DefaultSqlSession 中使用了 Configuration 和 Executor，这两个类的角色就相当于充当了它的底层
API。

我们先来看 Configuration，Configuration 中保存着 MyBatis 的各种环境信息、数据源、插件、解析后的 sqlMap
等对象，是 MyBatis 中非常核心的一个配置信息类。而在 DefaultSqlSession 中，Configuration 类的主要作用是获取
MappedStatement 并提供给 Executor 进行 SQL 的执行。同时，Configuration 类还能通过类型直接获取 Mapper
对象。相关代码如下所示：

    
    
    MappedStatement ms = configuration.getMappedStatement(statement);
    
    configuration.getMapper(type, this);
    

另一方面，Executor 是 MyBatis 为了封装语句执行、调用结果集解析的核心接口。DefaultSqlSession
为我们屏蔽了从配置信息中获取映射的 SQL 语句封装类，再交给 Executor 执行，Executor
会根据传入的参数进行查询（select）、更新（update）和获取游标（selectCursor）等操作。以代表查询的 select
方法为例，如下代码展示了使用 Executor 最终获得结果集的过程。

    
    
    @Override
    public void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
        try {
          MappedStatement ms = configuration.getMappedStatement(statement);
          executor.query(ms, wrapCollection(parameter), rowBounds, handler);
        } catch (Exception e) {
          throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
        } finally {
          ErrorContext.instance().reset();
        }
    }
    

通过以上的外观模式实现方式，我们使用 MyBatis 时就不需要获得底层的 Configuration 和 Executor 对象，而只需要和
SqlSession 这个门面打交道。

**2\. Configuration**

我们再来看 Configuration 类，实际上 Configuration 本身也是一个非常典型的门面类。Configuration
类的代码比较复杂，围绕这个类我们会展开很多讨论。今天我们关注于它在屏蔽底层接口上所起到的作用。事实上，Configuration
类主要对一些创建对象的操作进行封装，这些对象包括 ParameterHandler、ResultSetHandler、StatementHandler 和
Executor 等。这里以创建 Executor 为例，来看一下 Configuration 类中的 newExecutor 方法，如下所示。

    
    
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
    

上述 newExecutor 方法封装了如何创建 Executor 的过程，可以看到 Executor
的创建过程比较复杂，需要根据传入的类型选择创建具体哪种 Executor。Configuration
类通过该方法屏蔽了这一复杂过程。作为回顾，如果你还记得在《架构模式之管道—过滤器模式：管道—过滤器模式在 MyBatis
中的应用》这篇文章中的内容，就应该能够回想起上述代码，因为我们在这里用到了拦截器链 InterceptorChain。

### 装饰者模式

装饰器模式的本质是可以在不使用创造更多子类的情况下，将对象的功能加以扩展。MyBatis 中的缓存实现过程及时这一模式的典型应用。

#### **装饰器模式简介**

装饰器模式允许向一个现有的对象添加新的功能，同时又不改变其结构，这种模式创建了一个装饰类，用来包装原有的类，并在保持类方法签名完整性的前提下，提供了额外的功能。作为一种结构型设计模式，装饰器模式的目的是为了动态地给一个对象添加一些额外的职责，相比生成子类这种方式可以更为灵活。

从使用时机上讲，装饰器模式可以在不想增加很多子类的情况下扩展类，所以可以被认为是继承的一个替代模式。具体做法就是将想要的的具体功能按职责进行划分并继承装饰者模式。这样装饰类和被装饰类可以独立发展，不会相互耦合。装饰器模式的一个示例类图如下所示。

![25.03](https://images.gitbook.cn/2020-05-25-052700.png)

接下来，我们将简单的 Java 代码来实现上图中的类层结构。首先我们定义基础的 Shape 接口，如下所示：

    
    
    public interface Shape {
        void draw();
    }
    

Shape 接口有两个实现类，分别是 Circle 和 Rectangle，如下所示：

    
    
    public class Circle implements Shape {
    
        @Override
        public void draw() {
            System.out.println("Shape: Circle");
        }
    }
    
    public class Rectangle implements Shape {
    
        @Override
        public void draw() {
            System.out.println("Shape: Rectangle");
        }
    }
    

有了基本类层结构之后，我们就来定义装饰器类 ShapeDecorator，如下所示：

    
    
    public abstract class ShapeDecorator implements Shape {
        protected Shape decoratedShape;
    
        public ShapeDecorator(Shape decoratedShape) {
            this.decoratedShape = decoratedShape;
        }
    
        public void draw() {
            decoratedShape.draw();
        }
    }
    

我们看到 ShapeDecorator 是一个抽象类，在实现了 Shape 接口的同时又在内部包含了对 Shape
的引用，通过这个引用完成对接口方法的实现。这种设计就是装饰器模式的基本实现策略。

然后我们来看 ShapeDecorator 的一个实现类 RedShapeDecorator，该类添加了额外功能，即提供了包装实现：

    
    
    public class RedShapeDecorator extends ShapeDecorator {
    
        public RedShapeDecorator(Shape decoratedShape) {
            super(decoratedShape);
        }
    
        @Override
        public void draw() {
            decoratedShape.draw();
    
            //添加额外功能
            setRedBorder(decoratedShape);
        }
    
        private void setRedBorder(Shape decoratedShape) {
            System.out.println("Border Color: Red");
        }
    }
    

而在具体使用上，我们发现这个包装类和其他类实际上没有什么区别，即只要是使用 Shape 接口的地方都可以使用这个包装类，示例代码如下所示：

    
    
    Shape circle = new Circle();
    Shape redCircle = new RedShapeDecorator(new Circle());
    Shape redRectangle = new RedShapeDecorator(new Rectangle());    
    
    circle.draw();
    redCircle.draw();
    redRectangle.draw();
    

运行上述代码，得到如下所示的结果：

    
    
    Shape: Circle
    Shape: Circle
    Border Color: Red
    Shape: Rectangle
    Border Color: Red
    

### 面试题分析

#### **1\. 简要描述外观模式在 MyBatis 中的应用？**

**考点分析**

从概念上讲，外观模式本身是一种比较明确的模式，但从实现方式上讲，外观模式确是所有常见的模式中最为难以识别的一种模式，原因在于其没有固定的模式结构。所以这道题看似简单，但却非常考验面试者对
MyBatis 中源代码的理解程度。

**解题思路**

前面提到外观模式比较难以识别，这里有一个比较通用的原则，即直接暴露给开发者的常见类往往就是外观类，这点跟外观类的作用是一致的。我们知道外观类的定位就是封装内部的复杂逻辑，从而为开发人员提供统一且简洁的外部入口。从这个角度讲，MyBatis
中实际上大量应用了外观模式，那些被开发人员经常使用的核心类都属于外观模式的具体实现，最典型的就是
SqlSession。解题思路上，我们一般也是围绕上述要点进行展开，从外观类的特点和识别方式入手引出 MyBatis 中一些核心类并进行展开。

**本文内容与建议回答**

本文给出了 MyBatis 中两个具有代表性的外观类实现，即 SqlSession 和 Configuration 类。其中 SqlSession
内部包含了 Configuration 和 Executor 这两个底层 API，而 Configuration 类则对 Executor
的创建过程进行了封装。梳理清楚这两个外观类之间的关系就足以对应这道面试题。

### 日常开发技巧

在复杂系统开发过程中，外观模式是一种非常值得尝试的设计模式。但由于它没有固化的结构，所以应用起来有一定的难度。对于涉及到多个复杂对象之间的交互过程、或者涉及到两个分属于不同边界的系统或模块之间的交互、亦或者需要统一对外暴露访问入口时，使用外观模式是确保代码结构稳定和可扩展的一大技巧，鼓励大家多用。

### 小结与预告

本文介绍了 MyBatis
中所采用的结构型设计模式，主要包括外观模式的应用场景和实现方式，并给出了关于装饰器模式的一个简单示例。下一篇中，我们将继续介绍装饰器模式以及组合模式在
MyBatis 中的应用。

