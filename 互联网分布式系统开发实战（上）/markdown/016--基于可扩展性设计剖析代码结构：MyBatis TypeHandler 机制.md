上文我们讲到了基于回调机制实现系统可扩展性的场景，Mybatis TypeHandler 机制是 Mybatis 中提供的一种类似实现方式，让我们来看一下。

我们知道在 Mybatis 中，在 PreparedStatement
中设置一个参数或者从结果集中取出一个值时，都会用类型处理器将获取的值以合适的方式转换成 Java 类型。这种转换依赖的就是
TypeHandler，它负责完成数据库类型与 JavaBean 类型的转换工作。Mybatis 默认为我们实现了许多 TypeHandler,
当没有配置指定 TypeHandler 时，Mybatis 会根据参数或者返回结果的不同，默认为我们选择合适的 TypeHandler
处理。本节我们先来梳理 TypeHandler 的实现原理，并给出自定义 TypeHandler 的方法。

### TypeHandler 实现原理

在原生的 JDBC 编程中，要对参数进行赋值时，我们一般从 ResultSet
中提取值并赋值给相应的自定义对象，这个过程重复性高且类型转换容易出错。Mybatis 通过一种精妙的方法处理这一操作，其中最重要的就是使用了
TypeHandler 接口。

#### TypeHandler 接口定义

我们首先来看看这个 TypeHandler 接口的定义，该接口位于 org.apache.ibatis.type 包下，如下所示。

    
    
    public interface TypeHandler<T> {
    
          void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;
    
          T getResult(ResultSet rs, String columnName) throws SQLException;
    
          T getResult(ResultSet rs, int columnIndex) throws SQLException;
    
          T getResult(CallableStatement cs, int columnIndex) throws SQLException;
    }
    

其一共定义了四个方法，除了第一个方法 setParameter
是对数据库操作前的参数进行处理之外，另外三个都是在数据库操作完成后对返回值的提取。按照我们上面对接口里所声明的方法的研究，我们应该大致能猜测出
TypeHandler 接口的主要应用场景。其中一个重要场景就是对数据库操作前的参数处理，我们来看一下
org.apache.ibatis.scripting.defaults.DefaultParameterHandler 类中对它的使用方式，截取
setParameters()方法的部分代码，如下所示。。

    
    
    TypeHandler typeHandler = parameterMapping.getTypeHandler();
    JdbcType jdbcType = parameterMapping.getJdbcType();
    if (value == null && jdbcType == null) {
        jdbcType = configuration.getJdbcTypeForNull();
    }
    
    try {
        typeHandler.setParameter(ps, i + 1, value, jdbcType);
    } catch (TypeException | SQLException e) {
        throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
    }
    

以上代码中，首先获取针对本参数的 typeHandler，然后校验获取在配置文件中的设置项 jdbcTypeForNull，最后回调 typeHandler
的自定义实现，从而完成参数的设置。

#### BaseTypeHandler

BaseTypeHandler 是 TypeHandler 的实现，同样位于 org.apache.ibatis.type
包下。BaseTypeHandler 是一个抽象类，在 setParameter()方法中统一了异常处理，并提供了一个
setNonNullParameter()方法和一组 getNullableResult()方法用于子类对其进行扩展。

    
    
    public abstract class BaseTypeHandler<T> extends TypeReference<T> implements TypeHandler<T> {
    
        @Override
        public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType)   throws SQLException {
        }
    
        //省略其他方法
    
        public abstract void setNonNullParameter(PreparedStatement ps, int i, T parameter,  JdbcType jdbcType) throws SQLException;
    
         public abstract T getNullableResult(ResultSet rs, String columnName) throws SQLException;
    
          public abstract T getNullableResult(ResultSet rs, int columnIndex) throws SQLException;
    
          public abstract T getNullableResult(CallableStatement cs, int columnIndex) throws SQLException;
    }
    

在实现自定义 TypHandler 时应该继承 BaseTypeHandler，Mybatis 中内置的 TypHandler 也是如此。我们例举最简单的
StringTypeHandler 类来说明如何使用 BaseTypeHandler。StringTypeHandler 代码如下所示：

    
    
    public class StringTypeHandler extends BaseTypeHandler<String> {
    
      @Override
      public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType)
          throws SQLException {
        ps.setString(i, parameter);
      }
    
      @Override
      public String getNullableResult(ResultSet rs, String columnName)
          throws SQLException {
        return rs.getString(columnName);
      }
    
      @Override
      public String getNullableResult(ResultSet rs, int columnIndex)
          throws SQLException {
        return rs.getString(columnIndex);
      }
    
      @Override
      public String getNullableResult(CallableStatement cs, int columnIndex)
          throws SQLException {
        return cs.getString(columnIndex);
      }
    }
    

