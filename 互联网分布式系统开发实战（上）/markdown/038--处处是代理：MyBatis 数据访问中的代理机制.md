在 MyBatis 中，应用动态代理的场景实际上非常多，本课程无意对所有的场景都一一展开。这里举两个最典型的应用场景，一个是 Mapper
层的动态代理，用于根据 Mapper 层接口获取 SQL 执行结果；另一个是 ResultLoader
层动态代理，用于提供延迟加载（LazyLoad）技术对查询结果集进行处理。

### Mapper 和动态代理

在开始介绍 MyBatis 中的代理机制之前，我们先来回顾一下《基于核心执行流程剖析代码结构：MyBatis 框架主流程分析》中介绍的内容，我们看到
MyBatis 是通过 MapperProxy 动态代理 Mapper，如下所示：

![cL79P0](https://images.gitbook.cn/2020-05-25-052604.png)

使用过 MyBatis 的同学都应该接触过这样一种操作，我们只需要定义 Mapper 层的接口而不需要对其进行具体的实现，该接口却能够正常调用完成 SQL
执行等一系列操作，这是怎么做到的呢？让我们下面梳理一下整个调用流程。

在使用 MyBatis 时，业务层代码中调用各种 Mapper 的一般做法是通过 SqlSession 这个外观类，例如：

    
    
    TestMapper testMapper = sqlSession.getMapper(TestMapper.class);
    

我们知道 DefaultSqlSession 的 getMapper 方法，如下所示：

    
    
    @Override
    public <T> T getMapper(Class<T> type) {
        return configuration.getMapper(type, this);
    }
    

作为外观类，DefaultSqlSession 把这一操作转移给了 Configuration 对象，该对象中的 getMapper 方法如下所示：

    
    
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        return mapperRegistry.getMapper(type, sqlSession);
    }
    

让我们来到 MapperRegistry 类，该类位于 org.apache.ibatis.binding 包中，该包中包含了与 Mapper
相关的代理机制工具类，包括 MapperProxyFactory、MapperProxy、MapperMethod 以及
MapperRegistry。让我们从 MapperRegistry 入手梳理整个调用链路，MapperRegistry 类中的 getMapper
方法如下所示：

    
    
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
    

MapperRegistry 类中持有一个 knownMappers 对象，该对象是一个 Map，保存 Mapper 类型与
MapperProxyFactory 之间的一种对应关系。上述 getMapper 方法根据传入 Mapper 的类型从 knownMappers
对象中获取一个 MapperProxyFactory，然后由 MapperProxyFactory 负责创建真正的 Mapper
实例。请注意，mapperProxyFactory.newInstance 方法的传入参数是一个 SqlSession，如下所示。

    
    
    public T newInstance(SqlSession sqlSession) {
        final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
    }
    

这里引出另一个核心类 MapperProxy，从命名上看，我们可以猜想该类是一个代理类，因此势必使用了前面介绍的某种动态代理技术。可以先看一下
MapperProxy 类的签名，如下所示：

    
    
    public class MapperProxy<T> implements InvocationHandler, Serializable {
    }
    

显然，这里用到的是 JDK 自带的基于 InvocationHandler 的动态代理实现方案，因此，在 MapperProxy 类中同样肯定存在一个
invoke 方法，如下所示：

    
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
          if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
          } else if (method.isDefault()) {
            return invokeDefaultMethod(proxy, method, args);
          }
        } catch (Throwable t) {
          throw ExceptionUtil.unwrapThrowable(t);
        }
    
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        return mapperMethod.execute(sqlSession, args);
    }
    

可以看到，对于要执行的 SQL 语句而言，会把这部分工作交给 MapperMethod 处理。在 MapperMethod 的 execute
方法中，我们传入了 SqlSession 以及相关的参数，而在 execute 方法内部，根据 SQL
命令的不同类型（insert、update、delete、select）和返回类型分别调用 SqlSession 的不同方法。以
executeForMany 方法为例：

    
    
    private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
        List<E> result;
        Object param = method.convertArgsToSqlCommandParam(args);
        if (method.hasRowBounds()) {
          RowBounds rowBounds = method.extractRowBounds(args);
          result = sqlSession.selectList(command.getName(), param, rowBounds);
        } else {
          result = sqlSession.selectList(command.getName(), param);
        }
        if (!method.getReturnType().isAssignableFrom(result.getClass())) {
          if (method.getReturnType().isArray()) {
            return convertToArray(result);
          } else {
            return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
          }
        }
        return result;
    }
    

