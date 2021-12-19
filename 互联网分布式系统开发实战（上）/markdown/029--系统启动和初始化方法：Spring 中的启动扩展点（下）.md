上文中我们介绍了 InitializingBean、BeanPostProcessor 和 Aware 这三个 Spring
框架所提供的扩展点，今天我们继续围绕 Spring 框架介绍它的事件发布和监听机制。相比上文中介绍的几个扩展点，这种扩展点实现机制更加灵活，但也更加复杂。

对于一个事件驱动架构而言，在结构上与设计模式中的观察者模式比较类似，基本上都采用了如下所示的组件结构：

![30.01](https://images.gitbook.cn/2020-05-25-052616.png)

上图中 Event 对应于观察者模式中的主题，事件源发生某事件是特定事件监听器被触发的原因。事件监听器 EventListener
对应于观察者模式中的观察者，监听器监听特定事件，并在内部定义了事件发生后的响应逻辑。而事件发布器 EventPublisher
充当事件监听器的容器，对外提供发布事件和增删事件监听器的接口，维护事件和事件监听器之间的映射关系，并在事件发生时负责通知相关监听器。接下来，我们看一下
Spring 中对整个事件驱动架构的抽象和实现过程。

### Spring 中的事件发布和监听机制

如果 Spring 容器中的某一个 Bean 实现了 ApplicationListener 接口，那么每当 ApplicationContext 发布
ApplicationEvent 时，这个 Bean 将自动触发监听器处理程序。ApplicationListener 接口定义如下：

    
    
    public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    
        void onApplicationEvent(E event);
    }
    

有事件的监听器，就有事件的发送器。在 Spring 中，事件发送器由 ApplicationEventPublisher 接口进行表示，该接口如下所示：

    
    
    public interface ApplicationEventPublisher {
    
        void publishEvent(ApplicationEvent event);
    }
    

请注意 ApplicationContext 接口本身就继承了这个 ApplicationEventPublisher 接口，意味着所有
ApplicationContext
的实现类都具有发送事件的能力。我们可以通过一个简单的代码示例来完成整个事件发送和消费的流程。首先，我们来定义一个用户注册事件
UserRegisterEvent，如下所示：

    
    
    public class UserRegisterEvent extends ApplicationEvent {
            private String userName;
            private String password;
    
            public UserRegisterEvent(Object source, String userName, String password) {
            super(source);
            this.userName = userName;
            this.password = password;
        }
    }
    

然后我们实现一个 Service 层组件 UserService，如下所示：

    
    
    @Service
    public class UserService implements ApplicationContextAware {
    
        private ApplicationContext applicationContext;
        public void regiter(String userName, String password){
            applicationContext.publishEvent(new UserRegisterEvent(this, userName, password));
        }
    
        @Override
        public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
            this.applicationContext = applicationContext;       
        }
    }
    

这里我们使用了前面介绍的 ApplicationContextAware 类注入 ApplicationContext，然后通过
ApplicationContext 的 publishEvent 方法发送了事件。

接下来我们通过使用 @EventListener 注解来监听 UserRegisterEvent，如下所示：

    
    
    @Component
    public class AnnotationRegisterListener {
    
        @EventListener(UserRegisterEvent.class)
        public void register(UserRegisterEvent userRegisterEvent){
            String userName = userRegisterEvent.getUserName();
            String password = userRegisterEvent.getPassword();
            System.out.println("@EventListener 注册信息，用户名："+userName+"，密码："+password);
        }
    }
    

上诉代码示例中我们使用了自定义的 UserRegisterEvent，事实上，Spring 框架中已经内置了一批常见的事件，其中最常用的就是
ContextRefreshedEvent。当 ApplicationContext 被初始化或刷新时，ContextRefreshedEvent
事件就会被触发并进行发布。我们在 AbstractApplicationContext 的 refresh() 方法看到有一个 finishRefresh
方法，该方法如下所示：

    
    
    protected void finishRefresh() {
        initLifecycleProcessor();
        getLifecycleProcessor().onRefresh();
    
        // 发布 ContextRefreshedEvent 事件
        publishEvent(new ContextRefreshedEvent(this));
        LiveBeansView.registerApplicationContext(this);
    }
    

可以看到这里通过 `publishEvent(new ContextRefreshedEvent(this))` 语句发布了一个
ContextRefreshedEvent。当然，我们也可以在 ConfigurableApplicationContext 接口中直接使用
refresh() 方法来触发该事件。

另外诸如 ContextStartedEvent、ContextStoppedEvent、ContextClosedEvent 等事件会在 Spring
容器的生命周期处于启动、停止以及关闭阶段自动触发。如果我们对这些事件感兴趣，就可以直接实现 ApplicationListener
接口从而对引入的具体某一个事件进行响应。例如，如果我们想对 ContextRefreshedEvent 事件进行响应，可以使用如下的代码实现方法：

    
    
    @Component
    public class TestApplicationListener implements ApplicationListener<ContextRefreshedEvent>{
    
        @Override
        public void onApplicationEvent(ContextRefreshedEvent contextRefreshedEvent) {
            System.out.println(contextRefreshedEvent);
            System.out.println("TestApplicationListener...");
        }
    }
    

