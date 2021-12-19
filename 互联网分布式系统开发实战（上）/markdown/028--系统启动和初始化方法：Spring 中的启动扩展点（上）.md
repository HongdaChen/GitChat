对于一个开源框架而言，探索框架的第一件事就是找到它的系统启动和初始化方法。现代开源软件整体架构往往非常复杂，业界也都不崇尚重复造轮子。因此，一个开源软件本身也会依赖很对其他的开源软件，系统的启动和初始化过程也会与这些软件有交集，甚至于依赖于第三方系统才能完成运作。

在 Java 领域中，Spring 家族是当下最流行的开源生态，包含了诸如 Spring、Spring Boot 等一大批优秀框架。很多开源框架都选择了与
Spring 生态系统进行集成，一方面避免重复造轮子，一方面也为习惯了使用这些主流框架的开发人员降低了学习成本。

Dubbo、Spring Cloud 和 MyBatis 是目前构建分布式系统的主流框架，也是本课程重点介绍的对象。除了 Spring Cloud 本身就是
Spring 家族的一员之外，Dubbo 和 MyBatis 在提供了独立的系统启动和初始化实现方案之外，也与 Spring 家族进行了紧密的集成，利用到了
Spring 框架中的一些核心扩展点。在深入学习这些框架之前，我们有必要对 Spring 中的这些扩展点先做一定的展开。

### InitializingBean 和 DisposableBean

在 Spring 中，InitializingBean 和 DisposableBean 是两个标记接口，为 Spring 执行 Bean
初始化和销毁的某些行为提供有用的方法。

#### **InitializingBean**

顾名思义，InitializingBean 应该是用于初始化 Bean，该接口定义如下：

    
    
    public interface InitializingBean {
        void afterPropertiesSet() throws Exception;
    }
    

我们看到 InitializingBean 接口只有一个方法，即
afterPropertiesSet。从命名上看，这个方法应该作用于属性被设置之后。也就是说， 该方法的初始化会晚于属性的初始化。

实际上，InitializingBean 只是 Spring 初始化时可以采用的其中一个扩展点。与 InitializingBean 类似的一种机制是
init-method。在 Spring 初始化 Bean 时，可以配置 Bean 的 init-method 属性。init-method
背后采用的是反射原理，通过读取配置文件中的 init-method 标签直接构建初始化方法。

然后，我们再来看一下@PostConstruct 注解，这是与 InitializingBean
类似的另一种初始化机制。不难看出，这个注解作用于构造函数之后。

这三种 Spring 初始化扩展机制都非常常见，我们在后面的课程中介绍 Dubbo、Spring Cloud
等框架时会经常遇到。既然它们都能对初始化过程做一定的控制，我们来通过一个示例分析各个机制的执行顺序。示例如下所示：

    
    
    public class TestInitBean implements InitializingBean {
    
        public TestInitBean (){
            System.out.println("constructMethod");
        }
    
        @Override
        public void afterPropertiesSet() throws Exception {
            System.out.println("afterPropertiesSet");
        }
    
        @PostConstruct
        public void postConstruct(){
            System.out.println("postConstruct");
        }
    
        public void initMethod() {
            System.out.println("initMethod");
        }
    }
    
    
    
    <bean class="com.tianyalan.springinitialization. TestInitBean" init-method="initMethod"></bean>
    

上述示例的执行结果如下所示：

    
    
    constructMethod
    postConstruct
    afterPropertiesSet
    initMethod
    

显然，通过上述输出结果，三者的先后顺序也就一目了然了，即 Constructor→ @PostConstruct → InitializingBean →
init-method。

结论已经有了，我们简单对这个结论做源码分析。Spring 中，关于 Bean
初始化的过程非常复杂，对于整个过程的介绍不是本课程的重点，大家可以参考郝佳所著的《Spring 源码深度解析》第 2 版中的内容。我们直接来到
AbstractAutowireCapableBeanFactory 的 initializeBean
方法，这个方法完成了我们这里提到的相关操作，我们对该方法在表现形式上做一些简化，可以得到如下所示的结构：

    
    
    protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    
        //调用 Aware 相关方法
        invokeAwareMethods(beanName, bean);
    
        Object wrappedBean = bean;
        //在初始化之前应用 BeanPostProcessor
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    
        //调用 Init 相关方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    
        //在初始化之后应用 BeanPostProcessor
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    
        return wrappedBean;
    }
    

