我们在《基于架构演进过程剖析代码结构：Dubbo 框架的架构演进过程分析》中提到，当使用 Dubbo
运行远程服务调用时，我们所做的事情就是在配置文件或代码中添加对某个服务的引用，典型的用法如下所示：

    
    
    <dubbo:reference id="demoService" interface="com.alibaba.dubbo.demo.DemoService" />
    

整个过程让人感觉并没有执行任何与远程方法调用相关的网络连接、数据传输、序列化/反序列化等操作，就像在调用一个普通的本地方法一样，这就是所谓的远程调用本地化。远程调用本地化背后用到的就是动态代理机制。

在 Dubbo 中，当 Invoker
创建完毕后，接下来要做的事情是为服务接口生成代理对象。有了代理对象，我们就可以通过代理对象进行远程调用。代理对象生成的入口方法为在
ProxyFactory（位于 com.alibaba.dubbo.rpc 包中）的 getProxy，如下所示：

    
    
    @SPI("javassist")
    public interface ProxyFactory {
    
        @Adaptive({Constants.PROXY_KEY})
        <T> T getProxy(Invoker<T> invoker) throws RpcException;
    
        @Adaptive({Constants.PROXY_KEY})
        <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;
    }
    

我们同样再来回顾一下 ProxyFactory 的类层结构，如下所示：

![xog9by](https://images.gitbook.cn/2020-05-25-052615.png)

我们先来看看 AbstractProxyFactory 类，该类的主要目的就是为了增加一个 EchoService.class 接口。在 Dubbo
中，EchoService 相当于是一个健康检查（Health Check）机制，可以在 Java
程序中进行调用以测试目标服务是否运行正常。通过代理机制，无论是否定义，每个服务都会自动增加这个接口，保证每个代理都可以使用回声服务。AbstractProxyFactory
的 getProxy 方法如下所示：

    
    
    public abstract class AbstractProxyFactory implements ProxyFactory {
    
        public <T> T getProxy(Invoker<T> invoker) throws RpcException {
            Class<?>[] interfaces = null;
            //从 URL 中获取包含接口列表的字符串
            String config = invoker.getUrl().getParameter("interfaces");
            if (config != null && config.length() > 0) {
                //切分接口列表字符串
                String[] types = Constants.COMMA_SPLIT_PATTERN.split(config);
                if (types != null && types.length > 0) {
                    interfaces = new Class<?>[types.length + 2];
    
                    // 设置服务接口类和 EchoService.class 到 interfaces 中
                    interfaces[0] = invoker.getInterface();
                    interfaces[1] = EchoService.class;
                    // 加载接口类
                    for (int i = 0; i < types.length; i++) {
                        interfaces[i + 1] = ReflectUtils.forName(types[i]);
                    }
                }
            }
    
            //如果接口类为空，则新建一个并添加 EchoService.class
            if (interfaces == null) {
                interfaces = new Class<?>[]{invoker.getInterface(), EchoService.class};
            }
            return getProxy(invoker, interfaces);
        }
    
        public abstract <T> T getProxy(Invoker<T> invoker, Class<?>[] types);
    }
    

可以看到，上述方法中为每个服务自动添加 EchoService 的做法是操作 interfaces。我们从 URL 中获取传入的
`"interfaces"` 参数，然后构建一个 interfaces 数组添加这个 EchoService.class，最后整个 interfaces
数组传递给模块方法供 AbstractProxyFactory 的子类进行使用。通过 interfaces
添加所需的服务，这也是代理机制一种非常巧妙的使用方法，可以在日常开发过程中进行参考。

在 AbstractProxyFactory 预留了一个 getProxy 抽象方法供子类进行实现。在 Dubbo 中存在两种 ProxyFactory
的实现类，即 JavassistProxyFactory 和 JdkProxyFactory。而基于《架构模式之微内核模式：微内核模式在 Dubbo
中的应用》中的介绍，我们知道 ProxyFactory 接口上的 `@SPI("javassist")` 注解的含义在于默认使用的是基于 Javassist
的代理机制。在 Dubbo 中，JdkProxyFactory 的实现比较典型，而 JavassistProxyFactory 的实现非常复杂，我们先从
JdkProxyFactory 开始展开。

### JdkProxyFactory

JdkProxyFactory 的 getProxy 方法如下所示：

    
    
    public class JdkProxyFactory extends AbstractProxyFactory {
    
        @SuppressWarnings("unchecked")
        public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
            return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
        }   
        ...
    }
    

