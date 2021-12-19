### 引言

Spring Boot 大大简化了我们的开发配置，节省了大量的时间，确实比较方便。但是对于新手来说，如果不了解个中原理，难免会遇到坑。

本文作者将带领大家走近神秘的 Spring Boot，一步步破开它的神秘面纱，探索 Spring Boot 的启动原理。

开发任何基于 Spring Boot 的项目，我们都会使用以下的启动类：

    
    
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    
    @SpringBootApplication
    public class Application {
    
        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }
    }
    

可以看到，Application 类中定义了注解 `@SpringBootApplication`，main 方法里通过
SpringApplication.run 来启动整个应用程序。因此要研究 Spring Boot 的启动原理，我们就需要从这两个地方入手。

### 强大的 SpringBootApplication

首先，我们先来看看 SpringBootApplication 源码是怎么定义这个注解的：

    
    
    /**
     * Indicates a {@link Configuration configuration} class that declares one or more
     * {@link Bean @Bean} methods and also triggers {@link EnableAutoConfiguration
     * auto-configuration} and {@link ComponentScan component scanning}. This is a convenience
     * annotation that is equivalent to declaring {@code @Configuration},
     * {@code @EnableAutoConfiguration} and {@code @ComponentScan}.
     *
     * @author Phillip Webb
     * @author Stephane Nicoll
     * @since 1.2.0
     */
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
    
        /**
         * Exclude specific auto-configuration classes such that they will never be applied.
         * @return the classes to exclude
         */
        @AliasFor(annotation = EnableAutoConfiguration.class, attribute = "exclude")
        Class<?>[] exclude() default {};
    
        /**
         * Exclude specific auto-configuration class names such that they will never be
         * applied.
         * @return the class names to exclude
         * @since 1.3.0
         */
        @AliasFor(annotation = EnableAutoConfiguration.class, attribute = "excludeName")
        String[] excludeName() default {};
    
        /**
         * Base packages to scan for annotated components. Use {@link #scanBasePackageClasses}
         * for a type-safe alternative to String-based package names.
         * @return base packages to scan
         * @since 1.3.0
         */
        @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
        String[] scanBasePackages() default {};
    
        /**
         * Type-safe alternative to {@link #scanBasePackages} for specifying the packages to
         * scan for annotated components. The package of each class specified will be scanned.
         * <p>
         * Consider creating a special no-op marker class or interface in each package that
         * serves no purpose other than being referenced by this attribute.
         * @return base packages to scan
         * @since 1.3.0
         */
        @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
        Class<?>[] scanBasePackageClasses() default {};
    
    }
    

可以看到，除了最基础的注解外，还增加了三个
`@SpringBootConfiguration`、`@EnableAutoConfiguration`、`@ComponentScan`。因此，正如上一篇所讲的一样，我们将
SpringBootApplication 替换成这三个注解也是相同的效果：

    
    
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.SpringBootConfiguration;
    import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
    import org.springframework.context.annotation.ComponentScan;
    
    @SpringBootConfiguration
    @EnableAutoConfiguration
    @ComponentScan
    public class Application {
    
        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }
    }
    

每次我们都写这三个注解比较麻烦，因此我们只写 `@SpringBootApplication` 就行了。

下面，我们分别来介绍这三个注解。

#### SpringBootConfiguration

