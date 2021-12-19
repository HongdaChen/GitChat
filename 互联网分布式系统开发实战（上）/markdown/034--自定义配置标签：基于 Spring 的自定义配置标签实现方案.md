在《系统启动和初始化方法：Dubbo 启动过程解析》中介绍 Dubbo 的启动方法时，我们已经看到针对配置文件中的
application、protocol、registry、provider、service 等服务器端和客户端的配置项，Dubbo
提供了一套解析方法和过程来完成配置项到具体实现类的转变过程。这个过程值得我们作为专门的一个主题进行讲解，这个主题就是今天的“”自定义配置标签"。

从扩展性的角度讲，基于 XML schema 的扩展也是非常常见和实用的扩展方法。在 Spring 中，同样允许我们自己定义 XML
的的结构并且可以用自己的 Bean 解析器进行解析。本文将对基于 Spring 的自定义配置标签实现方案展开讨论，并给出简单的代码示例。这些是介绍基于
Dubbo 的自定义配置标签体系的基础。

### Spring Schema 的扩展时机

在介绍 Spring 的可扩展 Schema 机制之前，我们先来回顾一下《系统启动和初始化方法：Spring 中的启动扩展点（上）》中提到的
AbstractApplicationContext 中的 refresh 方法，如下所示：

    
    
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
    

其中这里的

    
    
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory()
    

语句会刷新 BeanFactory，加载 XML 配置文件从而生成 Map<String,BeanDefinition> 的一个映射。这个加载 Bean
的场景是一个比较常见的扩展点时机。

我们知道在 Spring 的 XML 中配置如下所示的 bean 的定义，Spring 就会在容器启动时进行解析然后转换成特定的
BeanDefinition。

    
    
    <bean id ="demoClass" class="com.demo.DemoClass"/>
    

显然，对于扩展性而言，我们完全可以在这种 bean 定义中添加自己所需的任何一些标签，从而实现定制化控制功能。自定义标签也算一种扩展。Spring
允许你自己定义 XML 的结构并且可以用自己的 bean 解析器进行解析。而要做到这一点，Spring 也提供了一套固定的开发流程。

### Spring Schema 扩展的开发流程

Spring Schema 扩展的开发流程包括编写 XSD 文件、编写 NamespaceHandler 实现类、编写
BeanDefinitionParser 实现类以及编写 spring.handlers 和 spring.schemas 这四个主要步骤。

#### **编写 XSD 文件**

我们可以根据需要设计一个业务对象，例如如下所示的 Account 对象：

    
    
    public class People {
        private String id;
        private String name;
        private Integer age;
    }
    

如果我们希望该对象中的所有字段都能够进行配置，那么就可以针对这些配置项编写 XSD（XML Schema Definition）文件，XSD 是
schema 的定义文件，本例中的 XSD 文件如下所示：

    
    
    <?xml version="1.0" encoding="UTF-8"?>
    <xsd:schema 
        xmlns="http://blog.csdn.net/cutesource/schema/people"
        xmlns:xsd="http://www.w3.org/2001/XMLSchema" 
        xmlns:beans="http://www.springframework.org/schema/beans"
        targetNamespace="http://blog.csdn.net/cutesource/schema/people"
        elementFormDefault="qualified" 
        attributeFormDefault="unqualified">
        <xsd:import namespace="http://www.springframework.org/schema/beans" />
        <xsd:element name="people">
            <xsd:complexType>
                <xsd:complexContent>
                    <xsd:extension base="beans:identifiedType">
                        <xsd:attribute name="name" type="xsd:string" />
                        <xsd:attribute name="age" type="xsd:int" />
                    </xsd:extension>
                </xsd:complexContent>
            </xsd:complexType>
        </xsd:element>
    </xsd:schema>
    

上述定义中，`<xsd:element name="people">` 对应着配置项节点的名称，因此在应用中会用 people 作为节点名来引用这个配置。而
`<xsd:attribute name="name" type="xsd:string" />` 和 `<xsd:attribute name="age"
type="xsd:int" />` 对应着配置项 people 的两个属性名，因此在应用中可以配置 name 和 age 两个属性，分别是 string
和 int 类型。

XSD 文件完成后需要存放在 classpath 下，一般都放在 META-INF 目录下。

#### **编写 NamespaceHandler 实现类**