这里看到了 Proxy.newProxyInstance 方法，这是典型的 JDK
动态代理的用法。根据传入的接口获得动态代理类，当调用这些接口的方法时都会转而调用
InvokerInvocationHandler(invoker)，我们来看一下 InvokerInvocationHandler 类。基于 JDK
动态代理的实现机制，可以想象 InvokerInvocationHandler 类必定实现了 InvocationHandler 接口。

    
    
    public class InvokerInvocationHandler implements InvocationHandler {
    
        private final Invoker<?> invoker;
    
        public InvokerInvocationHandler(Invoker<?> handler) {
            this.invoker = handler;
        }
    
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            String methodName = method.getName();
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (method.getDeclaringClass() == Object.class) {
                return method.invoke(invoker, args);
            }
            if ("toString".equals(methodName) && parameterTypes.length == 0) {
                return invoker.toString();
            }
            if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
                return invoker.hashCode();
            }
            if ("equals".equals(methodName) && parameterTypes.length == 1) {
                return invoker.equals(args[0]);
            }
            return invoker.invoke(new RpcInvocation(method, args)).recreate();
        }
    }
    

可以看到，如果传入的方法继承自 Object，则直接通过反射进行调用。而针对不是来自 Object 的 `"toString"`、`"hashCode"`
和 `"equals"` 方法，则调用 invoker 中的对应方法。最后，这里通过传入方法名与参数构建了一个 RpcInvocation 对象，并传递到
Invoker 的 invoke 函数中去。关于 invoker 的介绍不是本篇内容的重点，我们在后续讲到 Dubbo
的服务发布和引用时会重点对其进行展开。

### JavassistProxyFactory

和 JdkProxyFactory 一样，我们直接来看 JavassistProxyFactory 的 getProxy 方法，如下所示：

    
    
    public class JavassistProxyFactory extends AbstractProxyFactory {
    
        public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
            return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
        }
        ...
    }
    

可以看到这里的 getProxy 方法与 JdkProxyFactory 中的同名方法在使用上非常相似，但请注意这里使用的 Proxy 类并不是 JDK
自带的，而是 Dubbo 自己实现的一个代理类，位于 dubbo-common 工程的 com.alibaba.dubbo.common.bytecode
包中。同时，Proxy 是一个抽象类，定义了如下所示的 newInstance 抽象方法：

    
    
    abstract public Object newInstance(InvocationHandler handler);
    

然后，我们发现 Dubbo 并没有显式的提供这个方法的实现，也就是说作为抽象类的 Proxy 并没有提供具体的实现类。但是在 Proxy
中提供了如下所示的静态方法完成具体实现类的构建，这个方法也就是暴露给 JavassistProxyFactory 的入口方法：

    
    
    public static Proxy getProxy(Class<?>... ics) {
        return getProxy(ClassHelper.getClassLoader(Proxy.class), ics);
    }
    

我们看到，这里进一步调用了一个静态的 getProxy 方法。总体而言，这个 getProxy 方法的实现比较复杂，代码也非常冗长，而且代码风格上也与
Dubbo
中的其他代码相差较大，各种变量的命名普通采用英文单词的缩写，让人觉得比较费解。因此，我们将采用代码框架结构的方式来展示这个方法的实现过程，效果如下所示：

    
    
    public static Proxy getProxy(ClassLoader cl, Class<?>... ics) {
    
        //第一部分：处理缓存，以提高代理类的获取效率
        //基于输入的 ClassLoader 构建缓存    
        //从缓存中获取目前 Proxy 对象，并进行多线程控制，保证只有一个线程可以进行后续操作
    
        //第二部分：构建服务接口的代理类
        //构建用于生成服务接口代理类的 ClassGenerator 工具类
        for (int i = 0; i < ics.length; i++) { 
            //添加接口和方法定义到 ClassGenerator 中
        }    
        //构建接口代理类名称：pkg + ".proxy" + id，比如 com.tianyalan.proxy0
        //生成服务接口的代理类
    
        //第三部分：基于服务接口的代理类实现来创建 Proxy 的实现类
          //构建用于 Proxy 抽象类所对应的子类的 ClassGenerator 工具类
        //构建 Proxy 子类名称，比如 Proxy1，Proxy2 等
        //为 Proxy 的抽象方法 newInstance 生成实现代码，这里会用到前面创建的服务接口代理类
        //通过反射创建 Proxy 实例并返回    
    }
    

