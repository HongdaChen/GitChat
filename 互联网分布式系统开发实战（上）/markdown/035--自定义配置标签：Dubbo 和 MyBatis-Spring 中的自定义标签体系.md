上文中我们介绍了 Spring
中自定义配置标签体系的实现方法，这是一种常见的扩展性实现方案。今天我们会结合目前主流的一些开源框架来介绍这种扩展性实现方法在这些框架中的不同应用方式。

### 解析 Dubbo 中的自定义配置标签体系

让我们回顾《系统启动和初始化方法：Dubbo 启动过程解析》中介绍的 Dubbo 启动方法。我们知道针对 Dubbo 配置项中提供的
application、protocol、registry、provider、service 等服务器端和客户端的配置项，Dubbo
提供了一套解析方法和过程来完成配置项到具体实现类的转变过程。

我们也提到了 Dubbo 提供了专门的 DubboNamespaceHandler 来完成各个配置项的解析，代码如下所示：

    
    
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
    

可以看到这个 DubboNamespaceHandler 就直接继承了 Spring 提供的 NamespaceHandlerSupport，然后通过
registerBeanDefinitionParser 方法针对各个配置项注册了各自对应的 BeanDefinitionParser，Dubbo 中这个
BeanDefinitionParser 就是 DubboBeanDefinitionParser。DubboBeanDefinitionParser
的定义如下所示：

    
    
    public class DubboBeanDefinitionParser implements BeanDefinitionParser {
        private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
            ...
        }
    }
    

可以看到与上一篇中介绍的实现方法不同，DubboBeanDefinitionParser 并不是继承自
AbstractSingleBeanDefinitionParser 抽象类，而是直接实现了 BeanDefinitionParser
接口。DubboBeanDefinitionParser 中 parse 方法的代码结构，这个方法比较长，我们采用分段挑重点语句的方法进行讲解：

    
    
    RootBeanDefinition beanDefinition = new RootBeanDefinition();
        beanDefinition.setBeanClass(beanClass);
        beanDefinition.setLazyInit(false);
    

首先初始化一个 RootBeanDefinition，它是 AbstractBeanDefinition 的一个实现类。然后设置这个
BeanDefinition 的 BeanClass 和指定懒加载的 LazyInit 标志位。设置懒加载为 false 表示 Spring
启动时会立即对该类进行实例化。Spring 中只有单例模式下的 Bean 才能使用懒加载设置，也就意味着这些 BeanDefinition 都是单例的。

然后，parse 方法通过很多准备和校验工作获取 bean 的 id 属性，我们需要考虑 id 不存在、id 重复以及 Dubbo 中存在的一些固定 id
等场景，并最终通过以下方法设置这一属性：

    
    
    parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
        beanDefinition.getPropertyValues().addPropertyValue("id", id);
    

接下来我们注意到 parse 方法分别对一些特定的 Bean 定义做了特殊处理，包括
ProtocolConfig、ServiceBean、ProviderConfig、ConsumerConfig 等，考虑这些 Bean
中存在的子节点、ref 引用等配置项。然后，我们也看到了大量对 Dubbo 中配置属性的解析过程。这些解析过程最终都是使用 Spring 的
Bea`nDefinition.getPropertyValues().addPropertyValue(property, value)`
这样的语句完成了 beanDefinition 的填充。例如，设置 `"parameters"` 的语句如下所示：

    
    
    if (parameters != null) {
         beanDefinition.getPropertyValues().addPropertyValue("parameters", parameters);
    }
    

我们结合《系统启动和初始化方法：Dubbo 启动过程解析》中展示的 Dubbo 的配置项示例不难理解这些代码的处理逻辑。

总体而言，DubboBeanDefinitionParser 的实现复杂度来自于 Dubbo 中配置项的复杂度。对于我们日常开发中所需要实现的
BeanDefinitionParse 组件的开发模式而言，本质上没有什么特殊之处。

明确了 Dubbo 中 NamespaceHandler 和 BeanDefinitionParser 实现类之后，我们可以来找一下
spring.handlers。我们发现在 dubbo-config-spring 工程中，存在如下所示的 spring.handlers
文件，显然这里添加了对 DubboNamespaceHandler 类的引用：

    
    
    http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
    

同样，我们在同一个工程中也找到了 spring.schemas，内容如下所示：

    
    
    http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
    