需要完成对上述配置项的解析工作，需要用到 NamespaceHandler 和 BeanDefinitionParser 这两个概念。其中
NamespaceHandler 会根据 schema 和节点名找到某个 BeanDefinitionParser，然后由
BeanDefinitionParser 完成具体的解析工作。NamespaceHandler 接口的定义如下所示：

    
    
    public interface NamespaceHandler {
    
        void init();
    
        BeanDefinition parse(Element element, ParserContext parserContext);
    
        BeanDefinitionHolder decorate(Node source, BeanDefinitionHolder definition, ParserContext parserContext);
    }
    

想要从零开始实现这个接口还是有一定复杂度的，幸好 Spring 已经为我们提供了默认的 NamespaceHandlerSupport
实现类。实际上，Spring 内部很多特定组件的配置项解析也依赖于在这个 NamespaceHandlerSupport
类的基础之上演变出各种子类。我们从它的 parse 方法中不能看出，NamespaceHandlerSupport 首先寻找合适的
BeanDefinitionParser，然后把解析工作委托给了这个找到的 BeanDefinitionParser，如下所示：

    
    
    public BeanDefinition parse(Element element, ParserContext parserContext) {
            return findParserForElement(element, parserContext).parse(element, parserContext);
        }
    
        private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
            String localName = parserContext.getDelegate().getLocalName(element);
            BeanDefinitionParser parser = this.parsers.get(localName);
            if (parser == null) {
                parserContext.getReaderContext().fatal(
                        "Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
            }
            return parser;
        }
    

而这里的 parsers 对象中的具体 BeanDefinitionParser 是通过 registerBeanDefinitionParser
进行注册的，如下所示：

    
    
    protected final void registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser) {
        this.parsers.put(elementName, parser);
    }
    

所以，如果我们想要实现一个自定的 NamespaceHandler，最简单的方法就是继承 NamespaceHandlerSupport 然后在它的
init 方法中执行 registerBeanDefinitionParser，如下所示：

    
    
    public class MyNamespaceHandler extends NamespaceHandlerSupport {
        public void init() {
            registerBeanDefinitionParser("people", new PeopleBeanDefinitionParser());
        }
    }
    

通过以上方法，当我们在 Spring 的在配置中引用 `<people>` 配置项时，就会用 PeopleBeanDefinitionParser
来解析配置。

#### **编写 BeanDefinitionParser 实现类**

上述 PeopleBeanDefinitionParser 就是一个 BeanDefinitionParser
接口的实现类，该接口只有一个方法定义，如下所示：

    
    
    public interface BeanDefinitionParser {
        BeanDefinition parse(Element element, ParserContext parserContext); 
    }
    

针对这一接口，Spring 同样为我们提供了它的一个抽象实现类
AbstractBeanDefinitionParser。请注意，AbstractBeanDefinitionParser 类是一个典型的模板类，这点从它的
parse 方法就可以看出（为了简单起见对代码做了裁剪只保留主流程语句）：

    
    
    public final BeanDefinition parse(Element element, ParserContext parserContext) {
            AbstractBeanDefinition definition = parseInternal(element, parserContext);
            if (definition != null && !parserContext.isNested()) {
                try {
                    String id = resolveId(element, definition, parserContext);              
                    String[] aliases = new String[0];
                    String name = element.getAttribute(NAME_ATTRIBUTE);
    
                    BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
                    registerBeanDefinition(holder, parserContext.getRegistry());
                }
            }
            return definition;
        }
    

这段代码完成了 BeanDefinition 从解析到注册的主流程。这里第一句 parseInternal 是
AbstractBeanDefinitionParser 类提供的抽象方法，需要子类进行实现。一方面，我们可以直接继承
AbstractBeanDefinitionParser 以实现这个子类（例如 Dubbo 就是采用这种实现方法），而在 Spring 中也抽象了几个
AbstractBeanDefinitionParser 的实现类以降低二次扩展的难度，其中最典型的就是
AbstractSingleBeanDefinitionParser 类。我们找到这个类发现它同样也是一个模板类。在
AbstractSingleBeanDefinitionParser 的 parseInternal 方法的代码结构如下所示：

    
    
    protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
            BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
            builder.getRawBeanDefinition().setParentName(parentName);
    
            Class<?> beanClass = getBeanClass(element);
            if (beanClass != null) {
                builder.getRawBeanDefinition().setBeanClass(beanClass);
            }
            else {              builder.getRawBeanDefinition().setBeanClassName(beanClassName)
            }
            builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
                builder.setScope(parserContext.getContainingBeanDefinition().getScope());
            builder.setLazyInit(true);
    
            doParse(element, parserContext, builder);
    
            return builder.getBeanDefinition();
        }
    

显然，这里再次调用了 doParse 这个抽象方法。请注意这里的 BeanDefinitionBuilder，作为 BeanDefinition
的构造器，它同样传递给了 doParse 方法。

