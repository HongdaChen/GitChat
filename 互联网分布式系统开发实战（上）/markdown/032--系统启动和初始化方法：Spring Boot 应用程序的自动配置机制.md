当下，Spring Boot 已经成为开发单体服务的主流框架。本课程后面将要介绍的 Spring Cloud 微服务开发框架，而我们知道在 Spring
Cloud 框架中，每个微服务本质上就是一个 Spring Boot 应用程序，因此了解 Spring Boot 是分析 Spring Cloud
框架的前提。另一方面，Dubbo 和 MyBatis 也分别实现了了 Dubbo Spring Boot Starter 和 MyBatis Spring
Boot Starter 这两个 Starter 来提供对 Spring Boot 的支持。在引入这些组件之前，我们希望通过一个专题来介绍 Spring
Boot 应用程序开发过程中与系统启动相关的概念和技术。

作为 Spring 家族新的一员，Spring Boot
提供了令人兴奋的特性，这些特性主要体现在开发过程的简单之美。例如，支持快速构建项目、不依赖外部容器独立运行、开发部署效率高以及与云平台天然集成。这些特点促使我们选择
Spring Boot 来构建微服务。Spring Boot 的核心优势体现在编码、配置、部署、监控等多个方面，为构建基于 Spring Cloud
的微服务架构提供了技术基础。

Spring Boot
是一套强大而复杂的体系，我们主要关注于其最基础、最核心的自动配置（AutoConfiguration）机制，我们先从@SpringBootApplication
注解开始。

### @SpringBootApplication 注解

@SpringBootApplication 注解位于 spring-boot-autoconfigure 工程的
org.springframework.boot.autoconfigure 包中，定义如下：

    
    
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @SpringBootConfiguration
    @EnableAutoConfiguration
    @ComponentScan(excludeFilters = {
            @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
            @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
    public @interface SpringBootApplication {
        @AliasFor(annotation = EnableAutoConfiguration.class)
        Class<?>[] exclude() default {};
    
        @AliasFor(annotation = EnableAutoConfiguration.class)
        String[] excludeName() default {};
    
        @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
        String[] scanBasePackages() default {};
    
        @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
        Class<?>[] scanBasePackageClasses() default {};
    }
    

相较一般的注解，@SpringBootApplication
注解显得有点复杂。我们可以通过该注解配置不需要实现自动装配的类或类名，也可以配置需要进行扫描的包路径和类路径。注意到@SpringBootApplication
注解实际上是一个组合注解，是由三个注解组合而成，分别是@SpringBootConfiguration、@EnableAutoConfiguration
和@ComponentScan。

  * **@ComponentScan 注解** ：@ComponentScan 注解不是 Spring Boot 引入的新注解，而是属于 Spring 容器管理的内容。@ComponentScan 注解就是扫描基于@Component 等注解所标注的类所在包下的所有需要注入的类，并把相关 Bean 定义批量加载到 IoC 容器中。显然，Spring Boot 应用程序中同样需要这个功能。
  * **@SpringBootConfiguration 注解** ：@SpringBootConfiguration 注解比较简单，事实上它是一个空注解，只是使用了 Spring 中的@Configuration 注解。@Configuration 注解比较常见，提供了 JavaConfig 配置类实现。
  * **@EnableAutoConfiguration 注解** ：@EnableAutoConfiguration 注解是我们需要重点需要剖析的对象，下面进行重点展开。

#### **@EnableAutoConfiguration 注解**

@EnableAutoConfiguration 注解的定义如下：

    
    
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @AutoConfigurationPackage
    @Import(AutoConfigurationImportSelector.class)
    public @interface EnableAutoConfiguration {
    
        String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
    
        Class<?>[] exclude() default {};
    
        String[] excludeName() default {};
    }    
    

这里我们关注两个注解，@AutoConfigurationPackage 和
@Import(AutoConfigurationImportSelector.class)。其中，@AutoConfigurationPackage
注解定义如下：

    
    
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @Import(AutoConfigurationPackages.Registrar.class)
    public @interface AutoConfigurationPackage {
    
    }
    

从命名上讲，在这个注解中我们获取该注解所在包下的组件并进行自动配置，而在实现方式上用到了 Spring 中的 @Import 注解。在使用 Spring
Boot 时，@Import 也是一个非常常见的注解，可以用来动态创建 Bean。为了便于理解后续内容，这里有必要对@Import
注解的运行机制做一些展开，该注解定义如下：

    
    
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    public @interface Import {
        Class<?>[] value();
    }
    

在@Import 注解的参数中可以设置需要引入的类名，例如这里的
@Import(AutoConfigurationPackages.Registrar.class)。根据该类的不同类型，Spring 容器针对
@Import 注解有以下四种处理方式：

  * 如果该类实现了 ImportSelector 接口，Spring 容器就会实例化该类，并且调用其 selectImports 方法。
  * 如果该类实现了 DeferredImportSelector 接口，则 Spring 容器也会实例化该类并调用其 selectImports 方法。DeferredImportSelector 继承了 ImportSelector，但 DeferredImportSelector 的实例的 selectImports 方法调用时机晚于 ImportSelector 的实例，要等到@Configuration 注解中相关的业务全部都处理完了才会调用。
  * 如果该类实现了 ImportBeanDefinitionRegistrar 接口，Spring 容器就会实例化该类，并且调用其 registerBeanDefinitions 方法。
  * 如果该类没有实现 ImportSelector、DeferredImportSelector、ImportBeanDefinitionRegistrar 等其中的任何一个，Spring 容器就会直接实例化该类。

有了对 @Import 注解的基本理解，我们来看 AutoConfigurationPackages.Registrar 类，定义如下：

    
    
    static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
    
            @Override
            public void registerBeanDefinitions(AnnotationMetadata metadata,
                    BeanDefinitionRegistry registry) {
                register(registry, new PackageImport(metadata).getPackageName());
            }
    
            @Override
            public Set<Object> determineImports(AnnotationMetadata metadata) {
                return Collections.singleton(new PackageImport(metadata));
            }
    }
    

可以看到这个 Registrar 类实现了前面 @Import 注解第三种情况中提到的 ImportBeanDefinitionRegistrar
接口并重写了 registerBeanDefinitions 方法，该方法中调用 AutoConfigurationPackages 自身的
register 方法：

    
    
    public static void register(BeanDefinitionRegistry registry, String... packageNames) {
            if (registry.containsBeanDefinition(BEAN)) {
                BeanDefinition beanDefinition = registry.getBeanDefinition(BEAN);
                ConstructorArgumentValues constructorArguments = beanDefinition
                        .getConstructorArgumentValues();
                constructorArguments.addIndexedArgumentValue(0,
                        addBasePackages(constructorArguments, packageNames));
            }
            else {
                GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
                beanDefinition.setBeanClass(BasePackages.class);
                beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0,
                        packageNames);
                beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
                registry.registerBeanDefinition(BEAN, beanDefinition);
            }
        }
    