可以看到 Dubbo 中的 XSD 文件 dubbo.xsd 也同样位于 META-INF 文件夹下。dubbo.xsd 包含了所有配置项的定义，对应于
Dubbo 中所有配置类。这里以 ApplicationConfig 为例，类的部分定义如下所示（作为演示，只挑选了一部分内容）：

    
    
    public class ApplicationConfig extends AbstractConfig {
        // application name
        private String name;
    
        // module version
        private String version;
    
        // 省略其他定义
    }
    

而 ApplicationConfig 的各个属性也都在 dubbo.xsd 中找到定义，如下所示：

    
    
    <xsd:complexType name="applicationType">
            <xsd:sequence minOccurs="0" maxOccurs="unbounded">
                <xsd:element ref="parameter" minOccurs="0" maxOccurs="unbounded"/>
            </xsd:sequence>
            <xsd:attribute name="id" type="xsd:ID">
                <xsd:annotation>
                    <xsd:documentation><![CDATA[ The unique identifier for a bean. ]]></xsd:documentation>
                </xsd:annotation>
            </xsd:attribute>
            <xsd:attribute name="name" type="xsd:string" use="required">
                <xsd:annotation>
                    <xsd:documentation><![CDATA[ The application name. ]]></xsd:documentation>
                </xsd:annotation>
            </xsd:attribute>
            <xsd:attribute name="version" type="xsd:string">
                <xsd:annotation>
                    <xsd:documentation><![CDATA[ The application version. ]]></xsd:documentation>
                </xsd:annotation>
            </xsd:attribute>
        …
    </xsd:complexType>
    

Dubbo 中的整个自定义配置标签体系完全采用了 Spring 中基于 Scheme 进行扩展的标准实现机制。通过这样一个过程，就实现了将 XML
自定义的标签加载到 Spring 容器中，而不需要使用 Spring 自己的 Bean 去定义。这也是日常开发过程中的一种简单而实用的技巧。

### 解析 MyBatis-Spring 中的自定义配置标签体系

接下来，我们同样对 MyBatis-Spring 中的自定义配置标签体系进行讲解。在 MyBatis 中，只是用了一个自定义配置标签，即如下所示的
`<scan>`：

    
    
    <mybatis:scan base-package="com.tianyalan.mappers" />
    

因此，它的 NamespaceHandler 比较简单，如下所示：

    
    
    public class NamespaceHandler extends NamespaceHandlerSupport {
    
      @Override
      public void init() {
        registerBeanDefinitionParser("scan", new MapperScannerBeanDefinitionParser());
      }
    }
    

在介绍这里的 MapperScannerBeanDefinitionParser 之前，我们首先需要对 Spring 中的 Scanner
机制进行一定的解释，这也是很多开源框架中所集成的一项基础功能。

如果在 Spring 项目中添加 `<context:component-scan base-package="com.example.demo" />`
类似的配置项 ，那么在容器初始化阶段（即调用 beanFactoryPostProcessor 阶段），就会采用
ClassPathBeanDefinitionScanner 进行扫描包下所有类，并将符合过滤条件的类注册到容器内。

我们直接来到 ClassPathBeanDefinitionScanner 类，找到它的 scan
方法，我们看到这个方法并不真正执行扫描操作，而是把这部分职责转移到了 doScan 方法中，该方法返回了一个
`Set<BeanDefinitionHolder>`。而在 doScan 方法中，我们同样发现 findCandidateComponents
才是实际执行包扫描的方法入口。该方法定义在 ClassPathBeanDefinitionScanner 的父类
ClassPathScanningCandidateComponentProvider 中，ClassPathBeanDefinitionScanner
的主要功能实现都在这个函数中。

findCandidateComponents 方法返回一个
`Set<BeanDefinition>`，核心逻辑如下所示（为了简洁，对代码做了裁剪，去掉了很多 log 处理，只保留主体流程）：

    
    
        public Set<BeanDefinition> findCandidateComponents(String basePackage) {
            Set<BeanDefinition> candidates = new LinkedHashSet<BeanDefinition>();
            try {
                //根据指定包名 生成包搜索路径
                String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
                        resolveBasePackage(basePackage) + '/' + this.resourcePattern;
                //资源加载器加载搜索路径下的所有 class 转换为 Resource[]
                Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);
    
                //循环处理每一个 Resource 
                for (Resource resource : resources) {               
                    if (resource.isReadable()) {
                        try {
                            //读取类的注解信息和类信息，信息储存到 MetadataReader
                            MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);
                            // 执行判断是否符合 过滤器规则，函数内部用过滤器 对 metadataReader 过滤  
                            if (isCandidateComponent(metadataReader)) {
                                //把符合条件的 类转换成 BeanDefinition
                                ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                                sbd.setResource(resource);
                                sbd.setSource(resource);
                                if (isCandidateComponent(sbd)) {
                                    candidates.add(sbd);
                                }
                            }
                        }
                    }
                }
            }
    
            return candidates;
        }
    