这里方法我们后面会多次讨论，现在先看这里的
invokeInitMethods，从命名上看该方法的作用是调用一批初始化方法，我们继续对这个方法的结构做一些简化调整以便容易理解：

    
    
    protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd) throws Throwable {
    
            boolean isInitializingBean = (bean instanceof InitializingBean);
            if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
            ((InitializingBean) bean).afterPropertiesSet();
    }
    
            if (mbd != null) {
                String initMethodName = mbd.getInitMethodName();
                if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                        !mbd.isExternallyManagedInitMethod(initMethodName)) {
                    invokeCustomInitMethod(beanName, bean, mbd);
                }
            }
        }
    

可以看到，这里判断该 Bean 是否实现了 InitializingBean 接口，如果是则直接调用它的 afterPropertiesSet()
方法。然后我们根据 Bean 的定义获取它的 init-method，并通过 invokeCustomInitMethod 完成自定义 init-
method 的调用。invokeCustomInitMethod 方法的整体结构如下所示：

    
    
    protected void invokeCustomInitMethod(String beanName, final Object bean, RootBeanDefinition mbd) throws Throwable {
            String initMethodName = mbd.getInitMethodName();
            final Method initMethod = (mbd.isNonPublicAccessAllowed() ?
                    BeanUtils.findMethod(bean.getClass(), initMethodName) :
                    ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));
    
            ReflectionUtils.makeAccessible(initMethod);
            initMethod.invoke(bean);
        }
    

显然，这里通过反射机制找到 init-method 并执行。所以从顺序上讲，init-method 的执行肯定是在 afterPropertiesSet
方法之后。

至于 @PostConstruct 注解的执行顺序，由于涉及到了 PostProcessor 系列的扩展点，我们一会再来介绍。

#### **DisposableBean**

DisposableBean 在 Bean 生命周期结束前调用 destory() 方法做一些收尾工作。与 init-method 一样，Spring
也提供了基于反射机制的 destory-method 以及与 @PostConstruct 注解相对应的@PreDestroy
注解。有了对前面知识的理解，这些概念和原理也都比较类似，这里不再展开。

### BeanPostProcessor

从命名上看，BeanPostProcessor 是 Bean 的一种加工器，可以在 Bean 的实例化过程中对 Bean 做一些包装处理。

#### **BeanPostProcessor 接口和调用方法**

紧跟上节内容，我们再来推导@PostConstruct 注解的执行顺序。我们注意到在 AbstractAutowireCapableBeanFactory
的 initializeBean 中存在一对 applyBeanPostProcessorsBeforeInitialization 和
applyBeanPostProcessorsAfterInitialization 方法。其中
applyBeanPostProcessorsBeforeInitialization 方法如下所示：

    
    
    public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) throws BeansException {
    
            Object result = existingBean;
            for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
                result = beanProcessor.postProcessBeforeInitialization(result, beanName);
                if (result == null) {
                    return result;
                }
            }
            return result;
        }
    

applyBeanPostProcessorsAfterInitialization 的结构与上述方法完全一致，只是调用了每个
BeanPostProcessor 的 postProcessAfterInitialization 方法。实际上 BeanPostProcessor
接口也只包含了这两个方法，如下所示：

    
    
    public interface BeanPostProcessor {
        Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    
        Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
    }
    

#### **BeanPostProcessor 的应用：@PostConstruct 注解**

接下来，我们将通过 @PostConstruct 注解的实现过程来分析 BeanPostProcessor 的应用方式。因为@PostConstruct
是一个注解，所以我们找到专门处理注解的 BeanPostProcessor，即
CommonAnnotationBeanPostProcessor，该类扩展了 InitDestroyAnnotationBeanPostProcessor
类。我们先来看 InitDestroyAnnotationBeanPostProcessor 的
postProcessBeforeInitialization 方法，该方的核心实际上就是以下两句代码：

    
    
    LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
        metadata.invokeInitMethods(bean, beanName);
    

