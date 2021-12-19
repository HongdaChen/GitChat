今天，我们在深入了解了 Spring 框架的扩展点之后，来解析 Dubbo 中的启动过程。要讨论 Dubbo 的启动方法，我们先来简单回顾一下基于
Spring 容器的下的 Dubbo 使用过程，这里以典型的服务端配置为例：

    
    
    <!-- 提供方应用信息，用于计算依赖关系 -->  
    <dubbo:application name="demo-provider"/>  
    <!-- 用 dubbo 协议在 20880 端口暴露服务 -->  
    <dubbo:protocol name="dubbo" port="20880"/>  
    <!-- 使用 zookeeper 注册中心暴露服务地址 -->  
    <dubbo:registry address="zookeeper://127.0.0.1:1234" id="registry"/>  
    <!-- 默认的服务端配置 -->  
    <dubbo:provider registry="registry" retries="0" timeout="5000"/>  
    <!-- 和本地 bean 一样实现服务 -->  
    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl"/>  
    <!-- 声明需要暴露的服务接口 -->  
    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>
    

可以看到这里暴露了一个 DemoService，而客户端引用这个 DemoService 的过程也很简单，典型的客户端配置如下所示：

    
    
    <!-- 消费方应用信息 -->
    <dubbo:application name="consumer" default="false"/>
    <!-- 连接注册中心配置 -->
    <dubbo:registry address="zookeeper://127.0.0.1:2181" protocol="zookeeper" check="false"/>
    <!-- 生成远程服务代理，可以和本地 bean 一样使用 demoService。check="false" 启动时不检查依赖是否已经启动 -->
    <dubbo:reference id=" demoService " interface=" com.alibaba.dubbo.demo.DemoService "  check="false"/>
    

通过《基于架构演进过程剖析代码结构：Dubbo 框架的架构演进过程分析》中的内容，我们知道 Dubbo 通过代理的方式实现了透明化远程调用，即当使用
Dubbo
运行远程服务调用时，我们所做的事情就是在配置文件或代码中添加对某个服务的引用，整个过程让人感觉并没有执行任何与远程方法调用相关的网络连接、数据传输、序列化等操作，就像在调用一个普通的本地方法一样。这点从上述的
Spring 配置中可见一斑。

关于在 Dubbo 中如何实现动态代理以及 RPC
相关的网络连接、数据传输、序列化等操作，我们会在课程后续内容针对各个主题进行详细展开，今天我们关注的核心点是 Dubbo 的启动过程。通过学些 Dubbo
框架的启动原理，我们即可以更加深入理解上一篇中介绍的 Spring 中的各个启动扩展点在 Dubbo 这一特定框架中的应用，也可以学习 Dubbo
在启动过程中使用到的一些特定技巧。

针对上述 application、protocol、registry、provider、service 等服务器端和客户端的配置项，Dubbo
提供了一套解析方法和过程来完成配置项到具体实现类的转变过程。这个过程值得我们作为专门的一个主题进行讲解，这个主题我们安排在后续的《自定义配置标签：Dubbo
中的自定义标签体系》一文进行讨论，这里不做详细展开。今天我们只需要知道 Dubbo 提供了专门的 DubboNamespaceHandler
来完成各个配置项的解析，代码如下所示：

    
    
    public class DubboNamespaceHandler extends NamespaceHandlerSupport {
    
        public void init() {
            registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
            registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
            registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
            registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
            registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
            registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
            registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
            registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
            registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
            registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
        }
    }
    

请注意，从命名上看，其他大部分 Bean 以“Config”结尾，只用于提供 Dubbo 的运行参数。而 ServiceBean 和
ReferenceBean 则不同，它们分布充当了服务发布和服务引用的入口。我们先从 ServiceBean 切入，探究 Dubbo 服务器端的启动方法。

### 服务器端启动方法