我们先来看看它的源码：

    
    
    import java.lang.annotation.Documented;
    import java.lang.annotation.ElementType;
    import java.lang.annotation.Retention;
    import java.lang.annotation.RetentionPolicy;
    import java.lang.annotation.Target;
    import org.springframework.context.annotation.Configuration;
    
    @Target({ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Configuration
    public @interface SpringBootConfiguration {
    }
    

它其实就是一个 Configuration，但是 Spring Boot 推荐用 SpringBootConfiguration 来代替
Configuration。

Spring Boot 社区推荐使用 JavaConfig 配置，所以要用到 `@Configuration`。

我们先来看看 SpringMVC 基于 XML 是如何配置的：

    
    
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd"
           default-lazy-init="true">
        <!--bean定义-->
    </beans>
    

而 JavaConfig 的配置是这样的：

    
    
    import org.springframework.boot.SpringBootConfiguration;
    
    @SpringBootConfiguration
    public class WebConfig {
        //bean定义
    }
    

任何标注了 SpringBootConfiguration 或 Configuration 的类都是一个 JavaConfig。

我们再来看看基于 XML 的 Bean 是如何定义的：

    
    
    <bean id="service" class="ServiceImpl">
    
    </bean>
    

而 JavaConfig 的配置是这样的：

    
    
    import org.springframework.boot.SpringBootConfiguration;
    
    @SpringBootConfiguration
    public class WebConfig {
        //bean定义
        @Bean
        public Service service(){
            return new ServiceImpl();
        }
    }
    

任何标注了 Bean 的方法都被定义为一个 Bean，我们可以在任何 Spring 的 IoC 容器中注入进去。

#### EnableAutoConfiguration

这个注解尤为重要，它的作用是自动将 JavaConfig 中的 Bean 装载到 IoC 容器中。

通过分析其源码，我们就能分析出其原理，它的源码如下：

    
    
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @AutoConfigurationPackage
    @Import(AutoConfigurationImportSelector.class)
    public @interface EnableAutoConfiguration {
    
        String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
    
        /**
         * Exclude specific auto-configuration classes such that they will never be applied.
         * @return the classes to exclude
         */
        Class<?>[] exclude() default {};
    
        /**
         * Exclude specific auto-configuration class names such that they will never be
         * applied.
         * @return the class names to exclude
         * @since 1.3.0
         */
        String[] excludeName() default {};
    
    }
    

我们注意到该注解引入了 @AutoConfigurationPackage 注解，通过其字面意思，就知道它的作用是自动配置
Package，即它会默认配置启动类所在包及其子包下的所有标注了 Configuration 注解的类。

而以上注解使用 @Import 注解，该注解的作用是自动执行该注解指定的类。在上述注解中，其导入了
AutoConfigurationImportSelector 类，通过其类名就知道该类的作用是自动配置选择器，因此，我们使用了
@EnableAutoConfiguration 注解后，它就会自动执行 AutoConfigurationImportSelector
类，最终会调用哪个方法呢？请看它的源码：

    
    
    @Override
    public Class<? extends Group> getImportGroup() {
        return AutoConfigurationGroup.class;
    }
    

程序启动后，只要标注了 @EnableAutoConfiguration 注解，那么最后会调用 getImportGroup 方法，它返回的是一个
Group 对象。而上述代码中 AutoConfigurationGroup 继承的是 Group 接口，通过查看 Group 源码得知，Group
接口被定义在 DeferredImportSelector 接口中，继续查看 AutoConfigurationGroup 类的代码，我们发现以下源码：

    
    
    @Override
            public void process(AnnotationMetadata annotationMetadata,
                    DeferredImportSelector deferredImportSelector) {
                Assert.state(
                        deferredImportSelector instanceof AutoConfigurationImportSelector,
                        () -> String.format("Only %s implementations are supported, got %s",
                                AutoConfigurationImportSelector.class.getSimpleName(),
                                deferredImportSelector.getClass().getName()));
                AutoConfigurationEntry autoConfigurationEntry = ((AutoConfigurationImportSelector) deferredImportSelector)
                        .getAutoConfigurationEntry(getAutoConfigurationMetadata(),
                                annotationMetadata);
                this.autoConfigurationEntries.add(autoConfigurationEntry);
                for (String importClassName : autoConfigurationEntry.getConfigurations()) {
                    this.entries.putIfAbsent(importClassName, annotationMetadata);
                }
            }
    
    @Override
            public Iterable<Entry> selectImports() {
                if (this.autoConfigurationEntries.isEmpty()) {
                    return Collections.emptyList();
                }
                Set<String> allExclusions = this.autoConfigurationEntries.stream()
                        .map(AutoConfigurationEntry::getExclusions)
                        .flatMap(Collection::stream).collect(Collectors.toSet());
                Set<String> processedConfigurations = this.autoConfigurationEntries.stream()
                        .map(AutoConfigurationEntry::getConfigurations)
                        .flatMap(Collection::stream)
                        .collect(Collectors.toCollection(LinkedHashSet::new));
                processedConfigurations.removeAll(allExclusions);
    
                return sortAutoConfigurations(processedConfigurations,
                        getAutoConfigurationMetadata())
                                .stream()
                                .map((importClassName) -> new Entry(
                                        this.entries.get(importClassName), importClassName))
                                .collect(Collectors.toList());
            }
    

也就是说最终会自动执行 process 和 selectImports 方法。

可以注意到 autoConfigurationEntry.getConfigurations()，它就是获取所有标注了 @Configuration
注解的类，并加入到 Map 中。

需要注意的是，AutoConfigurationImportSelector 类实现的是 DeferredImportSelector 接口，而
process 是 DeferredImportSelector.Group 接口定义的方法。

#### ComponentScan

这个注解的作用是自动扫描并加载符合条件的组件（如：Component、Bean 等），我们可以通过 basePakcages
来指定其扫描的范围，如果不指定，则默认从标注了 `@ComponentScan` 注解的类所在包开始扫描。如下代码：

    
    
    @ComponentScan(basePackages = "com.lynn")
    

因此，Spring Boot 的启动类最好放在 root package 下面，因为默认不指定 basePackages，这样能保证扫描到所有包。

以上只是从表面来研究 Spring Boot 的启动原理，那么，为什么通过 SpringBootApplication 和
SpringApplication.run() 就能启动一个应用程序，它的底层到底是怎么实现的呢？别急，我们马上来一探究竟。

### 源码解析

我们知道，启动类先调用了 SpringApplication 的静态方法 run，跟踪进去后发现，它会先实例化 SpringApplication，然后调用
run 方法。

    
    
    /**
         * Static helper that can be used to run a {@link SpringApplication} from the
         * specified sources using default settings and user supplied arguments.
         * @param primarySources the primary sources to load
         * @param args the application arguments (usually passed from a Java main method)
         * @return the running {@link ApplicationContext}
         */
        public static ConfigurableApplicationContext run(Class<?>[] primarySources,
                String[] args) {
            return new SpringApplication(primarySources).run(args);
        }
    

所以，要分析它的启动源码，首先要分析 SpringApplicaiton 的构造过程。

#### SpringApplication 构造器

在 SpringApplication 构造函数内部，他会初始化一些信息：

    
    
    public SpringApplication(Class<?>... primarySources) {
            this(null, primarySources);
        }
    
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
            this.resourceLoader = resourceLoader;
            Assert.notNull(primarySources, "PrimarySources must not be null");
            this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
            this.webApplicationType = WebApplicationType.deduceFromClasspath();
            setInitializers((Collection) getSpringFactoriesInstances(
                    ApplicationContextInitializer.class));
            setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
            this.mainApplicationClass = deduceMainApplicationClass();
        }
    

通过上述代码，我们分析到 SpringApplication 实例化时有以下几个步骤：

1.将所有 sources 加入到全局 sources 中，目前只有一个 Application。

2.判断是否为 Web
程序（javax.servlet.Servlet、org.springframework.web.context.ConfigurableWebApplicationContext
这两个类必须存在于类加载器中）。

判断过程可以参看以下源码：

    
    
    static WebApplicationType deduceFromClasspath() {
            if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
                    && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
                    && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
                return WebApplicationType.REACTIVE;
            }
            for (String className : SERVLET_INDICATOR_CLASSES) {
                if (!ClassUtils.isPresent(className, null)) {
                    return WebApplicationType.NONE;
                }
            }
            return WebApplicationType.SERVLET;
        }
    