这个方法的逻辑是先判断整个 Bean 有没有被注册，如果已经注册则获取 Bean 的定义，通过 Bean
获取构造函数的参数并添加参数值；如果没有，则创建一个新的 Bean 的定义，设置 Bean 的类型为 AutoConfigurationPackages
类型并进行 Bean 的注册。

然后我们再来看@EnableAutoConfiguration 注解中的
@Import(AutoConfigurationImportSelector.class) 部分，首先我们明确
AutoConfigurationImportSelector 类实现了 @Import 注解第二种情况中的 DeferredImportSelector
接口，所以会执行如下所示的 selectImports 方法：

    
    
        @Override
        public String[] selectImports(AnnotationMetadata annotationMetadata) {
            if (!isEnabled(annotationMetadata)) {
                return NO_IMPORTS;
            }
            AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
                    .loadMetadata(this.beanClassLoader);
            AnnotationAttributes attributes = getAttributes(annotationMetadata);
    
            //获取 configurations 集合
            List<String> configurations = getCandidateConfigurations(annotationMetadata,
                    attributes);
            configurations = removeDuplicates(configurations);
            Set<String> exclusions = getExclusions(annotationMetadata, attributes);
            checkExcludedClasses(configurations, exclusions);
            configurations.removeAll(exclusions);
            configurations = filter(configurations, autoConfigurationMetadata);
            fireAutoConfigurationImportEvents(configurations, exclusions);
            return StringUtils.toStringArray(configurations);
        }
    

这段代码的核心是通过 getCandidateConfigurations 方法获取 configurations
集合并进行过滤。getCandidateConfigurations 方法如下所示：

    
    
    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
                AnnotationAttributes attributes) {
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
                getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
        Assert.notEmpty(configurations,
                "No auto configuration classes found in META-INF/spring.factories. If you "
                        + "are using a custom packaging, make sure that file is correct.");
        return configurations;
    }
    

看这段代码可以先看关注这个 Assert 校验，该校验是一个非空校验，会提示“在 META-INF/spring.factories
中没有找到自动配置类”这个异常信息。看到这里，希望大家能够把前面学到的知识串联起来，马上想起在《架构模式之微内核模式：微内核模式及 Java SPI
机制》一文中讲到的 SPI 机制，因为无论从 SpringFactoriesLoader 这个类的命名上，还是 META-
INF/spring.factories 这个文件目录，两者之前都存在很大的想通性。

从类名上看，AutoConfigurationImportSelector
类是一种选择器，负责从各种配置项中找到需要导入的具体配置类。AutoConfigurationImportSelector
类的结构如下图所示，其所依赖的最关键组件就是 SpringFactoriesLoader，下面我们对其进行具体展开。

