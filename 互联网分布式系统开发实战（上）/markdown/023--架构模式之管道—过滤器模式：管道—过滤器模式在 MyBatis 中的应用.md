在 MyBatis 中，管道—过滤器模式通过拦截器（Interceptor）的概念进行体现。而对外进行暴露时，则用到了 Plugin 配置项。

我们可以先来看一下在 MyBatis 中使用管道—过滤器模式的方式，那就是在配置文件中添加类似如下配置项，可以看到在 <plugin> 配置段（位于
<configuration> 配置段下）中可以添加一个自定义的 interceptor 配置项。

    
    
    <plugins>  
         <plugin interceptor=”com.tianyalan.mybatis.interceptor.MyInterceptor”>  
              <property name=”prop1″ value=”prop1″/>  
              <property name=”prop2″ value=”prop2″/>  
        </plugin>  
    </plugins>  
    

然后，在解析 XML 配置文件的 XMLConfigBuilder 类中，我们找到了以下解析 <plugin> 配置段的代码。

    
    
     private void pluginElement(XNode parent) throws Exception {
        if (parent != null) {
          for (XNode child : parent.getChildren()) {
            String interceptor = child.getStringAttribute("interceptor");
            Properties properties = child.getChildrenAsProperties();
            Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
            interceptorInstance.setProperties(properties);
            configuration.addInterceptor(interceptorInstance);
          }
        }
    }
    

在上述代码中，我们看到了 MyBatis 中代表过滤器的 Interceptor 接口。我们根据配置的 interceptor 属性实例化
Interceptor 对象，然后通过 configuration.addInterceptor 方法添加新的 Interceptor 实例。

我们跟踪代码，发现 Configuration 中定义了一个如下所示的 InterceptorChain 对象，该方法就是将 Interceptor
实例添加到了 InterceptorChain 中。

    
    
    protected final InterceptorChain interceptorChain = new InterceptorChain();
    

这样，一个管道—过滤器模式中代表过滤器（Interceptor）和过滤器链（InterceptorChain）的相关对象都出现了，让我们首先从这些对象的定义和实现入手探究
MyBatis 的内部原理。

### MyBatis 中的 Interceptor

在 MyBatis 中，Interceptor 和 InterceptorChain 都位于 org.apache.ibatis.plugin 包中，其中
Interceptor 是个接口，InterceptorChain 是个实体类，它们的代码看上去都不多。让我们先来看一下 InterceptorChain
类，代码如下：

    
    
    public class InterceptorChain {
    
      private final List<Interceptor> interceptors = new ArrayList<>();
    
      public Object pluginAll(Object target) {
        for (Interceptor interceptor : interceptors) {
          target = interceptor.plugin(target);
        }
        return target;
      }
    
      public void addInterceptor(Interceptor interceptor) {
        interceptors.add(interceptor);
      }
    
      public List<Interceptor> getInterceptors() {
        return Collections.unmodifiableList(interceptors);
      }
    }
    

对比我们在前面介绍的自定义过滤器链示例，这个 InterceptorChain 同样提供了 addInterceptor
方法用于将拦截器添加到链中。不同之处在于，它在这里持有一个 interceptors 数组用于把新加入的 Interceptor
添加到这个数组中。通过这种方式，在 pluginAll 方法（类似于《架构模式之管道—过滤器模式：管道—过滤器模式与实现示例》一文中自定义过滤器链示例中的
execute 方法）就可以直接遍历 interceptors 数组并通过每个 interceptor 执行拦截逻辑。

显然 InterceptorChain 中我们尚不明确的就是 interceptor.plugin(target) 方法的逻辑，让我们把思路跳转到
Interceptor 接口，该接口定义如下：

    
    
    public interface Interceptor {
    
      Object intercept(Invocation invocation) throws Throwable;
    
      default Object plugin(Object target) {
        return Plugin.wrap(target, this);
      }
    
      default void setProperties(Properties properties) {
        // NOP
      }
    }
    

可以看到，Interceptor 接口中的 plugin 方法实际上存在一个默认实现，这里它通过 Plugin.wrap
方法完成对目标对象的拦截。Plugin.wrap 是一个静态方法，位于 Plugin 类中，Plugin
类的核心代码如下所示（为了简洁省略了部分辅助性代码）：

    
    
    public class Plugin implements InvocationHandler {
    
      //省略变量定义和构造函数
    
      public static Object wrap(Object target, Interceptor interceptor) {
        Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
        Class<?> type = target.getClass();
        Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
        if (interfaces.length > 0) {
          return Proxy.newProxyInstance(
              type.getClassLoader(),
              interfaces,
              new Plugin(target, interceptor, signatureMap));
        }
        return target;
      }
    
      @Override
      public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
          Set<Method> methods = signatureMap.get(method.getDeclaringClass());
          if (methods != null && methods.contains(method)) {
            return interceptor.intercept(new Invocation(target, method, args));
          }
          return method.invoke(target, args);
        } catch (Exception e) {
          throw ExceptionUtil.unwrapThrowable(e);
        }
      }
    
        //省略 getSignatureMap 方法和 getAllInterfaces 辅助方法
    }
    

