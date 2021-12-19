Dubbo中应用微内核模式的方法也是基于SPI的思想，但采用了一套新的实现方式，并添加了一些扩展功能。

首先，Dubbo提供了几个扩展点注解，这些注解在介绍《基于架构演进过程剖析代码结构：Dubbo框架的架构演进过程分析》时已经有所提及，但没有具体展开，今天就让我们一起来看一下。

### 扩展点注解

Dubbo中与微内核模式相关的注解主要包括@SPI、@Adaptive和@Activate。

#### @SPI注解

Dubbo提供专门的@SPI注解，只有添加@SPI注解的接口类才会去查找扩展点实现。@SPI注解位于com.alibaba.dubbo.common.extension包中，定义如下：

    
    
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE})
    public @interface SPI {
        String value() default "";
    }
    

该注解标识了接口是一个扩展点，可以看到存在一个value属性，该属性用来指定默认适配扩展点的名称。在Dubbo中，微内核模式应用非常广泛，因此随处可以看到@SPI注解的应用场景。例如，我们以前介绍过的Protocol接口定义如下，在该接口上使用的就是@SPI("dubbo")注解，代表该接口是一个扩展点，同时该扩展点的默认实现是DubboProtocol。

    
    
    @SPI("dubbo")
    public interface Protocol 
    

与JDK中的SPI的查找位置META-INF/services/不同，Dubbo中的查找位置包括META-INF/dubbo/和META-
INF/services/，而META-
INF/dubbo/internal/中则定义了各项用于供Dubbo本身使用的内部扩展。举例来说，在传输协议上，Dubbo对前面提到的Protocol提供了Hessian、Dubbo等多种实现，Dubbo内部通过扩展点的配置确定使用何种机制。在dubbo-
rpc-default工程和dubbo-rpc-hessian工程中，我们在META-
INF/dubbo/internal/目录下都发现了很多配置文件，其中都包含了com.alibaba.dubbo.rpc.Protocol配置文件。dubbo-
rpc-default工程的代码结构如下所示：

![19.01](https://images.gitbook.cn/2020-05-25-052638.png)

而dubbo-rpc-hessian工程中，同样存在类似的代码结构：

![19.02](https://images.gitbook.cn/2020-05-25-052640.png)

我们分别打开这两个工程的com.alibaba.dubbo.rpc.Protocol配置文件，可以发现分别指向了com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol和com.alibaba.dubbo.rpc.protocol.hessian.HessianProtocol类，如下所示：

    
    
    //dubbo-rpc-default工程：
    dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol
    
    //dubbo-rpc-hessian工程
    hessian=com.alibaba.dubbo.rpc.protocol.hessian.HessianProtocol
    

这个意味着当我们引用这两个工程中的某一个具体工程时，通过该工程中的配置项就可以找到相应的扩展点实现。同时，我们从上述配置项中也看出，Dubbo中采用的定义方式与JDK中不一样。Dubbo中使用的一个key值（如上面的dubbo和hession）来指定具体的配置项名称，而不是像JDK中所采用的完整类路径。

#### @Adaptive注解

从字面意思而言，Adaptive就是适配的意思，所以@Adaptive注解的作用就是找到能够适配的扩展。与@SPI注解只能用于类上不同，@Adaptive注解可以同时用在类和方法上。如果@Adaptive注解在类上，那么这个类就是缺省的适配扩展；如何该注解用到扩展点接口的方法上时，那么dubbo就会动态的生成一个这个扩展点的适配扩展类。事实上，在Dubbo中，绝大多数的@Adaptive注解都在用在方法上的，该注解的定义如下：

    
    
    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE, ElementType.METHOD})
    public @interface Adaptive {
    
        String[] value() default {};
    }
    

可以看到@Adaptive注解的value属性可以接收一个字符串数组，意味着我们可以为该注解添加多个值用于进行扩展点的适配。

同样，我们来看一下Dubbo中使用该注解的示例。这次，让我们找到com.alibaba.dubbo.remoting包下的Transporter接口，如下所示。

    
    
    @SPI("netty")
    public interface Transporter {
    
        @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
        Server bind(URL url, ChannelHandler handler) throws RemotingException;
    
        @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
        Client connect(URL url, ChannelHandler handler) throws RemotingException;
    }
    

这里，我们看到在bind和connect这两个方法上都添加了@Adaptive注解。同时，这两个注解都包含了两个键值。关于这两个值的使用方法，我们下文中介绍ExtensionLoader时会具体展开。

#### @Activate注解