![NsrwaB](https://images.gitbook.cn/2020-05-25-052614.png)

#### **SpringFactoriesLoader**

要想理解 SpringFactoriesLoader 类，首先需要了解 JDK 中 SPI（Service Provider
Interface，服务提供者接口）机制，这里我们对 SPI 机制做简要回顾。JDK 提供了实现服务查找的一个工具类
java.util.ServiceLoader 来实现 SPI 机制。当服务提供者提供了服务接口的一种实现之后，我们可以在 jar 包的 META-
INF/services/目录创建一个以服务接口命名的文件，该文件里配置着一组 Key-
Value，用于指定服务接口与实现该服务接口具体实现类的映射关系。而当外部程序装配这个 jar 包时，就能通过该 jar 包 META-
INF/services/目录中的配置文件找到具体的实现类名，并装载实例化，从而完成模块的注入。SPI
提供了一种约定，基于该约定就能很好的找到服务接口的实现类，而不需要在代码里硬编码指定。

SpringFactoriesLoader 类似这种 SPI 机制，只不过以服务接口命名的文件是放在 META-INF/spring.factories
文件夹下，对应的 Key 为 EnableAutoConfiguration。SpringFactoriesLoader 会查找所有 META-
INF/spring.factories 文件夹中配置文件，并把 Key 为 EnableAutoConfiguration
所对应的配置项通过反射实例化为配置类并加载到容器。我们可以在 SpringFactoriesLoader 的 loadSpringFactories
方法中印证这一点：

    
    
        private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
            MultiValueMap<String, String> result = cache.get(classLoader);
            if (result != null) {
                return result;
            }
    
            try {
                Enumeration<URL> urls = (classLoader != null ?
                        classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                        ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
                result = new LinkedMultiValueMap<>();
                while (urls.hasMoreElements()) {
                    URL url = urls.nextElement();
                    UrlResource resource = new UrlResource(url);
                    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                    for (Map.Entry<?, ?> entry : properties.entrySet()) {
                        String factoryClassName = ((String) entry.getKey()).trim();
                        for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                            result.add(factoryClassName, factoryName.trim());
                        }
                    }
                }
                cache.put(classLoader, result);
                return result;
            }
            catch (IOException ex) {
                throw new IllegalArgumentException("Unable to load factories from location [" +
                        FACTORIES_RESOURCE_LOCATION + "]", ex);
            }
        }
    

以下就是 spring-boot-autoconfigure 工程中所使用的 spring.factories 配置文件片段，可以看到
EnableAutoConfiguration 项中包含了各式各样的配置项，这些配置项在 Spring Boot 启动过程中都能够通过
SpringFactoriesLoader 加载到运行时环境，从而实现自动化配置。

    
    
    # Auto Configure
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
    org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
    org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
    org.springframework.boot.autoconfigure.MessageSourceAutoConfiguration,\
    org.springframework.boot.autoconfigure.PropertyPlaceholderAutoConfiguration,\
    org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
    org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
    org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
    org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
    org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
    …
    

以上就是 Spring Boot 中基于@SpringBootApplication
注解实现自动配置的基本过程和原理。当然，@SpringBootApplication
注解也可以基于外部配置文件加载配置信息。基于约定优于配置思想，Spring Boot 在加载外部配置文件的过程中大量使用了默认配置。

### 面试题分析

#### **Spring Boot 如何实现自动配置机制？**

**考点分析：**

随着 Spring Boot 的日渐流行，这个问题也变成了一个经典问题。但凡是使用过 Spring Boot
的候选人，笔者一般都会通过问这个问题来考查对方的知识体系。

自动配置是 Spring Boot 的最核心机制，该问题的考点明确，问题的答案也很明确，大家通过记忆以及加上自己的一些理解进行解答即可。

**解题思路：**

从回答思路上，Spring Boot 的自动配置机制涉及的知识点包括两大部分内容。首先我们需要介绍 @SpringBootApplication
注解及其背后的三大注解 @SpringBootConfiguration、@EnableAutoConfiguration 和
@ComponentScan，这几个注解必须点到。然后，随着这些注解的深入阐述，我们需要再引出 @Import 这个核心注解的应用方式，以及具有与 SPI
实现机制类似功能的 SpringFactoriesLoader 类，该类包含了自动配置相关的一些配置项处理方式。

**本文内容与建议回答：**

本文详细阐述了 Spring Boot 自动配置机制的实现原理，从源码角度分析了为什么 Spring Boot
能够做到自动配置。文中所述的知识体系都可以用来回答这个问题。由于内容比较多，回答过程中还是建议需要把握节奏，避免过多嵌入细节。

### 小结与预告

本文介绍了 Spring Boot 框架中的自动配置机制的相关知识点，这是我们实现对 Spring Boot 框架的扩展，以及完成自定义自动 starter
的基础。在介绍完了自动配置之后，下一篇中，我们就以 MyBatis Spring Boot Starter 为例来看这些知识点在开源框架中的具体应用。