另外，Spring 提供了一个 @EventListener 注解来简化事件监听实现过程。使用 @EventListener 注解时，我们可以不需要显式实现
ApplicationListener 接口就可以消费事件。我们已经在前面的代码示例中看到了 @EventListener 注解的使用方法。

### ApplicationListener 实现原理

前面提到，ApplicationContext 接口本身就继承了这个 ApplicationEventPublisher 接口，所以它的核心实现类
AbstractApplicationContext 实现了 publishEvent 方法，如下所示（为了显示简单，代码做了裁剪）：

    
    
    public void publishEvent(ApplicationEvent event) {
        ...
        getApplicationEventMulticaster().multicastEvent(event);
        if (this.parent != null) {
            this.parent.publishEvent(event);
        }
    }
    

显然，对于一个事件而言，可以存在多个消费者。所以上述代码中首先获取了 Spring 中的事件多播器
ApplicationEventMulticaster，并调动它的事件多播方法 multicastEvent
实现事件的多播发送。ApplicationEventMulticaster 接口的定义如下：

    
    
    public interface ApplicationEventMulticaster {
    
        void addApplicationListener(ApplicationListener listener); 
        void addApplicationListenerBean(String listenerBeanName); 
        void removeApplicationListener(ApplicationListener listener); 
        void removeApplicationListenerBean(String listenerBeanName); 
        void removeAllListeners(); 
        void multicastEvent(ApplicationEvent event);
    }
    

从该接口的各个方法而言，我们不难明白 ApplicationEventMulticaster 相当于观察者模式中的 Subject，维护着一个
ApplicationListener 列表，并能实现对这些 ApplicationListener 发送事件。

SimpleApplicationEventMulticaster 是 ApplicationEventMulticaster 的最终实现类，它的
multicastEvent 方法实现如下：

    
    
        public void multicastEvent(final ApplicationEvent event) {
            for (final ApplicationListener listener : getApplicationListeners(event)) {
                Executor executor = getTaskExecutor();
                if (executor != null) {
                    //异步回调监听方法
                    executor.execute(new Runnable() {
                        public void run() {
                            listener.onApplicationEvent(event);
                        }
                    });
                }
                else {
                    //同步回调监听方法
                    listener.onApplicationEvent(event);
                }
            }
        }
    

这里先通过 getApplicationListeners 方法（位于 SimpleApplicationEventMulticaster 的父类
AbstractApplicationEventMulticaster 中）从 Context 中获取所注册的
ApplicationListener，如下所示：

    
    
    ListenerRetriever retriever = ...;
    
    synchronized (this.retrievalMutex) {
                    retriever = this.retrieverCache.get(cacheKey);
                    if (retriever != null) {
                        return retriever.getApplicationListeners();
                    }
                    retriever = new ListenerRetriever(true);
                    Collection<ApplicationListener> listeners =
                            retrieveApplicationListeners(eventType, sourceType, retriever);
                    this.retrieverCache.put(cacheKey, retriever);
                    return listeners;
    }
    

上述代码中的 ListenerRetriever 是定义在抽象类 AbstractApplicationEventMulticaster
中的内部成员，用来用来保存所有事件监听器。然后再通过 JDK 提供的 Executor 为每个 ApplicationListener
启动一个独立的线程，并在该线程中回调了 onApplicationEvent 方法。

我们已经明白了事件发送的独立过程，现在回到 Spring 容器来获取用于事件发送的 ApplicationEventMulticaster
以及各个用于事件监听的 ApplicationListener 的具体方式。让我们再次回到已经多次强调的
AbstractApplicationContext 中的 refresh() 方法，这其中有两个步骤与事件相关，即
initApplicationEventMulticaster() 和 registerListeners()。

    
    
        public void refresh() throws BeansException, IllegalStateException {
            synchronized (this.startupShutdownMonitor) {
                ...            
                try {
                    ...
                    // 注册用来拦截 Bean 创建的 BeanPostProcessor
                    registerBeanPostProcessors(beanFactory);
                    ...
                    // 初始化自定义事件广播器
                    initApplicationEventMulticaster();
                    // 执行刷新
                    onRefresh();
                    // 注册监听器
                    registerListeners();
                    ...
                }
            }
        }
    

initApplicationEventMulticaster 方法比较简单，先判断容器中是否实现了一个自定义的
ApplicationEventMulticaster，如果有就直接使用；如果没有则创建一个前面介绍的
SimpleApplicationEventMulticaster 并添加到上下文中，如下所示：

    
    
        protected void initApplicationEventMulticaster() {
            ConfigurableListableBeanFactory beanFactory = getBeanFactory();
            //判断 beanFactory 里是否定义了 applicationEventMulticaster，默认是没有的
            if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
                this.applicationEventMulticaster =
                        beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
                //省略 log 处理
            }
            else {
                //创建一个 SimpleApplicationEventMulticaster 并交由容器管理
                this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
                beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
                //省略 log 处理
            }
        }
    

