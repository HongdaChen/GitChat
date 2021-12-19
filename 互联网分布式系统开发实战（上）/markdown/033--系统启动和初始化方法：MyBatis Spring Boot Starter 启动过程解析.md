介绍完 Spring Boot 中应用程序的启动配置机制之后，我们来做一些实践，通过剖析 MyBatis Spring Boot Starter
的启动过程来加深对所介绍内容的理解。

本文中介绍的是 mybatis-spring-boot-starter 版本为 2.x。mybatis-spring-boot-starter
中存在几个代码工程，我们重点关注 mybatis-spring-boot-autoconfigure 工程，而这个代码中，最重要的显然就是
MybatisAutoConfiguration 类。对于 Spring Boot 中的 AutoConfiguration
类，我们首先需要重点关注的是类定义上的注解，如下所示：

    
    
    @org.springframework.context.annotation.Configuration
    @ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
    @ConditionalOnSingleCandidate(DataSource.class)
    @EnableConfigurationProperties(MybatisProperties.class)
    @AutoConfigureAfter(DataSourceAutoConfiguration.class)
    public class MybatisAutoConfiguration implements InitializingBean {
    

我们看到这里用到了一个新的@ConditionalOnXXX 注解，这是一个非常重要的注解。在介绍 MybatisAutoConfiguration
之前，有必要对这个注解做详细展开。

### @ConditionalOn 系列条件注解

Spring Boot 默认提供了 100 多个 AutoConfiguration
类，显然我们不可能会全部引入。所以在自动装配时，系统会去类路径下寻找是否有对应的配置类。如果有配置类，则按条件注解 @Conditional 或者
@ConditionalOnXXX 等相关注解进行判断，决定是否需要装配。这里就引出了在阅读 Spring Boot 代码时经常会碰到的另一批注解，即
@ConditionalOn 系列条件注解。

#### **@ConditionalOn 系列条件注解**

我们在 MybatisAutoConfiguration 类上看到了两个@ConditionalOn 注解，一个是
@ConditionalOnClass，一个是 @ConditionalOnProperty，一个是
@ConditionalOnSingleCandidate。实际上，Spring Boot 中提供了一系列的条件注解，包括：

  * @ConditionalOnProperty：只有当所提供的属性属于 true 时才会实例化 Bean
  * @ConditionalOnBean：只有在当前上下文中存在某个对象时才会实例化 Bean
  * @ConditionalOnClass：只有当某个 Class 位于类路径上时才会实例化 Bean
  * @ConditionalOnExpression：只有当表达式为 true 的时候才会实例化 Bean
  * @ConditionalOnMissingBean：只有在当前上下文中不存在某个对象时才会实例化 Bean
  * @ConditionalOnMissingClass：只有当某个 Class 在类路径上不存在的时候才会实例化 Bean
  * @ConditionalOnSingleCandidate：当指定的 Bean 在容器中只有一个，或者有多个但是指定了首选的 Bean 时触发实例化
  * @ConditionalOnNotWebApplication：只有当不是 Web 应用时才会实例化 Bean

当然 Spring Boot 还提供了一些不大常用的@ConditionalOnXXX 注解，这些注解都定义在
org.springframework.boot.autoconfigure.condition 包中。

显然上述 MybatisAutoConfiguration 能够实例化的前提有两个，一个是类路径中存在 SqlSessionFactory 和
SqlSessionFactoryBean，另一个是容器中只存在一个 DataSource 实例。两者缺一不可，这是一种常用的自动配置控制技巧。

#### **@ConditionalOn 系列条件注解的实现原理**

@ConditionalOn
系列条件注解非常多，我们无意对所有这些组件进行展开。事实上这些注解的实现原理也大致相同，我们只要深入了解其中一个就能做到触类旁通。这里我们挑选
@ConditionalOnClass 注解进行展开，该注解定义如下：

    
    
    @Target({ ElementType.TYPE, ElementType.METHOD })
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Conditional(OnClassCondition.class)
    public @interface ConditionalOnClass {
        Class<?>[] value() default {};
        String[] name() default {};
    }
    

可以看到 @ConditionalOnClass 本身带有两个属性，一个 class 类型的 value，一个 String 类型的
name，所以我们可以采用这两种方式中的任意一种来使用该注解。同时 ConditionalOnClass 注解本身还带了一个
@Conditional(OnClassCondition.class) 注解。所以，其实 ConditionalOnClass 注解的判断条件就在于
OnClassCondition 这个中法。

OnClassCondition 是 SpringBootCondition 的子类，而 SpringBootCondition 又实现了
Condition 接口。Condition 接口只有一个 matches 方法，如下所示：

    
    
    public interface Condition {
        boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
    }
    

SpringBootCondition 中的 matches 方法实现如下：

    
    
    @Override
    public final boolean matches(ConditionContext context,
            AnnotatedTypeMetadata metadata) {
        String classOrMethodName = getClassOrMethodName(metadata);
        try {
            ConditionOutcome outcome = getMatchOutcome(context, metadata);
            logOutcome(classOrMethodName, outcome);
            recordEvaluation(context, classOrMethodName, outcome);
            return outcome.isMatch();
        }
        //省略其他方法
    }
    

getClassOrMethodName 方法获取加上了 @ConditionalOnClass 注解的类或者方法的名称，而 getMatchOutcome
方法用于获取匹配的输出。我们看到 getMatchOutcome 方法实际上是一个抽象方法，需要交由 SpringBootCondition
的各个子类完成实现，这里的子类就是 OnClassCondition 类。在理解 OnClassCondition 时，我们要明白在 Spring Boot
中，@ConditionalOnClass 或者@ConditionalOnMissingClass 注解对应的条件类都是
OnClassCondition，所以在 OnClassCondition 的 getMatchOutcome 中会同时处理两种情况。这里我们挑选处理
@ConditionalOnClass 注解的代码，核心逻辑如下所示：

    
    
    ClassLoader classLoader = context.getClassLoader();
            ConditionMessage matchMessage = ConditionMessage.empty();
            List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
            if (onClasses != null) {
                List<String> missing = getMatches(onClasses, MatchType.MISSING, classLoader);
                if (!missing.isEmpty()) {
                    return ConditionOutcome
                            .noMatch(ConditionMessage.forCondition(ConditionalOnClass.class)
                                    .didNotFind("required class", "required classes")
                                    .items(Style.QUOTE, missing));
                }
                matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
                        .found("required class", "required classes").items(Style.QUOTE,
                                getMatches(onClasses, MatchType.PRESENT, classLoader));
            }
    

这里有两个方法值得注意，一个是 getCandidates 方法，一个是 getMatches 方法。首先通过 getCandidates 方法获取了
ConditionalOnClass 的 name 属性和 value 属性。然后通过 getMatches 方法（通过应用
MatchType）把这些属性值进行比对，得到这些属性所指定的但在类加载器中不存在的类。如果发现类加载器中应该存在但事实上又不存在的类，则返回一个匹配失败的
Condition；反之，如果类加载器中存在对应类的话，则把匹配信息进行记录并返回一个 ConditionOutcome。

然后，我们在 MybatisAutoConfiguration 类上看到了一个 @EnableConfigurationProperties
注解，该注解的作用就是使添加了 @ConfigurationProperties 注解的类生效。在 Spring Boot 中，如果该类只使用了
@ConfigurationProperties 注解，然后该类没有在扫描路径下或者没有使用 @Component 等注解，就会导致无法被扫描为
bean，那么就必须在配置类上使用 @EnableConfigurationProperties 注解去指定这个类，才能使
@ConfigurationProperties 生效，然后作为 bean 添加进 Spring 容器中。这里的
@EnableConfigurationProperties 注解中指定的是 MybatisProperties 类，该类定义了 MyBatis
运行时所需要的各种配置信息，而我们在 MybatisProperties 类上确实也发现了 @ConfigurationProperties 注解。

最后，MybatisAutoConfiguration 类上还存在一个@AutoConfigureAfter
注解，这个注解可以根据字面意思进行理解，即在完成某一个类的自动配置之后（这里是
DataSourceAutoConfiguration）再执行当前类的自动配置。

### MybatisAutoConfiguration

理解上 @ConditionalOnXXX、@EnableConfigurationProperties 和 @AutoConfigureAfter
等一些类注解之后，我们再来看 MybatisAutoConfiguration
类的代码结构就显得比较简单明了。MybatisAutoConfiguration 类中核心方法之一就是如下所示的 sqlSessionFactory 方法：

    
    
      @Bean
      @ConditionalOnMissingBean
      public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSource);
        factory.setVfs(SpringBootVFS.class);
        if (StringUtils.hasText(this.properties.getConfigLocation())) {
          factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
        }
        applyConfiguration(factory);
    
        //省略一系列配置项设置方法
    
        return factory.getObject();
      }
    