显然，这里 Jdbc 类型为 VARCHAR，Java 类型为 String。我们在前面提到 Mybatis 内置了一批包括
StringTypeHandler 在内的 TypeHandler，这些 TypeHandler 都被注册在 TypeHandlerRegister 中。

### 自定义 TypeHandler

除了 Mybatis 内置的各种 TypeHandler 之外，如果我们的 Java 类型与数据库类型无法通过默认规则进行匹配时，Mybatis
提供了自定义 TypeHandler 机制，允许开发人员根据需要定制化类型转换过程。举一个常见的例子，我们希望将 Java 字符数组
String[]与数据库 varchar 类型之间实现相互转换，例如将 String[] books = {"book1",
"book2"}转换为”book1, book2"，反之亦然。这里，我们定义个 CustomStringTypeHandler 来实现上述操作，代码如下。

    
    
    @MappedTypes({String[].class})
    @MappedJdbcTypes({JdbcType.VARCHAR})
    public class CustomStringTypeHandler extends BaseTypeHandler<String[]>{
    
        @Override
        public void setNonNullParameter(PreparedStatement ps, int i, String[] parameter, JdbcType jdbcType) throws SQLException {
            StringBuffer result = new StringBuffer();
            for (String value : parameter) {
                result.append(value).append(",");
            }
            result.deleteCharAt(result.length() - 1);
            ps.setString(i, result.toString());
        }
    
        @Override
        public String[] getNullableResult(ResultSet rs, String columnName) throws SQLException {
            return this.getStringArray(rs.getString(columnName));
        }
    
        @Override
        public String[] getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
            return this.getStringArray(rs.getString(columnIndex));
        }
    
        @Override
        public String[] getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
            return this.getStringArray(cs.getString(columnIndex));
        }
    
        private String[] getStringArray(String columnValue) {
            if (columnValue == null)
                return null;
            return columnValue.split(",");
        }
    }
    

上述代码都是自解释的，我们不难明白其中的逻辑。接下来我们只要在 Mybatis 的配置段中添加对 CustomStringTypeHandler
类的引用即可。通过这个示例，我们明白通过简单实现 TypeHandler 接口就能对现有 Mybatis 进行扩展，Mybatis
为此提供了一个非常灵活的扩展点。

### 面试题分析

#### 如何理解 Mybatis 中 TypeHandler 的作用和基本原理？

  * 考点分析

这道题的考点是针对 Mybatis 框架中的某一个具体特性展开考查，主要还是考查面试者的知识体系而不是实践能力。当然，TypeHandler
有一定的冷僻性，没有结果实践的话可能并不太清楚这个话题，需要提前准备。

  * 解题思路

关于 Mybatis 中 TypeHandler 的相关内容实际上并不复杂，首要的是补充自己对 Mybatis 的知识体系，需要了解在 Mybatis
中存在这个机制，最好有实际的应用经历。然后，如果我们能够结合扩展性话题来对 TypeHandler
的作用和原理进行展开，那无疑是一个加分操作，可以提供面试官的认可度。

  * 本文内容与建议回答

本文内容从理论和实践两方面给出了 Mybatis 中 TypeHandler 的详细介绍，回答上也建议先从扩展性的大方向进行切入，然后给出
TypeHandler 的设计思想和在 Mybatis 中的应用场景。最后，通过一个简单的案例阐述基于 TypeHandler 实现扩展性的具体方式。

### 日常开发技巧

Mybatis TypeHandler 的应用在日常开发过程中也很常见，我们完全可以根据需要编写一个针对自定义对象的 TypeHandler
来处理复杂对象与数据库存储媒介之间的映射管理。具体实现方法在前面的"自定义 TypeHandler"部分已经给出了示例。

### 小结与预告

扩展性是系统设计的一个永恒话题，关于这个话题的讨论还远远没有结束。我们将在本课程接下来的《剖析框架中的架构模式与设计模式》模块中再基于 Dubbo
对微内核模式、管道-过滤器模式进行详细展开。

至此，整个课程的第一个模块《剖析框架代码的结构》就介绍完毕。从下一篇开始，我们将进入本课程的第二部分内容，即《剖析框架中的架构模式与设计模式》的相关学习。