这里我们看到了熟悉的 InvocationHandler 接口和 Proxy.newProxyInstance 实现方法，从而明白了原来用到了 JDK
的动态代理机制。我们通过 getSignatureMap 方法从拦截器的注解中获取拦截的类名和方法信息，获取要改变行为的类，然后通过
getAllInterfaces 方法获取接口，最后通过动态代理机制产生代理，这样使得只有是 Interceptor
注解的接口的实现类才会产生代理。在这里，getSignatureMap 方法和 getAllInterfaces 方法都是基于反射机制进行实现。

另一方面，我们实现了 InvocationHandler 接口的 invoke 方法，该方法会判断是否有需要拦截的方法，如果有则并调用
Interceptor.intercept 方法，在这里我们就可以加入任何我们想要加入的业务逻辑；如果没有，则调用 method.invoke
方法执行原来逻辑。

请注意，这里关于动态代理实现机制的介绍我们没有做具体展开，因为在后续课程的《剖析通用机制与组件》模块中专门会有一个主题来全面介绍动态代理及其在远程调用和数据访问中的实现原理。

讲完 Interceptor 和 InterceptorChain 之后，让我们再次回到 Configuration 类，并找到以下代码：

    
    
    public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
        ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
        parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
        return parameterHandler;
    }
    
    public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
          ResultHandler resultHandler, BoundSql boundSql) {
        ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
        resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
        return resultSetHandler;
    }
    
    public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
        StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
        statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
        return statementHandler;
    }
    
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
    

介绍这段代码的目的是在说明这样一个事实：在 MyBatis 中，拦截器只能拦截
ParameterHandler、StatementHandler、ResultSetHandler 和 Executor 这四种类型的接口，这点在
Configuration 类通过以上代码是预先定义好的。我们想要自定义拦截器，也只能围绕上述四种接口添加逻辑。

这四个接口之间的关系和拦截顺序如下图所示：

![23.01](https://images.gitbook.cn/2020-05-25-052622.png)

可以看到，对于 SQL 的执行过程而言，上述四个环节的拦截机制基本已经可以满足日常的定制化需求。接下来，我们就来看一下如何在 MyBatis 中自定义一个
Interceptor。

### 自定义 Interceptor

如果我们想要在 MyBatis 中实现一个自定义拦截器，要做的事情就是实现前面介绍的 Interceptor 接口，并在该接口上指定相应的
Signature 信息。一个空白的 Interceptor 实现类模版如下所示：

    
    
    @Intercepts({@Signature(type = Executor.class, method ="update", args = {MappedStatement.class, Object.class})})
    public class MyInterceptor implements Interceptor {
        @Override
        public Object intercept(Invocation invocation) throws Throwable {
            return invocation.proceed();
        }
    
        @Override
        public Object plugin(Object target) {
            returnPlugin.wrap(target, this);
        }
    
        @Override
        public void setProperties(Properties properties) {
        }
    }
    

在上述 MyInterceptor 类的 intercept 方法中，需要执行 invocation.proceed() 方法显式地推进
InterceptChain，而我们可以在这个方法的前后添加定制化处理过程。

接下来我们来考虑一个拦截器的常见应用场景。在实现数据库插入和更新操作时，我们往往对这条记录的更新时间进行同步更新。我们当然可以为每句 SQL
添加相应的时间处理方法，但更好的一种方式是通过自定义拦截器的方式来自定完成这一步操作。显然，这一步操作应该是在 Executor 中进行完成。根据前文中对
Plugin 类的介绍，我们首先需要明确 Executor 中对应的方法，而这方法就是如下所示的 update 方法：

    
    
    int update(MappedStatement ms, Object parameter) throws SQLException;
    

明确了 Signature 信息之后，我们就可以来实现着手实现整个流程。为了方便起见，我们可以提供一个如下所示的注解，专门用来标识具体需要进行拦截字段：

    
    
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.FIELD})
    public @interface UpdateTimeStamp {
    
       String value() default "";
    }
    

然后，我们创建一个业务领域类，把该注解作用于具体的更新时间注解上，如下所示：

    
    
    public class MyDomain {
    
        //省略其他字段定义
    
        @UpdateTimeStamp
        public Date updateTimeStamp;
    }
    