显然，这里基于《系统启动和初始化方法：Mybatis-Spring 启动过程解析》中介绍的 SqlSessionFactoryBean
构建了实例。注意到该方法上同样添加了一个 @ConditionalOnMissingBean 注解，标明只有在当前上下文中不存
SqlSessionFactoryBean 对象时才会执行上述方法。

同样添加了 @ConditionalOnMissingBean 注解的还有如下所示的 sqlSessionTemplate 方法：

    
    
      @Bean
      @ConditionalOnMissingBean
      public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        ExecutorType executorType = this.properties.getExecutorType();
        if (executorType != null) {
          return new SqlSessionTemplate(sqlSessionFactory, executorType);
        } else {
          return new SqlSessionTemplate(sqlSessionFactory);
        }
      }
    

该方法用于构建一个 SqlSessionTemplate 对象实例，关于 SqlSessionTemplate
的设计思想和实现过程，我们在《系统启动和初始化方法：MyBatis-Spring 启动过程解析》中也进行了详细介绍。

接下来，我们需要在 META-INF/spring.factories 文件中明确所指定的自动配置类。根据 Spring Boot 自动配置机制的原理，对于
mybatis-spring-boot-autoconfigure 工程而言，这个配置项内容应该如下所示：

    
    
    # Auto Configure
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
    