在扩展点的实现类上，@Activate注解表示了一个扩展类被获取到的的条件和时机。如果符合条件就被获取，不符合条件就不获取。相较@SPI注解和@Adaptive注解，@Activate显得比较复杂，定义如下：

    
    
    @Documented
    @Retention(RetentionPolicy.RUNTIME)
    @Target({ElementType.TYPE, ElementType.METHOD})
    public @interface Activate {
    
        String[] group() default {};
        String[] value() default {};
        String[] before() default {};
        String[] after() default {};
        int order() default 0;
    }
    

@Activate注解同样可以作用在类型和方法上。基于上述定义，我们可以根据@Activate中的group、value等属性来过滤扩展功能。

在Dubbo中，@Activate注解的一大应用场景是过滤器（Filter）。因为每个过滤器实现了不同维度的功能，因此可以通过该注解进行筛选。例如，在如下所示的ExecuteLimitFilter中（该类主要用于对执行线程进行限制），我们@Activate注解指定了两个筛选条件，表示该过滤器只在生产者（对应的group为Constants.PROVIDER）中生效，而传入参数中应该包含指定的键值（对应的value为Constants.EXECUTES_KEY）。

    
    
    @Activate(group = Constants.PROVIDER, value = Constants.EXECUTES_KEY)
    public class ExecuteLimitFilter implements Filter {
    
        public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
             //省略了方法体实现
        }
    }
    

同样，@Activate注解也与整个扩展点加载机制关系密切。接下来，就让我们深入探讨Dubbo中的扩展点加载机制，即ExtensionLoader。

### ExtensionLoader

ExtensionLoader是实现扩展点加载的核心类，如果我们想要获取DubboProtocol的实现类，那么可以通过以下基于ExtensionLoader的方式获取该类的实例：

    
    
    DubboProtocol dubboProtocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(DubboProtocol.NAME); 
    

我们来看一下getExtension()方法的细节，该方法代码如下所示：

    
    
    public T getExtension(String name) {
            if (name == null || name.length() == 0)
                throw new IllegalArgumentException("Extension name == null");
            if ("true".equals(name)) {
                return getDefaultExtension();
            }
            Holder<Object> holder = cachedInstances.get(name);
            if (holder == null) {
                cachedInstances.putIfAbsent(name, new Holder<Object>());
                holder = cachedInstances.get(name);
            }
            Object instance = holder.get();
            if (instance == null) {
                synchronized (holder) {
                    instance = holder.get();
                    if (instance == null) {
                        instance = createExtension(name);
                        holder.set(instance);
                    }
                }
            }
            return (T) instance;
    }
    

我们看到这里用到了缓存机制，该方法会首先检查缓存中是否已经存在扩展点实例，如果没有则通过createExtension方法进行创建。我们一路跟踪createExtension方法，发现它又调用了getExtensionClasses，而getExtensionClasses内部又使用了loadExtensionClasses方法，在loadExtensionClasses方法中，我们终于看到了熟悉的SPI，该方法代码如下所示：

    
    
    private Map<String, Class<?>> loadExtensionClasses() {
            final SPI defaultAnnotation = type.getAnnotation(SPI.class);
            if (defaultAnnotation != null) {
                 //确定缓存名称
            }
    
            Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
            loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
            loadFile(extensionClasses, DUBBO_DIRECTORY);
            loadFile(extensionClasses, SERVICES_DIRECTORY);
            return extensionClasses;
    }
    

在这里，我们看到了调用了三次loadFile方法，分别从META-INF/dubbo/、META-INF/services/和META-
INF/dubbo/internal/加载扩展点。在loadFile方法中，可以看到，Dubbo是直接通过Class.forName()的形式加载这些SPI的扩展类，并进行缓存。

类似的，在ExtensionLoader中还存在一个getAdaptiveExtension()方法，它的使用方式如下所示：

    
    
    Protocol Protocol = 
    ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension(); 
    

我们注意到getAdaptiveExtension()这个方法的命名应该来源于@Adaptive注解。ExtensionLoader注入的扩展点是一个Adaptive实例，直到扩展点方法执行时才决定调用哪一个扩展点实现。

这里有必要简单介绍一下Dubbo中的URL机制。在Dubbo中，所有Bean对象都会被转换成URL格式作为统一的数据模型。在格式上，URL中包含了很多类似“key=value&key=value”的Key-
Value对。Dubbo使用URL对象传递配置信息，扩展点方法调用会根据URL参数中包含的Key-
Value实现自适应，而URL的Key通过@Adaptive注解在接口方法上提供。我们可以把这种设计方式称为"URL总线"，即很多参数通过URL对象来传递，在实际中，具体要用到哪个值，可以通过URL中的参数值来指定。