我们获取包含注解的元数据，然后基于这些元数据利用反射技术执行注解方法。其中获取元数据的方法如下所示（为了简单起见，对一些日志代码做了裁剪）：

    
    
    private LifecycleMetadata buildLifecycleMetadata(Class<?> clazz) {
            final boolean debug = logger.isDebugEnabled();
            LinkedList<LifecycleElement> initMethods = new LinkedList<LifecycleElement>();
            LinkedList<LifecycleElement> destroyMethods = new LinkedList<LifecycleElement>();
            Class<?> targetClass = clazz;
    
            do {
                LinkedList<LifecycleElement> currInitMethods = new LinkedList<LifecycleElement>();
                LinkedList<LifecycleElement> currDestroyMethods = new LinkedList<LifecycleElement>();
                for (Method method : targetClass.getDeclaredMethods()) {
                    if (this.initAnnotationType != null) {
                        if (method.getAnnotation(this.initAnnotationType) != null) {
                            LifecycleElement element = new LifecycleElement(method);
                            currInitMethods.add(element);
                        }
                    }
                    if (this.destroyAnnotationType != null) {
                        if (method.getAnnotation(this.destroyAnnotationType) != null) {
                            currDestroyMethods.add(new LifecycleElement(method));
                        }
                    }
                }
                initMethods.addAll(0, currInitMethods);
                destroyMethods.addAll(currDestroyMethods);
                targetClass = targetClass.getSuperclass();
            }
            while (targetClass != null && targetClass != Object.class);
    
            return new LifecycleMetadata(clazz, initMethods, destroyMethods);
        }
    

这里我们是根据 initAnnotationType 这个类型来获取相应的注解方法，该属性由
InitDestroyAnnotationBeanPostProcessor 的 setInitAnnotationType 方法进行传入，如下所示：

    
    
    public void setInitAnnotationType(Class<? extends Annotation> initAnnotationType) {
            this.initAnnotationType = initAnnotationType;
        }
    

而我们 InitDestroyAnnotationBeanPostProcessor 的子类
CommonAnnotationBeanPostProcessor 的构造函数中发现了 setInitAnnotationType 的调用：

    
    
    public CommonAnnotationBeanPostProcessor() {
            setOrder(Ordered.LOWEST_PRECEDENCE - 3);
            setInitAnnotationType(PostConstruct.class);
            setDestroyAnnotationType(PreDestroy.class);
            ignoreResourceType("javax.xml.ws.WebServiceContext");
        }
    

显然，InitDestroyAnnotationBeanPostProcessor 通过传入 PostConstruct.class 类型的
initAnnotationType 来完成用于执行反射机制的元数据的收集和构建。

现在我们终于知道@PostConstruct 注解它是在 initializeBean 的
applyBeanPostProcessorsBeforeInitialization 方法中被执行。显然
applyBeanPostProcessorsBeforeInitialization 方法位于 invokeInitMethods
方法之前，所以它的执行顺序先与 afterPropertiesSet 方法。

### Aware

我们在前面的 AbstractAutowireCapableBeanFactory 类的 initializeBean 中还看到了一个
invokeAwareMethods 方法，而且该方法的执行还在 invokeInitMethods 方法之前。这个 invokeAwareMethods
就涉及到本小结要介绍的 Spring 中所提供的 Aware 系列扩展机制。

#### **Aware 接口示例**

在 Spring 中，Aware 接口是一个空接口，但却具有一大批直接或间接的子接口。比方说，以常见的 ApplicationContextAware
接口为例，定义如下：

    
    
    public interface ApplicationContextAware extends Aware {
    
        void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
    }
    

当一个类实现了 ApplicationContextAware 之后，就可以方便获得当前的（指的是所运行的代码和已启动的 Spring 代码处于同一个
Spring 上下文）ApplicationContext，进而获取 ApplicationContext 中的所有
Bean。ApplicationContextAware 的使用方法也非常简单，直接使用它所提供的 setApplicationContext
方法，并把传入的 ApplicationContext 暂存起来使用。

再如 BeanNameAware，定义如下：

    
    
    public interface BeanNameAware extends Aware {
    
        void setBeanName(String name);
    }
    

当实现 BeanNameAware 的 setBeanName() 方法时，Spring 会将 Bean 的 id 传给 setBeanName() 方法。