ServiceBean 位于 dubbo-config-spring 代码工程的 com.alibaba.dubbo.config.spring
包中，先来看一下这个 Bean 的类定义：

    
    
    public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware {
    
        public void afterPropertiesSet() {}
        ...
        public void setApplicationContext(ApplicationContext applicationContext) {}
        ...
        public void setBeanName(String name) {}
        ...
        public void onApplicationEvent(ApplicationEvent event) {}
        ...
        public void destroy() {}
    }
    

可以看到 ServiceBean 实现了 Spring 的 InitializingBean、DisposableBean
ApplicationContextAware 和 ApplicationListener 等接口，重写了
afterPropertiesSet()、destroy()、setApplicationContext()、onApplicationEvent()
等扩展方法。这些方法就是 Dubbo 与 Spring 整合的关键，一般我们自己设计一个框架基本都是通过这几个接口和 Spring 进行整合。

通过前面《系统启动和初始化方法：Spring
中的启动扩展点》中的分析，我们已经对这些接口和方法的应用方式已以及基本原理有足够的理解。对于这些扩展点而言，Dubbo 就相当于是 Spring
的扩展方。我们来看一下 Dubbo 中如何利用了这些扩展点来完成与 Spring 的整合。首先关注 InitializingBean 接口的
afterPropertiesSet() 方法，这个方法非常长，但结构并不复杂，我们采用框架代码的方式展示该方法的结构，如下所示：

    
    
    public void afterPropertiesSet(){    
        getProvider();
        getApplication()
        getModule();
        getRegistries();
        getMonitor();
        getProtocols();
        getPath();
        if (!isDelay()) {
            export();
        }
    }
    

这个框架代码中的 getProvider、getApplication、getModule、getMonitor 的执行逻辑和流程基本一致。以
Provider 为例，Dubbo 首先会从配置文件中读取 <dubbo:provide> 配置项。如果配置文件中没有定义这些配置项，相当于获取的
ProviderConfig 对象为空，那么就尝试从 BeanFactory 中去获取 ProviderConfig
对象。这里会校验获取的对象的唯一性，显然，Provider、Application 和 Module 在 Dubbo 中应该只能出现一次。

对于注册中心和传输协议而言，Dubbo 中允许存在多个配置中心，一个服务也可以通过多种协议暴露给消费者，所以 RegistryConfig 和
ProtocolConfig 都被保存为一个数组。而 path 属性存放的是<dubbo: service> 配置项中的 id 属性，也就是
BeanName。最后，如果没有启用延迟暴露（ProviderConfig 中可以进行配置）机制，则调用 export 方法暴露服务。

接下来我们关注 ServiceBean 中 ApplicationListener 接口的实现，该接口的 onApplicationEvent
方法如下所示（简单起见代码做了裁剪）：

    
    
    public void onApplicationEvent(ContextRefreshedEvent event) {
            if (isDelay() && !isExported() && !isUnexported()) {           
                export();
            }
    }
    

可以看到 Dubbo 的 ServiceBean 类从 ApplicationContext 监听 ContextRefreshedEvent
事件。我们在《系统启动和初始化方法：Spring 中的启动扩展点（下）》中介绍到，当调用 AbstractApplicationContext 的
refresh() 方法时，Spring 会发送 ContextRefreshedEvent 事件，Dubbo 接收到该事件之后就会执行 export()
方法。

ServiceBean 类中，另一个值得介绍的扩展点实现是 ApplicationContextAware 接口的
setApplicationContext 方法。该方法的核心代码实际上就是以下三句：

    
    
    this.applicationContext = applicationContext;       SpringExtensionFactory.addApplicationContext(applicationContext);
    supportedApplicationListener = true;
    