回到我们前面介绍@Adaptive注解时看到的Transporter接口，对于如下所示的bind方法定义，Adaptive实现先查找Constants.SERVER
_KEY （即“server”）值，如果该Key没有值则找Constants.TRANSPORTER_ KEY
（即“transport”）值，从而决定加载哪个具体扩展点：

    
    
    @SPI("netty")
    public interface Transporter {
    
        @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
        Server bind(URL url, ChannelHandler handler) throws RemotingException;
    
        ...
    }
    

而在Dubbo配置模块中，扩展点均有对应配置属性或标签，通过配置指定使用哪个扩展实现，如即代表应该获取dubbo协议扩展点，在调用过程中，Dubbo会在URL中自动加入相应的Key-
Value对。

ExtensionLoader还有另外一个核心方法，即getActivateExtension()方法，对应于@Activate注解。翻看该方法的源代码，其内部也使用了介绍getExtension()方法时提到过的getExtensionClasses()方法，所以可以认为getExtension()方法是getAdaptiveExtension()和getActivateExtension()这两个方法的基础。

作为总结，对于Dubbo的整体架构而言，微内核作为一种架构风格只负责组装功能而所有功能通过扩展点实现。Dubbo自身功能也是基于这一机制实现，所有功能点都可被用户自定义扩展和替换。所有扩展点定义都通过传递携带配置信息的URL总线在运行时传入Dubbo，确保整个框架采用一致性数据模型。

### 面试题分析

#### Dubbo中实现的SPI机制与JDK中默认的方式有什么不同？

  * 考点分析

这个问题也比较典型，在实际面试过程中经常被问到。从这个问题中，我们也需要明确，所谓的SPI机制只是一种设计理念，而具体的实现策略视框架的不同会有所差别。

  * 解题思路

一方面，从Dubbo配置项定义中我们已经发现Dubbo中采用了与JDK不同的实现机制。另一方面，虽然Dubbo也采用了SPI机制，也是从JAR中加载对应的扩展，但它的实现方式与JDK中基于ServiceLoader是不一样的。我们需要从这两方面的差异点出发来梳理这个问题的解答思路。

  * 本文内容与建议回答

基于本文中给出的内容，从加载SPI实例的配置文件位置而言，Dubbo支持更多的加载路径。同时，Dubbo采用的是直接通过名称对应的Key值来定位具体的实现类，而ServiceLoader内部使用的是一个迭代器，在获取目标接口的实现类时只能通过遍历的方式把配置文件中的类全部加载并实例化，显然效率比较低下。Dubbo中提供与JDK不同的SPI实现机制的主要目的就是克服这种效率低下的情况，并提供更多的灵活性。

#### Dubbo中@Adaptive和@Activate注解有什么区别？

  * 考点分析

@Adaptive和@Activate这两个注解看上去比较类似，有点容易混淆，因此比较适合作为一个面试题来考查对Dubbo中核心注解的理解程度。实际上，这两个注解关注的是两个不同的维度，本身并没有什么联系。

  * 解题思路

解题思路上，我们需要分别明确这两个注解的作用。对于@Adaptive注解而言，它用于找到能够适配的扩展，如果作用于类上，那么这个类就是缺省的适配扩展；如果作用于扩展点接口的方法上，那么dubbo就会动态的生成一个这个扩展点的适配扩展类。而@Activate注解提供的是一种匹配机制，只有满足通过该注解所涉及的条件和时机时，扩展类才会进行获取。

  * 本文内容与建议回答

本文中对@Adaptive和@Activate这两个注解都进行了明确的说明，也给出在Dubbo中的具体应用场景。结合解题思路中的描述，我们先明确这两个注解的作用，然后用自己的语言进行展开即可。

### 日常开发技巧

如果我们想在Java语言环境中自己实现一套基于SPI机制的微内核架构，我们可以基于前面问题中所分析的Dubbo的SPI机制与JDK中的不同点来优化ServiceLoader类，Dubbo中实现的ExtensionLoader可以为我们提供很多有用的代码模板。

### 小结与预告

微内核架构的作用是满足系统的扩展性，对于Dubbo而言，内置的很多组件都基于SPI机制提供了扩展点。本文对Dubbo中如何基于SPI机制实现做了详细介绍，下一篇我们就来看一下Dubbo中的各种扩展点。