同样，在 ApplicationEventPublisherAware 中，我们可以获取 Spring 中的
ApplicationEventPublisher，如下所示：

    
    
    public interface ApplicationEventPublisherAware extends Aware {
    
        void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher);
    }
    

通过前面三个 Aware 的了解，我们发现它们都只有一个方法，且这个方法名都是 setXXX。显然，这些 Aware 接口的目的就是在 Spring
启动之后，如果一个 Bean 想要获取并使用 Spring 容器中的相关对象，我们就不需要再次执行重复的启动过程，而是通过这些 setXXX
引入相关对象即可。

#### **Aware 接口调用过程**

接下来，我们回到前面讨论的 AbstractAutowireCapableBeanFactory 类的 initializeBean
方法。我们看到这个方法执行上的首要一步就是 invokeAwareMethods()，如下所示：

    
    
    private void invokeAwareMethods(final String beanName, final Object bean) {
            if (bean instanceof Aware) {
                if (bean instanceof BeanNameAware) {
                    ((BeanNameAware) bean).setBeanName(beanName);
                }
                if (bean instanceof BeanClassLoaderAware) {
                    ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
                }
                if (bean instanceof BeanFactoryAware) {
                    ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
                }
            }
        }
    

我们看到如果这个 Bean 实现了 BeanNameAware、BeanClassLoaderAware 和 BeanFactoryAware
接口，那么这里就会调用对应的接口方法。

那么问题来了，其他 Aware 是在哪里被调用的呢？这就需要从 Spring 框架中最经典的一个方法说起，这个方法就是在介绍系统扩展性时已经提到过的
AbstractApplicationContext 类中的 refresh() 方法，如下所示：

    
    
        public void refresh() throws BeansException, IllegalStateException {
            synchronized (this.startupShutdownMonitor) {
                // 刷选容器前的准备，包括属性源的初始化
                prepareRefresh();
                // 刷选 BeanFactory，加载 XML 配置文件，生产 BeanDefinition
                ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
                // 准备类加载器、PostProcessor 等容器的一些标准特性
                prepareBeanFactory(beanFactory);
                try {
                    //修改或添加一些 BeanPostProcessor
                    postProcessBeanFactory(beanFactory);
                    // 执行扩展机制 BeanFactoryPostProcessor
                    invokeBeanFactoryPostProcessors(beanFactory);
                    // 注册用来拦截 Bean 创建的 BeanPostProcessor
                    registerBeanPostProcessors(beanFactory);
                    // 初始化消息源
                    initMessageSource();
                    // 初始化自定义事件广播器
                    initApplicationEventMulticaster();
                    // 执行刷新
                    onRefresh();
                    // 注册监听器
                    registerListeners();
                    // 初始化非延迟加载的单例 Bean
                    finishBeanFactoryInitialization(beanFactory);
                    // 完成刷新，发送完成事件
                    finishRefresh();
                }
            }
        }
    

这个方法可以说是整个 Spring 中的最重要的方法，该方法会在整个课程体系中多次被提起，其中所包含的每一个步骤都很关键。我们在
AbstractApplicationContext 中搜索“ApplicationContextAware”字样，发现在 refresh() 方法的
prepareBeanFactory() 方法中存在这样一个语句：

    
    
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    

也就是说，Spring 在启动时会我们添加 ApplicationContextAwareProcessor，它是一个
BeanPostProcessor。结合上一节中关于 BeanPostProcessor 的介绍，我们来看一下该类中
postProcessBeforeInitialization 方法的内容（该类的 postProcessAfterInitialization
不包含处理逻辑）。实际上这个方法中主要就是执行了 invokeAwareInterfaces() 方法，该方法如下所示：

    
    
    private void invokeAwareInterfaces(Object bean) {
            if (bean instanceof Aware) {
                if (bean instanceof EnvironmentAware) {
                    ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
                }
                if (bean instanceof EmbeddedValueResolverAware) {
                    ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(
                            new EmbeddedValueResolver(this.applicationContext.getBeanFactory()));
                }
                if (bean instanceof ResourceLoaderAware) {
                    ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
                }
                if (bean instanceof ApplicationEventPublisherAware) {
                    ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
                }
                if (bean instanceof MessageSourceAware) {
                    ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
                }
                if (bean instanceof ApplicationContextAware) {
                    ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
                }
            }
        }
    