可以看到，这里调用了 sqlSession.selectList 方法返回多个数据结果，然后对执行的返回值做适配。

目前为止，我们看到了 MapperProxy 类实现 InvocationHandler 接口，但还没有看到 Proxy.newProxyInstance
方法的调用，该方法实际上也同样位于 MapperProxyFactory 类中，该类还存在一个 newInstance 重载方法，通过传入
mapperProxy 的代理对象最终完成代理方法的执行，如下所示：

    
    
    protected T newInstance(MapperProxy<T> mapperProxy) {
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
    }
    

作为总结，我们梳理了 MyBatis 中 Mappe 层动态代理相关类的类层结构，如下所示：

![ee9HWm](https://images.gitbook.cn/2020-05-25-052605.png)

总体而言，基于代理机制，MyBatis 中 Mapper
层的接口调用过程还是比较简单明确的，这里采用的实现方案也比较经典，相关的类层结构设计也可以用于日常开发工作的参考。

### 延迟加载和动态代理

所谓延迟加载（LazyLoad，有时也称为懒加载），是指在进行表的关联查询时，按照设置的延迟规则推迟对关联对象的查询。其目的是为了避免一些无谓的性能开销，也就是在真正需要数据时才触发数据加载操作。主流的
MyBatis、Hibernate 等 ORM 框架中都包含了对延迟加载的支持。

如果大家使用过 MyBatis 的延迟加载机制，就应该知道可以在 resultMap 节点中使用 `fetchType="lazy"`
配置项来指定该嵌套查询是否启动延迟加载。MyBatis 通过 ResultLoaderMap、ResultLoader、ResultSetHandler
等一系列类来完成这个过程，整个过程比较复杂，篇幅有限，我们无意一一展开。我们直接跳到位于
org.apache.ibatis.executor.resultset 包中的 DefaultResultSetHandler 类的的
createResultObject 方法，这里是延迟加载过程中应用动态代理机制的入口。我们在该方法中找到了以下代码片段：

    
    
    if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
         resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
         break;
    }
    

显然，从 if 语句的判断逻辑可以看到，当配置信息中具备嵌套的 QueryId（即需要执行的嵌套查询）以及 isLazy 方法返回为 true 时，我们由
ProxyFactory 来处理延迟加载问题。作为 MyBatis 中实现动态加载机制的另一个典型场景，我们来看看这里的
ProxyFactory，该接口定义如下：

    
    
    public interface ProxyFactory {
    
      default void setProperties(Properties properties) {
        // NOP
      }
    
      Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs);
    }
    

这里的 createProxy 方法就是实现延迟加载逻辑的核心方法，也是我们重点要分析的对象。针对 ProxyFactory 接口，MyBatis
提供了两个实现类，即 CglibProxyFactory 和 JavassistProxyFactory，它们分别基于 Javassist 和 CGLib。

以 CGLib 为例，在 CglibProxyFactory 类中，我们看到了 createProxy
方法，其中核心代码如下所示。可以看到，这里初始化了一个 Enhancer，并调用 Enhancer.create
方法生成对象，这部分的处理方式与我们在前面介绍 CGLib 的用法时非常类似。

    
    
    static Object crateProxy(Class<?> type, Callback callback, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
        Enhancer enhancer = new Enhancer();
        enhancer.setCallback(callback);
        enhancer.setSuperclass(type);
    
        // 省略中间代码
    
        Object enhanced;
        if (constructorArgTypes.isEmpty()) {
          enhanced = enhancer.create();
        } else {
          Class<?>[] typesArray = constructorArgTypes.toArray(new Class[constructorArgTypes.size()]);
          Object[] valuesArray = constructorArgs.toArray(new Object[constructorArgs.size()]);
          enhanced = enhancer.create(typesArray, valuesArray);
        }
        return enhanced;
    }
    

同时，我们在 CglibProxyFactory 类中还发现了一个私有类 EnhancedResultObjectProxyImpl 类，该类实现了
CGLib 的 MethodInterceptor 接口。我们知道这个接口是 CGLib 拦截目标对象方法的入口，对目标对象方法的调用都会通过此接口的
intercept 方法进行代理。对于延迟加载而言，MyBatis 就通过这一方法实现了相关处理逻辑。我们来看一下这个重要的 intercept
方法，其中的核心代码片段如下所示：

    
    
    if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {
        if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
             lazyLoader.loadAll();
        } else if (PropertyNamer.isSetter(methodName)) {
             final String property = PropertyNamer.methodToProperty(methodName);
             lazyLoader.remove(property);
        } else if (PropertyNamer.isGetter(methodName)) {
             final String property = PropertyNamer.methodToProperty(methodName);
             if (lazyLoader.hasLoader(property)) {
                       lazyLoader.load(property);
              }
         }
    }
    