一方面 Dubbo 会在 ServiceBean 类内部保存所传入的 applicationContext，同时，Dubbo 还构建了一个
SpringExtensionFactory，并把所传入的 applicationContext
进行保存以供其他类进行使用。SpringExtensionFactory 位于
com.alibaba.dubbo.config.spring.extension 包中，如下所示：

    
    
    public class SpringExtensionFactory implements ExtensionFactory {
    
        private static final Set<ApplicationContext> contexts = new ConcurrentHashSet<ApplicationContext>();
    
        public static void addApplicationContext(ApplicationContext context) {
            contexts.add(context);
        }
    
        public static void removeApplicationContext(ApplicationContext context) {
            contexts.remove(context);
        }
    
        @SuppressWarnings("unchecked")
        public <T> T getExtension(Class<T> type, String name) {
            for (ApplicationContext context : contexts) {
                if (context.containsBean(name)) {
                    Object bean = context.getBean(name);
                    if (type.isInstance(bean)) {
                        return (T) bean;
                    }
                }
            }
            return null;
        }
    }
    

SpringExtensionFactory 实现了 ExtensionFactory 接口的 getExtension 方法，它提供了
addApplicationContext 和 removeApplicationContext 静态方法来添加和移除
ApplicationContext，并能根据名称和类型获取所需要的 Bean。这也是使用 applicationContext 的一种常用技巧。

同时，我们发现在 setApplicationContext 方法中还设置了一个 supportedApplicationListener 参数，当可以获取
applicationContext 时，这个参数就是 true。我们寻找这个参数的使用场景，发现它在如下所示的 isDelay 方法中被使用到：

    
    
    private boolean isDelay() {
            Integer delay = getDelay();
            ProviderConfig provider = getProvider();
            if (delay == null && provider != null) {
                delay = provider.getDelay();
            }
            return supportedApplicationListener && (delay == null || delay == -1);
    }
    

isDelay 方法用于判断是否启用延迟暴露机制，通过 supportedApplicationListener 属性可以看到服务延迟暴露与 Spring
容器的 ApplicationListener 有关，这点在前面介绍 ApplicationListener 的 onApplicationEvent
方法时已经明确。

我们在回到 ServiceBean 的定义，发现该类实际上是 ServiceConfig 的子类。而暴露服务的 export 方法就位于
ServiceConfig 类中。而在 export 中，实际上执行服务暴露的方法是 doExport，而在这个方法中，Dubbo 对
application、registry、protocol 等做了很多校验工作，这些工作都与主流程无关，真正核心的是 doExportUrls()
方法，如下所示：

    
    
    private void doExportUrls() {
            List<URL> registryURLs = loadRegistries(true);
            for (ProtocolConfig protocolConfig : protocols) {
                doExportUrlsFor1Protocol(protocolConfig, registryURLs);
            }
    }
    

doExportUrls 首先遍历 ServiceBean 的所有注册中心的配置信息，这些配置属性最终转换成
URL。然后遍历配置的所有协议，并根据每个协议向注册中心暴露服务。这样 Dubbo
的服务端就正式进入到服务暴露的流程，这部分流程不是这节课的重点，我们会在分布式服务架构模块中专门讨论 Dubbo 中服务暴露的实现原理。至此，Dubbo
服务器端启动过程介绍完毕。

### 客户端启动方法

有了 Dubbo 服务器端的理解基础，我们再看 Dubbo 的客户端就会变得比较简单明了。Dubbo 的客户端启动方法需要参考 ReferenceBean
类，该类的类定义如下所示：

    
    
    public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean {
        ...
    }
    

可以看到，ReferenceBean 实现了 Spring 提供的
FactoryBean、ApplicationContextAware、InitializingBean 和 DisposableBean
扩展点。这次，我们先挑最简单的进行介绍。这里的 ApplicationContextAware 接口的 setApplicationContext
方法同样只是把传入的 applicationContext 同时保存在 ReferenceBean 内部以及 SpringExtensionFactory
中。

接下来，我们同样关注 InitializingBean 接口的 afterPropertiesSet() 方法，这个方法同样非常长，但结构同样不复杂，而且与
ServiceBean 中的 afterPropertiesSet
结构比较对称。这里，我们不在该方法的框架代码，而是直接来到该方法的末端，试图找到该方法中最核心的代码。显然，这句最核心的代码就是：

    
    
    public void afterPropertiesSet() throws Exception {
        ...
        getObject();
        ...
    }
    