那么这个 ApplicationContextAwareProcessor
是什么时候被调用的呢？事实上，AbstractAutowireCapableBeanFactory 中的 inititalBean() 方法就是
BeanPostProcessor 的调用处。根据前面的介绍，inititalBean() 方法中
applyBeanPostProcessorsBeforeInitialization 就会触发
ApplicationContextAwareProcessor 的 postProcessBeforeInitialization
方法，从而进一步触发上述的 invokeAwareInterfaces 方法。

### 面试题分析

#### **1\. Spring 中 Bean 中的 Constructor、@PostConstruct 以及 InitializingBean
这三者之间的执行顺序是怎么样的？**

**考点分析**

这个一道比较好的面试题，笔者作为面试官也经常会用这道题考查面试者对于 Spring
中一些基本实现机制的理解程度。从考点上讲，一方面这道题的知识点比较丰富，涉及到对 Spring
框架中一些核心概念的理解；另一方面，这道题也具有一定的综合性，要求面试者在理解这些核心概念的基础上能够用自己的表达方式串接起来。

**解题思路**

首先，我们应该对 Spring 中@PostConstruct 注解和 InitializingBean
接口的作用和应用场景有明确的认识，最好有一定的实践基础，在回答这个问题时先说明这一点。然后，在介绍了应用方式的基础上，我们需要深入这些注解和方法的背后，进一步介绍
BeanPostProcessor 等 Spring
内部的实现接口及其实现过程。整个回答过程中涉及的概念和对象可能有点多，注意侧重点和回答问题的节奏。有些地方不用展开的过于细化，只需要围绕“执行顺序”这一核心考点进行展开即可，避免陷入细节而导致问题回答思路上的混乱。

**本文内容与建议回答**

本文从应用方式和实现原理对这个问题的各个方面都做了详细的介绍，其中对 AbstractAutowireCapableBeanFactory 的
initializeBean 方法的介绍是回答该问题的重点，需要大家熟悉该方法内部的几个核心方法。

#### **2\. Spring 中提供 Aware 接口的目的是什么？它是如何实现这一目的的？**

**考点分析**

这也是一道比较经典的问题，因为 Aware 接口作为 Spring 框架中的一个常见接口以及作为我们扩展 Spring
框架的一种常见手段得到了广泛的应用。这道题考查了 Aware
接口的设计目的，换句话说我们能拿这个接口做什么，属于面向应用层的考点。进一步的，该题还考查了实现过程，属于面向跟深层次的原理性的考点。

**解题思路**

解题思路上，我们也比较明确，即 Aware 系列接口的作用是在 Spring 启动之后方便某一个 Bean 获取并使用 Spring
容器中的其他相关对象（最典型的就是
ApplicationContext）。而从实现原理上，它的实现方式也不负载，只是调用实现了这些接口的对应方法而已。所以，整体而言，这道题回答起来思路还是很明确的，不算难题。

**本文内容与建议回答**

本文对 Spring 中与 Aware 接口相关的实现原理做了详细的介绍，这部分内容本身也比较好理解。重点还是需要把控整体的方法调用链关系，所以我们也引出了
Spring 中最为重要的 refresh 方法，试图把所有内容串联起来。

### 日常开发技巧

作为系统启动和初始化相关的常见扩展点，本文中介绍 InitializingBean 接口和 Aware
系列接口可以说应用非常广泛。我们在后续的内容中会发现主流的 Dubbo、MyBatis 等框架都是基于这些扩展性接口完成了与 Spring
框架的整合。如果我们需要实现与 Spring
框架的集成和扩展，这些接口是必定需要掌握的内容，建议在日常开发过程中多关注这些接口的应用场景和方式，并视情况集成到自己的代码当中。

### 小结与预告

本文介绍了 Spring 框架中的几个核心扩展点，这些扩展点是 Dubbo 和 MyBatis 等开源框架与 Spring
进行集成的主要入口。下一篇中，我们还将介绍一个 Spring 框架中非常重要的扩展点，即事件处理机制。