完整的 UpdateTimeStampInterceptor 实现如下，我们对关键代码都添加了注释：

    
    
    @Intercepts({@Signature(type = Executor.class, method = "update", args = {MappedStatement.class, Object.class})})
    public class UpdateTimeStampInterceptor implements Interceptor {
    
        @Override
        public Object intercept(Invocation invocation) throws Throwable {
            //获取 MappedStatement
            MappedStatement mappedStatement = (MappedStatement) invocation.getArgs()[0];
            //获取 SqlCommandType
            SqlCommandType sqlCommandType = mappedStatement.getSqlCommandType();
            //获取 Parameter
            Object parameter = invocation.getArgs()[1];
    
            if (parameter != null) {
                Field[] declaredFields = parameter.getClass().getDeclaredFields();
    
                for (Field field : declaredFields) {
                    //获取 UpdateTimeStamp 注解
                    if (field.getAnnotation(UpdateTimeStamp.class) != null) {
                        //如果是 Insert 或 Update 操作，则更新操作时间
                        if (SqlCommandType.INSERT.equals(sqlCommandType) || SqlCommandType.UPDATE.equals(sqlCommandType)) {
                            field.setAccessible(true);
                            if (field.get(parameter) == null) {
                                //设置参数值
                                field.set(parameter, new Date());
                            }
                        }
                    }
                }
            }
    
            //继续执行拦截器链
            return invocation.proceed();
        }
    
        @Override
        public Object plugin(Object o) {
            return Plugin.wrap(o,this);
        }
    
        @Override
        public void setProperties(Properties properties) {
        }
    }
    

上述 UpdateTimeStampInterceptor 类的实现过程展示了如何获取与 Executor 相关的 Statement、SQL
类型以及所携带的参数。通过这种方法，我们可以实现在新增或者删除数据库记录时，动态地添加所需要的字段值。同样，这种处理方式可以扩展到任何我们想要处理的字段和参数。

### 面试题分析

#### **1\. MyBatis 中的拦截器如何对执行过程进行拦截？**

**考点分析：**

针对这类问题，我们首先需要明确背后的考点实际上就是代理机制。在主流的开源框架中，涉及到过滤、拦截等常见下的实现原理往往都有代理模式相关。只要掌握动态代理机制，不同的问法都是类似的解答思路。

**解题思路：**

MyBatis 中实现拦截的基本原理还是基于动态代理机制，通过获取对应方法的签名信息以及输入的接口和参数来生成代理。而动态代理的实现方法也比较简单，直接基于
JDK 的 InvocationHandler 接口和 Proxy.newProxyInstance 静态方法。

**本文内容与建议回答：**

本文重点介绍了 Plugin 类，该类实现了 JDK 的 InvocationHandler 接口，也使用了 Proxy.newProxyInstance
静态方法，是动态代理模式的一种典型实现。掌握了该类的实现过程，都可以结合自己的理解对代理机制进行展开。

#### **2\. MyBatis 中内置了对那些操作的拦截？**

**考点分析：**

这个问题对于没有亲手实现过 MyBatis 拦截器的同学而言，乍一看可能有点不知道如何回答。但对于实现过的同学而言，则应该很明确，考查的是 MyBatis
中整个 SQL 执行过程中可以添加拦截器的操作和方法。

**解题思路：**

考虑到整个 SQL 的执行过程，MyBatis 内部可以对执行器 Executor、执行语句处理器 StatementHandler、参数处理器
ParameterHandler 和结果集处理器 ResultSetHandler 的各种方法和操作进行拦截，相当于对整个 SQL
执行过程而言，我们都可以有办法添加自定义的执行逻辑。

这四个 Handler 也比较容易记忆，涉及到 Statement、Parameter、Execute、ResultSet 处理等环节，可以基于 SQL
的一次执行过程进行梳理。

**本文内容与建议回答：**

在 MyBatis 中，我们知道上述四个 Handler 实际上是写死在 Configuration 类中了，我们可以对其进行扩展。我们也针对
Executor 的 update 方法给出了一个自定义的 Interceptor
实现，用于动态设置数据库中某些数据项的值，这些内容对于回答这个问题提供了理论和实践上的支持。

### 日常开发技巧

在日常开发过程中，可以实现自定义的 MyBatis 拦截器来扩展 MyBatis
框架的执行流程。本文中给出了一个简单的示例，用于完成对某一个参数的自动填充。同样，我们也可以添加一下针对 SQL
执行的分页逻辑，或者加强运行时的监控和统计功能。

### 小结与预告

本文结合 MyBatis 框架详细剖析了管道过滤器架构模式的实现和应用方式，这是架构模式相关内容的最后一篇。从下一篇开始，我们将通过剖析 MyBatis
源码来讨论典型的设计模式的实现机制。