而 registerListeners 方法则获取当前上下文中的所有 ApplicationListener 并添加到
ApplicationEventMulticaster 中，该方法如下所示：

    
    
        protected void registerListeners() {
            for (ApplicationListener<?> listener : getApplicationListeners()) {
                getApplicationEventMulticaster().addApplicationListener(listener);
            }
    
            String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
            for (String listenerBeanName : listenerBeanNames) {
                getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
            }
        }
    

我们继续来看其中的 addApplicationListener 方法，如下所示：

    
    
        public void addApplicationListener(ApplicationListener listener) {
            synchronized (this.retrievalMutex) {
                this.defaultRetriever.applicationListeners.add(listener);
                this.retrieverCache.clear();
            }
        }
    

这里的 defaultRetriever 就是前面提到的用来保存所有事件监听器的 ListenerRetriever。

那么这些 ApplicationListener 是什么时候添加到当前上下文中的呢？显然，最合理的时机就是在 Bean 初始化之后。我们在
AbstractApplicationContext 中找到了一个添加 ApplicationListener 的入口方法
addApplicationListener，如下所示：

    
    
        public void addApplicationListener(ApplicationListener<?> listener) {
            if (this.applicationEventMulticaster != null) {
                this.applicationEventMulticaster.addApplicationListener(listener);
            }
            else {
                this.applicationListeners.add(listener);
            }
        }
    

我们通过反向搜索这个 addApplicationListener 方法的调用关系，发现在 ApplicationListenerDetector 的
postProcessAfterInitialization 方法中触发了这一操作，如下所示：

    
    
        public Object postProcessAfterInitialization(Object bean, String beanName) {
                if (bean instanceof ApplicationListener) {
                    Boolean flag = this.singletonNames.get(beanName);
                    if (Boolean.TRUE.equals(flag)) {
                        addApplicationListener((ApplicationListener<?>) bean);
                    }
                    else if (flag == null) {
                        //省略 log 处理
                        this.singletonNames.put(beanName, Boolean.FALSE);
                    }
                }
                return bean;
        }
    

关于这个 postProcessAfterInitialization 方法我们已经非常熟悉，属于 BeanPostProcessor 的一部分。而
ApplicationListenerDetector 实现类了 MergedBeanDefinitionPostProcessor 接口，后者又继承了
BeanPostProcessor 接口，以为这它也是一个 BeanPostProcessor，符合前面我们对 BeanPostProcessor
的相关认知。我们再次回到 AbstractApplicationContext 的 refresh 方法中，在其内部的
registerBeanPostProcessors 方法中找到了如下语句，即把 ApplicationListenerDetector 添加到了
BeanPostProcessor 处理链中。

    
    
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector());
    

至此，整个 Spring 的时间发布和监听机制以及介绍完毕。作为总结，我们可以通过以下类图描述整个过程中所涉及到的类层结构关系：

![30.02](https://images.gitbook.cn/2020-05-25-052618.png)

### 面试题分析

#### **简要描述 Spring 中的事件发布和监听机制？**

**考点分析：**

这个问题的考点很明确，就是考查面试者对于 Spring
中事件模型的理解。显然，这道题的目的不仅仅是关注于应用层次上的话题，而是需要面试者回答事件模型的内部实现原理。

**解题思路：**

从回答思路上，与 Spring
中其他的扩展点机制一样，我们还是先阐述事件模型的基本概念以及具体的应用方式。通常，对于一个事件处理机制而言，我们都需要说清楚事件的发送过程和接收过程。所以，我们需要对
ApplicationEventPublisher 和 ApplicationListener
这两个核心接口进行展开。具体的展开程度视面试进程进行灵活把握，但事件多播、监听器管理以及与 Spring 中 BeanPostProcessor
之间的关联关系最好都能点到，确保给到面试官非常全面的回复体验。

**本文内容与建议回答：**

本文从内容组织上把整个 Spring
中的事件驱动模型从应用到原理都做了展开，内容还是比较多的，需要大家做一定的记忆工作。本文最后列出的一张类图，也是方便大家进行记忆之用。只要能够用自己的语言把这张类图能够说清楚，包括这道题在内的关于
Spring 事情模型的常见问题应该都能够进行顺利回答。

### 日常开发技巧

通过本文的学习，我们掌握了 Spring
中事件模型的实现过程，这也为实现自定义的时间驱动架构提供了很好的基础，很多实现上的细节可供我们参考。在日常开发过程中，事件驱动是一个常见的需求。在分布式环境下，我们可以通过消息中间件完成这个操作。而在单块系统内部，基于本文中
ApplicationEventPublisher 和 ApplicationListener
这两个核心接口及其各个实现类的理解，我们可以抽象出更加轻量级的、符合具体业务场景的事件处理机制。

### 小结与预告

本文介绍了 Spring 框架中的事件处理模型，事件处理模型也是我们扩展 Spring 框架的一个常用开发技巧。在介绍完了 Spring
中的这些扩展点之后，下一篇中，我们还将基于 Dubbo 框架来分析如何利用这些扩展点完成与 Spring 的集成和扩展。