最后，我们来到 mybatis-spring-boot-starter 工程，在其 META-INF/spring.factories
文件中找到了如下内容。

    
    
    provides: mybatis-spring-boot-autoconfigure,mybatis,mybatis-spring
    

至此，整个 MyBatis Spring Boot Starter 的介绍就告一段落。作为总结，我们可以把创建一个 Spring Boot Starter
的过程抽象成如下三步：

  1. 新建配置类，写好配置项和默认的配置值，指明配置项前缀；
  2. 新建自动装配类，使用 @Configuration 和 @Bean 来进行自动装配；
  3. 新建 spring.factories 文件，指定 Starter 的自动装配类。

### 面试题分析

#### **Spring Boot 中 @ConditionalOn 系列条件注解的作用是什么？请列举几个你所熟悉的 @ConditionalOn
注解？**

**考点分析：**

这道题可以说是对 Spring Boot 自动配置机制原理的一种补充，因为自动配置过程中一般都需要 @ConditionalOn
进行配合起来一起使用。从考点上讲，我们明确 @ConditionalOn 的作用是比较明确的，常见的 @ConditionalOn
注解也比较多，因此该题相对而言比较容易回答，

**解题思路：**

显然，我们明确 @ConditionalOn 系列注解的目的是提供一种条件判断，只有符合条件的目标类才会通过 Spring Boot
的自动配置机制进行记载。作为这一目的的补充，我们可以简单分析 Spring Boot
能够做到这一点的原理，这是该题的加分点，因为从考点上讲并没有明确需要面试者介绍原理。至于列举熟悉的 @ConditionalOn 注解就属于送分题了，只要对
Spring Boot 有一定的应用经验应该都能回答。

**本文内容与建议回答：**

本文对 @ConditionalOn 系列注解进行了详细的介绍，涉及到它们的目的、类别和实现原理。

同时我们围绕 MyBatis Spring Boot Starter 框架中的 MybatisAutoConfiguration 自动配置类介绍了
@ConditionalOn 注解的具体使用方法。当然，围绕 MybatisAutoConfiguration，我们还给出了
@AutoConfigureAfter 注解等辅助类的说明。

### 日常开发技巧

本文在结尾部分总结了开发一个 Spring Boot Starter 的三大步骤，在日常开发过程中，我们就可以基于这三大步骤来实现一个自定义的 Spring
Boot Starter。日常开发过程中，开发一个 Spring Boot Starter
也是常见的需求，大家在开发过程中可以基于本文的内容加深对其实现原理的理解。

### 小结与预告

本文与上一篇文章一起讲解了 Spring Boot 的自动配置机制以及在 MyBatis Spring Boot Starter
中的应用，关于这个主题的讨论就到这里。下一篇文章中，我们将关于 Spring 框架中另一种常见的扩展机制，即自定义标签的实现方案。