这里的 lazyLoader 就是一个 ResultLoaderMap 对象，用来表示需要延迟加载的属性集，本质上就是一个
HashMap。上述代码的处理流程取决于传入的方法名，分成了三个代码执行分支。如果 lazyLoader 不为空，标明还存在需要延迟加载的数据，而同时
aggressiveLazyLoading 为 true 或者执行的是能够触发延迟加载的方法（默认的包括 toString、hashcode、clone 和
equals），那么就对延迟加载的属性进行全部加载；如果传入的是 set 方法，说明已经进行赋值，那么该数据属性就不再需要延迟加载；而如果是 get
方法且在 lazyLoader 中存在该属性，即该属性属于延迟加载属性，那就执行延迟加载。

### 面试题分析

#### **在 MyBatis 中，为什么我们只需要定义 Mapper 层的接口而不需要对其进行具体的实现就能完成正常的 SQL 执行过程？**

**考点分析：**

在笔者眼中，这是一道很经典的面试题。经典之处在于一般的开发人员很难通过题目中的描述明确问题的考点。实际上，该题考查的还是代理机制。而从问题的问法上是基于
MyBatis 这个框架，但问题本身是跟 MyBatis 没有关系的，关注的是“没有具体实现的接口如何完成方法调用”这个本质问题。

**解题思路：**

我们要理理思路来慢慢解开这道题的回答内容。首先，我们明确通过 MyBatis 可以提供一系列的自定义 Mapper
接口，我们使用这些接口定义就可以执行数据库的查询、更新等操作。这是框架应用上的具体做法，相信所有用过 MyBatis
的同学都能明确这一点。然后，我们再次明确，通过接口定义就能提供方法调用的基本原理就是对这些接口进行了动态代理。只要明确了这一点，就可以把动态代理的相关内容嫁接到这个问题上，从而完成该问题的回答。

**本文内容与建议回答：**

本文给出了 Mapper 和动态代理在实现上的详细细节，在 MyBatis 内部实现了
MapperProxyFactory、MapperProxy、MapperMethod 以及 MapperRegistry
等一系列与动态代理相关的工具类，而其中的 MapperProxy 就实现了 JDK 中的 InvocationHandler
接口并通过反射机制完成了具体方法的调用。针对这些工具类，我们还给出了一张类图方便大家进行记忆。

#### **简要描述 MyBatis 中延迟加载的实现原理？**

**考点分析：**

针对 MyBatis、Hibernate 等 ORM 框架，这是一道常见的面试题。延迟加载是 ORM 框架的一项典型功能，其实现也是依赖于动态代理机制。

**解题思路：**

对于该问题，我们在回答上首先需要给出延迟加载的基本概念以及使用方法，在 MyBatis
中可以通过一个标志位来控制是否需要进行延迟加载。而针对实现原理，MyBatis 也抽象了一个 ProxyFactory 接口，并基于 Javassist 和
CGLib 分别提供了 CglibProxyFactory 和 JavassistProxyFactory
这两个实现类。我们在回答该问题时需要明确无论采用哪种实现方式，延迟加载最终处理的就是一个形式为 HashMap 的 ResultLoaderMap
对象，并根据标志位完成该对象中内容的填充工作。

**本文内容与建议回答：**

本文中给出了 MyBatis 中 CglibProxyFactory 这一代理工厂类的具体实现方法，该类负责基于 CGLib
框架创建结果对象的代理。我们看到这里 CGLib 的使用过程与《处处是代理：代理机制的应用场景和实现方式》中介绍的内容类似。

### 日常开发技巧

本文中给出的两个面试题，实际上都可能成为我们再日常开发中所需要完成的开发目标。我们可以只通过定义接口的形式就给出对外的服务入口而不需要提供具体的实现类，同时，我们可以根据需要在决定是否加载一些类中的具体字段。动态代理机制为实现这些开发目标提供了基础。如果现实中存在这样的需求，可以把本文中的内容作为基础进行扩展。

### 小结与预告

本文介绍了动态代理机制在 MyBatis 框架中的两大应用场景，即完成 Mapper 接口的执行以及延迟加载的实现。从下一篇开始，我们将讨论 MyBatis
框架中的另一个核心主题，即缓存机制。我们将通过两篇文章来分别介绍其一级缓存和二级缓存的实现原理。