3.设置应用程序初始化器 ApplicationContextInitializer，做一些初始化的工作。

4.设置应用程序事件监听器 ApplicationListener。

5.找出启动类，设置到 mainApplicationClass 中。

#### SpringApplication 的执行流程

SpringApplication 构造完成后，就会调用 run 方法，这时才真正的开始应用程序的执行。

先来看看源码：

    
    
    public ConfigurableApplicationContext run(String... args) {
            StopWatch stopWatch = new StopWatch();
            stopWatch.start();
            ConfigurableApplicationContext context = null;
            Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
            configureHeadlessProperty();
            SpringApplicationRunListeners listeners = getRunListeners(args);
            listeners.starting();
            try {
                ApplicationArguments applicationArguments = new DefaultApplicationArguments(
                        args);
                ConfigurableEnvironment environment = prepareEnvironment(listeners,
                        applicationArguments);
                configureIgnoreBeanInfo(environment);
                Banner printedBanner = printBanner(environment);
                context = createApplicationContext();
                exceptionReporters = getSpringFactoriesInstances(
                        SpringBootExceptionReporter.class,
                        new Class[] { ConfigurableApplicationContext.class }, context);
                prepareContext(context, environment, listeners, applicationArguments,
                        printedBanner);
                refreshContext(context);
                afterRefresh(context, applicationArguments);
                stopWatch.stop();
                if (this.logStartupInfo) {
                    new StartupInfoLogger(this.mainApplicationClass)
                            .logStarted(getApplicationLog(), stopWatch);
                }
                listeners.started(context);
                callRunners(context, applicationArguments);
            }
            catch (Throwable ex) {
                handleRunFailure(context, ex, exceptionReporters, listeners);
                throw new IllegalStateException(ex);
            }
    
            try {
                listeners.running(context);
            }
            catch (Throwable ex) {
                handleRunFailure(context, ex, exceptionReporters, null);
                throw new IllegalStateException(ex);
            }
            return context;
        }
    

通过上述源码，将执行流程分解如下：

  1. 初始化 StopWatch，调用其 start 方法开始计时。
  2. 调用 configureHeadlessProperty 设置系统属性 java.awt.headless，这里设置为 true，表示运行在服务器端，在没有显示器和鼠标键盘的模式下工作，模拟输入输出设备功能。
  3. 遍历 SpringApplicationRunListeners 并调用 starting 方法。
  4. 创建一个 DefaultApplicationArguments 对象，它持有 args 参数，就是 main 函数传进来的参数调用 prepareEnvironment 方法。
  5. 打印 banner。
  6. 创建 Spring Boot 上下文。
  7. 初始化 FailureAnalyzers。
  8. 调用 prepareContext。
  9. 调用 AbstractApplicationContext 的 refresh 方法，并注册钩子。
  10. 在容器完成刷新后，依次调用注册的 Runners。
  11. 调用 SpringApplicationRunListeners 的 finished 方法。
  12. 启动完成并停止计时。
  13. 初始化过程中出现异常时调用 handleRunFailure 进行处理，然后抛出 IllegalStateException 异常。