这个 getObject 方法实际上并不是 ReferenceBean 自身的代码，而是实现了 FactoryBean 接口中的接口方法。这里的
FactoryBean 接口前面没有介绍过，它的定义如下所示：

    
    
    public interface FactoryBean<T> {
        T getObject() throws Exception;
        Class<?> getObjectType();
        boolean isSingleton();
    }
    

实际上，FactoryBean 是 Spring 框架中非常核心的一个接口，负责从容器中获取具体的 Bean 对象。我们重点来看 ReferenceBean
中的 getObject() 方法，该方法又调用了 ReferenceBean 的父类 ReferenceConfig 中的 get() 方法，如下所示：

    
    
    public synchronized T get() {
            if (destroyed) {
                throw new IllegalStateException("Already destroyed!");
            }
            if (ref == null) {
                init();
            }
            return ref;
    }
    

显然，这里的核心应该是 init() 方法。这个 init() 与 ServiceConfig 中的 export
方法一样，做了非常多的准备和校验工作，最终来到了如下代码：

    
    
    ref = createProxy(map);
    

顾名思义，createProxy 方法用来创建代理对象，但并没有那么简单。通过这个 createProxy 方法，Dubbo
的客户端正式进入了节点注册和服务发现流程，并进行 Invoker 实例的构建以及合并。我们同样会在分布式服务架构模块中专门讨论 Dubbo
中服务引用的实现原理。至此，Dubbo 客户达端启动过程介绍完毕。

### 面试题分析

#### **Dubbo 框架如何完成与 Spring 框架的集成？**

**考点分析：**

这道题问的非常大，如果没有前面两篇关于 Spring 中提供的各种扩展机制的内容介绍，我们很难明白这道题具体的考点是什么。事实上，Dubbo 框架与
Spring 框架的集成过程完全就是依赖于前面所介绍的这些扩展点机制，只要明白这些扩展点机制，这道题的回答思路是很明确的。

**解题思路：**

Dubbo 中使用了
InitializingBean、DisposableBean、ApplicationContextAware、ApplicationListener、FactoryBean
以及 BeanNameAware 等众多 Spring 扩展点。从回答思路上讲，我们先要对这些扩展点机制做一些简要介绍，然后可以基于 Dubbo 中
ServiceBean 或 ReferenceBean（视自身的熟悉情况）来阐述这些扩展点在 Dubbo 中所起到的作用和实现的过程。

**本文内容与建议回答：**

本文在前面两篇文章的基础上，又引出了 FactoryBea 和 BeanNameAware 等几个常见的 Spring 扩展点。然后结合 Dubbo
框架的启动和初始化过程，分析了这些扩展的具体使用方法。在回答这个问题时，同样要注意的是点到即止，不要过于陷入具体某个扩展点的实现细节，这也是本文在行文组织上的一个考虑点。

### 日常开发技巧

从对 Dubbo 框架的梳理，我们能够学习到的一大技巧是第三方框架与 Spring 框架的整合方法。以 Dubbo 为例，它实现了一大批 Spring
框架的扩展点，从而充分利用了 Spring 框架中所具备的强大而丰富的功能。这在我们设计和实现自己的系统或框架时非常值得借鉴。

同时，我们在文中看到，Dubbo 中的 SpringExtensionFactory 实现了实现了 ExtensionFactory 接口的
getExtension 方法，它提供了 addApplicationContext 和 removeApplicationContext
静态方法来添加和移出 ApplicationContext，并能根据名称和类型获取所需要的 Bean。这同样也是我们在自己的代码中使用
ApplicationContext 的一种常用技巧。

### 小结与预告

本文基于 Dubbo 框架讲解了 Spring 中多种扩展点的应用方式，从而完成这两个框架之间的集成过程。下一篇中，我们将把视角转到 MyBatis
框架，分析 MyBatis 利用 Spring 扩展点的具体方法。