根据上述注释，我们发现整个 getProxy 方法可以分成三大部分，即缓存处理、服务接口代理类的构建以及 Proxy
具体实现类的构建。其中缓存处理上因为考虑到对象构建过程的时间消耗成本以及并发环境下的访问机制，用到了多线程技术进行控制，这部分不是重点不做展开。

而对于服务接口代理类的构建，我们需要先明确代理类的具体表现形式。以 org.apache.dubbo.demo.DemoService
这个服务接口为例，它的代理类的表现形式一般如下所示：

    
    
    public class proxy0 implements org.apache.dubbo.demo.DemoService {
     public static java.lang.reflect.Method[] methods;
     private java.lang.reflect.InvocationHandler handler;
     public proxy0() {
     }
     public proxy0(java.lang.reflect.InvocationHandler arg0) {
     handler = $1;
     }
     public java.lang.String sayHello(java.lang.String arg0) {
     Object[] args = new Object[1];
     args[0] = ($w) $1;
     Object ret = handler.invoke(this, methods[0], args);
     return (java.lang.String) ret;
     }
    }
    

getProxy 方法通过构建一个 StringBuilder 对象完全以字符串的方式把这个类的代码拼接出来，并通过 ClassGenerator
工具类完成该类的构建。

有了类似上文中的服务接口代理类 proxy0，我们就可以使用它来实现 Proxy 抽象类的子类，并实现它的 newInstance 方法，如下所示：

    
    
    public Object newInstance(java.lang.reflect.InvocationHandler h) { 
      return new com.tianxiaobo.proxy0($1);
    }
    

这里，同样基于 StringBuilder 对象来拼接 newInstance 方法的代码，并通过 ClassGenerator 工具类生成 Proxy
的实现类。

从前面的描述中，实际上，我们并没有看到关于 Javassist 的相关内容，这部分内容实际上都被封装在 ClassGenerator
工具类中。ClassGenerator 是 Dubbo 自己封装的，该类的核心是 toClass()，该方法通过 Javassist 构建
Class。toClass() 方法代码也比较长，我们同样同学框架结构的方式对这些代码进行分析，如下所示：

    
    
     public Class<?> toClass(ClassLoader loader, ProtectionDomain pd) {
         //从 ClassPool 中获取 SuperClass
        //基于 SuperClass 从 ClassPool 中创建目标类
        //为目标类添加接口、方法、字构造函数
        //创建目标类
     }
    

Javassist 中最核心的就是 CtClass 类，而 CtClass 类的获取依赖于 ClassPool。上述代码框架中都是基于 Javassist
的 ClassPool 获取 SuperClass 和目标类，最终目标类的构建也是依赖于 CtClass 类的 toClass 方法。

### 面试题分析

#### **简要描述 Dubbo 中使用 JDK 动态代理机制实现远程访问的过程？**

**考点分析：**

这是一道好题目，笔者也经常拿它来面试不同层级的候选人。Dubbo 中存在两个中动态代理的实现机制，即 JDK 和
Javassist。尽管默认采用的是后者，但由于 Javassist 的实现过程过于复杂，一般不大适合作为一个面试题进行考查，所以更多的时候我们还是会拿
JDK 中的动态代理机制来进行讨论。

**解题思路：**

在回答这种问题上，第一步肯定是回答关于动态代理实现机制本身的内容，例如这里我们就需要对 JDK 中的 InvocationHandler 接口和 Proxy
静态类做详细的介绍。然后，针对 Dubbo 框架，我们也要明确所有的远程调用过程都是封装在一个个的 Invoker
对象中，所以动态代理的目的实际上就是把本地代码的调用过程转移到对 Invoker 对象的使用上，这是我们在回答类似问题上的基本思路。

**本文内容与建议回答：**

上一篇中，我们介绍了 JDK 中动态代理机制的实现过程，包括 InvocationHandler 接口和 Proxy
静态类。而本文中，则基于远程调用这一具体应用场景给出了 Dubbo 中使用 JDK 动态代理机制的具体细节。Dubbo
中通过工厂方法对代理机制做了一层抽象，也提供了 InvocationHandler 接口的实现类 InvokerInvocationHandler 来完成对
Invoker 对象的使用过程。整个流程还是比较清晰和明确的，在理解上建议关注于各个接口方法的输入输出参数，尤其是 Invoker 对象的使用方式。

### 小结与预告

本文介绍了 Dubbo 中基于动态代理机制实现远程调用的实现过程，分别给出了 JDK 和 Javassist
这两个具体的技术体系应用方式。在下一篇中，我们将继续讨论动态代理的另一个典型应用场景，即数据访问。