按照 Spring 中一般的方法命名风格，doParse 应该是整个方法调用链的末端方法，也是我们自定义 BeanDefinitionParser
所需要实现的方法。因此，针对上述示例，我们可以实现如下所示的继承了 AbstractSingleBeanDefinitionParser 的
PeopleBeanDefinitionParser 类：

    
    
    public class PeopleBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {
        protected Class getBeanClass(Element element) {
            return People.class;
        }
        protected void doParse(Element element, BeanDefinitionBuilder bean) {
            String name = element.getAttribute("name");
            String age = element.getAttribute("age");
            String id = element.getAttribute("id");
            if (StringUtils.hasText(id)) {
                bean.addPropertyValue("id", id);
            }
            if (StringUtils.hasText(name)) {
                bean.addPropertyValue("name", name);
            }
            if (StringUtils.hasText(age)) {
                bean.addPropertyValue("age", Integer.valueOf(age));
            }
        }
    }
    

这里的核心还是使用 BeanDefinitionBuilder 添加了各种自定义的配置项。其中 element.getAttribute
就是从配置项中取得属性值，bean.addPropertyValue 就是把属性值放到 bean 中。

#### **编写 spring.handlers 和 spring.schemas**

实现可扩展 Schema 的最后一步是编写 spring.handlers 和 spring.schemas
这两个配置文件，这两个文件需要我们自己编写并放入 META-INF 文件夹中。回想一下前面的课程，在前面介绍 SPI
时，我们已经看到过类似的操作手法。而在今天的场景中，这两个文件的地址必须是 META-INF/spring.handlers 和 META-
INF/spring.schemas。spring 会默认去载入它们，本例中 spring.handlers 如下所示：

    
    
    http\://demo.com/schema/people=com.demo.MyNamespaceHandler
    

以上表示当使用到名为“http://demo.com/schema/people”的 schema 引用时，会通过
com.demo.MyNamespaceHandler 类来完成解析。

同样，spring.schemas 的内容如下所示，用于指定 XSD 文件的路径：

    
    
    http\://demo.com/schema/people.xsd=META-INF/people.xsd
    

至此，整个基于 Schema 扩展的开发流程介绍完毕。接下来，就让我们来看一下 Dubbo 和 MyBatis-Spring 等框架中如何基于 Schema
扩展实现自定义的配置标签体系。

### 面试题分析

#### **简要阐述 Spring 中实现自定义标签的开发过程？**

**考点分析：**

这道题的考点是非常明确的，Spring 中实现自定义标签的过程也并不复杂，关键看面试者有没有类似的开发经历。

如果我们不需要和 Spring 框架进行集成，一般我们也就不需要实现自定的标签。所以，对于大多数面试者而言，这道题考查的还是知识体系的完整性。

**解题思路：**

在 Spring 中，实现自定义标签的过程是非常固化的，本文中把这一过程总结成编写 XSD 文件、编写 NamespaceHandler 实现类、编写
BeanDefinitionParser 实现类、编写 spring.handlers 和 spring.schemas
这四个步骤。我们在回答时，可以按照这四大步骤进行讲解。当然，大家也可以按照自己的思路把这些步骤进行重新梳理和封装。

**本文内容与建议回答：**

本文对上述实现自定义标签的四个步骤都进行了详细介绍，并配有对应的代码示例。另一方面，如果希望在回答该问题的过程中给面试官留下更好的印象，我们也提供了涉及到
NamespaceHandler 和 BeanDefinitionParser 部分的源码分析，建议在面试过程中也能适当的点到这些实现原理。

### 日常开发技巧

如果想要基于 Spring 框架来实现一个自定义标签，从而为系统提供更多的可扩展性，那么本文内容可以帮助你完整这一目标。事实上，如果我们的目标是开发一个与
Spring 进行有效集成的自定义框架（例如 Dubbo 和 MyBatis），那么自定义标签可以说是必不可少的一个环节，因为在 Spring
框架中，我们一般都需要依赖于它的 Namespace 体系完成一些配置项的设置和装配工作。当然，对于 Spring Boot
而言，因为其本身就提供了自动配置功能，所以实现方式上就不是通过配置项来达到这一目标。这点是前面的课程中已经得到明确。

### 小结与预告

本文从原理和实践两方面给出了如何基于 Spring 框架实现自定义标签体系的系统方法。有了这些认知之后，我们就可以进一步来理解 Dubbo 和
MyBatis 等框架中如何利用这些系统方法来完成与 Spring 框架的整合过程，这就是下一篇要介绍的内容。