上述描述的整个执行流程如下所示：

![FLph7H](https://images.gitbook.cn/2020-05-25-052610.png)

MyBatis-Spring 中的 ClassPathMapperScanner 继承自 ClassPathBeanDefinitionScanner，对于
Spring 而言，相当于一个自定义 Scanner。ClassPathMapperScanner 提供了如下所示的类定义以及重新父类的 doScan
方法：

    
    
    public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {
      ...
    
      @Override
      public Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);
    
        if (beanDefinitions.isEmpty()) {
          logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
        } else {
          processBeanDefinitions(beanDefinitions);
        }
    
        return beanDefinitions;
      }
    
      ...
    }
    

可以看到，这里使用父类 ClassPathBeanDefinitionScanner 的 doScan 方法完成了 Bean 的扫描。

最后，我们回到解析 `"scan"` 自定义标签的 MapperScannerBeanDefinitionParser 类，它的 parse
如下所示（代码做了裁剪）：

    
    
    public class MapperScannerBeanDefinitionParser implements BeanDefinitionParser {
    
          @Override
          public synchronized BeanDefinition parse(Element element, ParserContext parserContext) {
          ClassPathMapperScanner scanner = new ClassPathMapperScanner(parserContext.getRegistry());
        ClassLoader classLoader = scanner.getResourceLoader().getClassLoader();
        XmlReaderContext readerContext = parserContext.getReaderContext();
        scanner.setResourceLoader(readerContext.getResourceLoader());
        ...
        scanner.setAnnotationClass(markerInterface);
        ...
        scanner.setMarkerInterface(markerInterface);
        ...
        scanner.setBeanNameGenerator(nameGenerator);   
        ...
        scanner.setSqlSessionTemplateBeanName(sqlSessionTemplateBeanName);
        ...
        scanner.setSqlSessionFactoryBeanName(sqlSessionFactoryBeanName);
        //注册过滤器
        scanner.registerFilters();
    
        String basePackage = element.getAttribute(ATTRIBUTE_BASE_PACKAGE);
        //执行扫描
        scanner.scan(StringUtils.tokenizeToStringArray(basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
    
        return null;
      }
    

### 面试题分析

#### **如何理解 Spring 框架中的 Scanner 机制？**

**考点分析：**

这道题对于很多面试者而言算有点冷门，平时可能不大会关注这方面的内容。当然，从应用角度讲，Scanner 也算是一项基础功能，我们还是可以通过 MyBatis
等框架接触到一些关于包扫描的配置机制，但深入研究其原理需要做专门的准备。

**解题思路：**

回答该问题时，我们可以先列举一些关于 Scanner 机制的应用场景，可以结合 MyBatis 框架来展开讨论。而针对实现原理，我们要明确 Scanner
机制的根本目的是为了获取所需要的 BeanDefinition，所以在其实现过程中势必会涉及到 Bean 资源的查找、获取和加载，而这一过程又往往需要依赖于
ClassPath。这样通过这几个点，我们就能把整个流程串接起来，并给出面试官所期望听到的各个要点。

**本文内容与建议回答：**

本文对 Spring 中的 Scanner 机制做了简要的介绍，很多开源框架中也都集成了这项基础功能。我们通过翻阅源代码对 Scanner
机制中所涉及的一些类结构做了分析，并给出了一张时序图方便大家对这块内容进行记忆。

### 小结与预告

本文讨论了 Dubbo 和 MyBatis
框架中自定义标签体系的具体应用方式，可以看到这些应用方式与在上一篇讨论的开发步骤是高度一致的。从下一篇开始，我们将花几篇文章来讨论一个我们已经接触过很多次的话题，即代理机制。我们会发现，在开源框架中，代理机制的应用非常广泛，可以说处处是代理。

