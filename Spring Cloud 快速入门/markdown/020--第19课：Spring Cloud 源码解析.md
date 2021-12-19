Spring Cloud
集成了很多第三方框架，把它的全部源码拿出来解析几本书都讲不完，也不太现实，本文带领读者分析其中一小部分源码（其余源码读者有兴趣可以继续跟进），包括
Eureka-Server、Config、Zuul 的 starter 部分，分析其启动原理。

如果我们开发出一套框架，要和 Spring Boot 集成，就需要放到它的 starter 里。因此我们分析启动原理，直接从每个框架的 starter
开始分析即可。

### Eureka-Server 源码解析

我们知道，要实现注册与发现，需要在启动类加上 `@EnableEurekaServer` 注解，我们进入其源码：

    
    
    @EnableDiscoveryClient//表示eurekaserver也是一个客户端服务
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Import(EurekaServerMarkerConfiguration.class)
    public @interface EnableEurekaServer {
    
    }
    

注意看 `@Import` 注解，这个注解导入了 EurekaServerMarkerConfiguration 类，继续跟进这个类：

    
    
    /**
     * Responsible for adding in a marker bean to activate
     * {@link EurekaServerAutoConfiguration}
     *
     * @author Biju Kunjummen
     */
    @Configuration
    public class EurekaServerMarkerConfiguration {
    
        @Bean
        public Marker eurekaServerMarkerBean() {
            return new Marker();
        }
    
        class Marker {
        }
    }
    

通过上面的注释，我们继续查看 EurekaServerAutoConfiguration 类的源码：

    
    
    @Configuration
    @Import(EurekaServerInitializerConfiguration.class)
    @ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
    @EnableConfigurationProperties({ EurekaDashboardProperties.class,
            InstanceRegistryProperties.class })
    @PropertySource("classpath:/eureka/server.properties")
    public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter {
        /**
         * List of packages containing Jersey resources required by the Eureka server
         */
        private static String[] EUREKA_PACKAGES = new String[] { "com.netflix.discovery",
                "com.netflix.eureka" };
    
        @Autowired
        private ApplicationInfoManager applicationInfoManager;
    
        @Autowired
        private EurekaServerConfig eurekaServerConfig;
    
        @Autowired
        private EurekaClientConfig eurekaClientConfig;
    
        @Autowired
        private EurekaClient eurekaClient;
    
        @Autowired
        private InstanceRegistryProperties instanceRegistryProperties;
    
        public static final CloudJacksonJson JACKSON_JSON = new CloudJacksonJson();
    
        @Bean
        public HasFeatures eurekaServerFeature() {
            return HasFeatures.namedFeature("Eureka Server",
                    EurekaServerAutoConfiguration.class);
        }
    
        //如果eureka.client.registerWithEureka=true，则把自己注册进去
        @Configuration
        protected static class EurekaServerConfigBeanConfiguration {
            @Bean
            @ConditionalOnMissingBean
            public EurekaServerConfig eurekaServerConfig(EurekaClientConfig clientConfig) {
                EurekaServerConfigBean server = new EurekaServerConfigBean();
                if (clientConfig.shouldRegisterWithEureka()) {
                    // Set a sensible default if we are supposed to replicate
                    server.setRegistrySyncRetries(5);
                }
                return server;
            }
        }
    
        //实例化eureka-server的界面
        @Bean
        @ConditionalOnProperty(prefix = "eureka.dashboard", name = "enabled", matchIfMissing = true)
        public EurekaController eurekaController() {
            return new EurekaController(this.applicationInfoManager);
        }
    
        static {
            CodecWrappers.registerWrapper(JACKSON_JSON);
            EurekaJacksonCodec.setInstance(JACKSON_JSON.getCodec());
        }
    
        @Bean
        public ServerCodecs serverCodecs() {
            return new CloudServerCodecs(this.eurekaServerConfig);
        }
    
        private static CodecWrapper getFullJson(EurekaServerConfig serverConfig) {
            CodecWrapper codec = CodecWrappers.getCodec(serverConfig.getJsonCodecName());
            return codec == null ? CodecWrappers.getCodec(JACKSON_JSON.codecName()) : codec;
        }
    
        private static CodecWrapper getFullXml(EurekaServerConfig serverConfig) {
            CodecWrapper codec = CodecWrappers.getCodec(serverConfig.getXmlCodecName());
            return codec == null ? CodecWrappers.getCodec(CodecWrappers.XStreamXml.class)
                    : codec;
        }
    
        class CloudServerCodecs extends DefaultServerCodecs {
    
            public CloudServerCodecs(EurekaServerConfig serverConfig) {
                super(getFullJson(serverConfig),
                        CodecWrappers.getCodec(CodecWrappers.JacksonJsonMini.class),
                        getFullXml(serverConfig),
                        CodecWrappers.getCodec(CodecWrappers.JacksonXmlMini.class));
            }
        }
    
        @Bean
        public PeerAwareInstanceRegistry peerAwareInstanceRegistry(
                ServerCodecs serverCodecs) {
            this.eurekaClient.getApplications(); // force initialization
            return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig,
                    serverCodecs, this.eurekaClient,
                    this.instanceRegistryProperties.getExpectedNumberOfRenewsPerMin(),
                    this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
        }
    
        @Bean
        @ConditionalOnMissingBean
        public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry,
                ServerCodecs serverCodecs) {
            return new PeerEurekaNodes(registry, this.eurekaServerConfig,
                    this.eurekaClientConfig, serverCodecs, this.applicationInfoManager);
        }
    
        @Bean
        public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
                PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
            return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs,
                    registry, peerEurekaNodes, this.applicationInfoManager);
        }
    
        @Bean
        public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
                EurekaServerContext serverContext) {
            return new EurekaServerBootstrap(this.applicationInfoManager,
                    this.eurekaClientConfig, this.eurekaServerConfig, registry,
                    serverContext);
        }
    
        /**
         * Register the Jersey filter
         */
        @Bean
        public FilterRegistrationBean jerseyFilterRegistration(
                javax.ws.rs.core.Application eurekaJerseyApp) {
            FilterRegistrationBean bean = new FilterRegistrationBean();
            bean.setFilter(new ServletContainer(eurekaJerseyApp));
            bean.setOrder(Ordered.LOWEST_PRECEDENCE);
            bean.setUrlPatterns(
                    Collections.singletonList(EurekaConstants.DEFAULT_PREFIX + "/*"));
    
            return bean;
        }
    
        /**
         * Construct a Jersey {@link javax.ws.rs.core.Application} with all the resources
         * required by the Eureka server.
         */
        @Bean
        public javax.ws.rs.core.Application jerseyApplication(Environment environment,
                ResourceLoader resourceLoader) {
    
            ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(
                    false, environment);
    
            // Filter to include only classes that have a particular annotation.
            //
            provider.addIncludeFilter(new AnnotationTypeFilter(Path.class));
            provider.addIncludeFilter(new AnnotationTypeFilter(Provider.class));
    
            // Find classes in Eureka packages (or subpackages)
            //
            Set<Class<?>> classes = new HashSet<Class<?>>();
            for (String basePackage : EUREKA_PACKAGES) {
                Set<BeanDefinition> beans = provider.findCandidateComponents(basePackage);
                for (BeanDefinition bd : beans) {
                    Class<?> cls = ClassUtils.resolveClassName(bd.getBeanClassName(),
                            resourceLoader.getClassLoader());
                    classes.add(cls);
                }
            }
    
            // Construct the Jersey ResourceConfig
            //
            Map<String, Object> propsAndFeatures = new HashMap<String, Object>();
            propsAndFeatures.put(
                    // Skip static content used by the webapp
                    ServletContainer.PROPERTY_WEB_PAGE_CONTENT_REGEX,
                    EurekaConstants.DEFAULT_PREFIX + "/(fonts|images|css|js)/.*");
    
            DefaultResourceConfig rc = new DefaultResourceConfig(classes);
            rc.setPropertiesAndFeatures(propsAndFeatures);
    
            return rc;
        }
    
        @Bean
        public FilterRegistrationBean traceFilterRegistration(
                @Qualifier("webRequestLoggingFilter") Filter filter) {
            FilterRegistrationBean bean = new FilterRegistrationBean();
            bean.setFilter(filter);
            bean.setOrder(Ordered.LOWEST_PRECEDENCE - 10);
            return bean;
        }
    }
    

这个类上有一个注解：`@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)`，这后面指定的类就是刚才那个类，而
`@ConditionalOnBean` 这个注解的作用是：仅仅在当前上下文中存在某个对象时，才会实例化一个 Bean。

因此，启动时就会实例化 EurekaServerAutoConfiguration 这个类。

    
    
    @EnableConfigurationProperties({ EurekaDashboardProperties.class,
            InstanceRegistryProperties.class })
    

这个注解就是定义了一些 Eureka 的配置项。

### Config 源码解析

通过上面的方法，我们找到了 ConfigServerAutoConfiguration 类：

    
    
    @Configuration
    @ConditionalOnBean(ConfigServerConfiguration.Marker.class)
    @EnableConfigurationProperties(ConfigServerProperties.class)
    @Import({ EnvironmentRepositoryConfiguration.class, CompositeConfiguration.class, ResourceRepositoryConfiguration.class,
            ConfigServerEncryptionConfiguration.class, ConfigServerMvcConfiguration.class, TransportConfiguration.class })
    public class ConfigServerAutoConfiguration {
    
    }
    

可以发现这个类是空的，只是多了几个注解，
`@EnableConfigurationProperties(ConfigServerProperties.class)` 表示开启 Config
配置属性。

最核心的注解是：`@Import`，它将其他一些配置类导入这个类，其中， EnvironmentRepositoryConfiguration
为环境配置类，内置了以下几种环境配置。

**1\. Native**

    
    
    @Configuration
        @Profile("native")
        protected static class NativeRepositoryConfiguration {
    
            @Autowired
            private ConfigurableEnvironment environment;
    
            @Bean
            public NativeEnvironmentRepository nativeEnvironmentRepository() {
                return new NativeEnvironmentRepository(this.environment);
            }
        }
    

**2\. git**

    
    
    @Configuration
        @Profile("git")
        protected static class GitRepositoryConfiguration extends DefaultRepositoryConfiguration {}
    

**3\. subversion**

    
    
    @Configuration
        @Profile("subversion")
        protected static class SvnRepositoryConfiguration {
            @Autowired
            private ConfigurableEnvironment environment;
    
            @Autowired
            private ConfigServerProperties server;
    
            @Bean
            public SvnKitEnvironmentRepository svnKitEnvironmentRepository() {
                SvnKitEnvironmentRepository repository = new SvnKitEnvironmentRepository(this.environment);
                if (this.server.getDefaultLabel()!=null) {
                    repository.setDefaultLabel(this.server.getDefaultLabel());
                }
                return repository;
            }
        }
    

**4.vault**

    
    
    @Configuration
        @Profile("subversion")
        protected static class SvnRepositoryConfiguration {
            @Autowired
            private ConfigurableEnvironment environment;
    
            @Autowired
            private ConfigServerProperties server;
    
            @Bean
            public SvnKitEnvironmentRepository svnKitEnvironmentRepository() {
                SvnKitEnvironmentRepository repository = new SvnKitEnvironmentRepository(this.environment);
                if (this.server.getDefaultLabel()!=null) {
                    repository.setDefaultLabel(this.server.getDefaultLabel());
                }
                return repository;
        }
        }
    

从代码可以看到 Git 是配置中心默认环境。

    
    
    @Bean
            public MultipleJGitEnvironmentRepository defaultEnvironmentRepository() {
                MultipleJGitEnvironmentRepository repository = new MultipleJGitEnvironmentRepository(this.environment);
                repository.setTransportConfigCallback(this.transportConfigCallback);
                if (this.server.getDefaultLabel()!=null) {
                    repository.setDefaultLabel(this.server.getDefaultLabel());
                }
                return repository;
            }
    

我们进入 MultipleJGitEnvironmentRepository 类：

    
    
    @ConfigurationProperties("spring.cloud.config.server.git")
    public class MultipleJGitEnvironmentRepository extends JGitEnvironmentRepository {
    }
    

这个类表示可以支持配置多个 Git 仓库，它继承自 JGitEnvironmentRepository 类：

    
    
    public class JGitEnvironmentRepository extends AbstractScmEnvironmentRepository
            implements EnvironmentRepository, SearchPathLocator, InitializingBean {
    /**
         * Get the working directory ready.
         */
        public String refresh(String label) {
            Git git = null;
            try {
                git = createGitClient();
                if (shouldPull(git)) {
                    fetch(git, label);
                    // checkout after fetch so we can get any new branches, tags,
                    // ect.
                    checkout(git, label);
                    if (isBranch(git, label)) {
                        // merge results from fetch
                        merge(git, label);
                        if (!isClean(git)) {
                            logger.warn("The local repository is dirty. Resetting it to origin/" + label + ".");
                            resetHard(git, label, "refs/remotes/origin/" + label);
                        }
                    }
                } else {
                    // nothing to update so just checkout
                    checkout(git, label);
                }
                // always return what is currently HEAD as the version
                return git.getRepository().getRef("HEAD").getObjectId().getName();
            } catch (RefNotFoundException e) {
                throw new NoSuchLabelException("No such label: " + label, e);
            } catch (NoRemoteRepositoryException e) {
                throw new NoSuchRepositoryException("No such repository: " + getUri(), e);
            } catch (GitAPIException e) {
                throw new NoSuchRepositoryException("Cannot clone or checkout repository: " + getUri(), e);
            } catch (Exception e) {
                throw new IllegalStateException("Cannot load environment", e);
            } finally {
                try {
                    if (git != null) {
                        git.close();
                    }
                } catch (Exception e) {
                    this.logger.warn("Could not close git repository", e);
                }
            }
        }
    }
    

refresh 方法的作用就是 ConfigServer 会从我们配置的 Git 仓库拉取配置下来。

### Zuul 源码解析

同理，我们找到 Zuul 的配置类 ZuulProxyAutoConfiguration：

    
    
    @Configuration
    @Import({ RibbonCommandFactoryConfiguration.RestClientRibbonConfiguration.class,
            RibbonCommandFactoryConfiguration.OkHttpRibbonConfiguration.class,
            RibbonCommandFactoryConfiguration.HttpClientRibbonConfiguration.class })
    @ConditionalOnBean(ZuulProxyMarkerConfiguration.Marker.class)
    public class ZuulProxyAutoConfiguration extends ZuulServerAutoConfiguration {
    
        @SuppressWarnings("rawtypes")
        @Autowired(required = false)
        private List<RibbonRequestCustomizer> requestCustomizers = Collections.emptyList();
    
        @Autowired
        private DiscoveryClient discovery;
    
        @Autowired
        private ServiceRouteMapper serviceRouteMapper;
    
        @Override
        public HasFeatures zuulFeature() {
            return HasFeatures.namedFeature("Zuul (Discovery)", ZuulProxyAutoConfiguration.class);
        }
    
        @Bean
        @ConditionalOnMissingBean(DiscoveryClientRouteLocator.class)
        public DiscoveryClientRouteLocator discoveryRouteLocator() {
            return new DiscoveryClientRouteLocator(this.server.getServletPrefix(), this.discovery, this.zuulProperties,
                    this.serviceRouteMapper);
        }
        //以下是过滤器，也就是之前zuul提到的实现的ZuulFilter接口
        // pre filters
        //路由之前
        @Bean
        public PreDecorationFilter preDecorationFilter(RouteLocator routeLocator, ProxyRequestHelper proxyRequestHelper) {
            return new PreDecorationFilter(routeLocator, this.server.getServletPrefix(), this.zuulProperties,
                    proxyRequestHelper);
        }
    
        // route filters
        // 路由时
        @Bean
        public RibbonRoutingFilter ribbonRoutingFilter(ProxyRequestHelper helper,
                RibbonCommandFactory<?> ribbonCommandFactory) {
            RibbonRoutingFilter filter = new RibbonRoutingFilter(helper, ribbonCommandFactory, this.requestCustomizers);
            return filter;
        }
    
        @Bean
        @ConditionalOnMissingBean(SimpleHostRoutingFilter.class)
        public SimpleHostRoutingFilter simpleHostRoutingFilter(ProxyRequestHelper helper, ZuulProperties zuulProperties) {
            return new SimpleHostRoutingFilter(helper, zuulProperties);
        }
    
        @Bean
        public ApplicationListener<ApplicationEvent> zuulDiscoveryRefreshRoutesListener() {
            return new ZuulDiscoveryRefreshListener();
        }
    
        @Bean
        @ConditionalOnMissingBean(ServiceRouteMapper.class)
        public ServiceRouteMapper serviceRouteMapper() {
            return new SimpleServiceRouteMapper();
        }
    
        @Configuration
        @ConditionalOnMissingClass("org.springframework.boot.actuate.endpoint.Endpoint")
        protected static class NoActuatorConfiguration {
    
            @Bean
            public ProxyRequestHelper proxyRequestHelper(ZuulProperties zuulProperties) {
                ProxyRequestHelper helper = new ProxyRequestHelper();
                helper.setIgnoredHeaders(zuulProperties.getIgnoredHeaders());
                helper.setTraceRequestBody(zuulProperties.isTraceRequestBody());
                return helper;
            }
    
        }
    
        @Configuration
        @ConditionalOnClass(Endpoint.class)
        protected static class RoutesEndpointConfiguration {
    
            @Autowired(required = false)
            private TraceRepository traces;
    
            @Bean
            public RoutesEndpoint zuulEndpoint(RouteLocator routeLocator) {
                return new RoutesEndpoint(routeLocator);
            }
    
            @Bean
            public RoutesMvcEndpoint zuulMvcEndpoint(RouteLocator routeLocator, RoutesEndpoint endpoint) {
                return new RoutesMvcEndpoint(endpoint, routeLocator);
            }
    
            @Bean
            public ProxyRequestHelper proxyRequestHelper(ZuulProperties zuulProperties) {
                TraceProxyRequestHelper helper = new TraceProxyRequestHelper();
                if (this.traces != null) {
                    helper.setTraces(this.traces);
                }
                helper.setIgnoredHeaders(zuulProperties.getIgnoredHeaders());
                helper.setTraceRequestBody(zuulProperties.isTraceRequestBody());
                return helper;
            }
        }
    
        private static class ZuulDiscoveryRefreshListener implements ApplicationListener<ApplicationEvent> {
    
            private HeartbeatMonitor monitor = new HeartbeatMonitor();
    
            @Autowired
            private ZuulHandlerMapping zuulHandlerMapping;
    
            @Override
            public void onApplicationEvent(ApplicationEvent event) {
                if (event instanceof InstanceRegisteredEvent) {
                    reset();
                }
                else if (event instanceof ParentHeartbeatEvent) {
                    ParentHeartbeatEvent e = (ParentHeartbeatEvent) event;
                    resetIfNeeded(e.getValue());
                }
                else if (event instanceof HeartbeatEvent) {
                    HeartbeatEvent e = (HeartbeatEvent) event;
                    resetIfNeeded(e.getValue());
                }
    
            }
    
            private void resetIfNeeded(Object value) {
                if (this.monitor.update(value)) {
                    reset();
                }
            }
    
            private void reset() {
                this.zuulHandlerMapping.setDirty(true);
            }
        }
    }
    

通过 `@Import` 注解可以找到几个类：

  * RibbonCommandFactoryConfiguration.RestClientRibbonConfiguration
  * RibbonCommandFactoryConfiguration.OkHttpRibbonConfiguration
  * RibbonCommandFactoryConfiguration.HttpClientRibbonConfiguration

我们知道 Zuul 提供网关能力，通过上面这几个类就能分析到，它内部其实也是通过接口请求，找到每个服务提供的接口地址。

进入 RibbonCommandFactoryConfiguration 类：

    
    
    public class RibbonCommandFactoryConfiguration {
        //以下提供了3个不同的请求模式
        @Configuration
        @ConditionalOnRibbonRestClient
        protected static class RestClientRibbonConfiguration {
    
            @Autowired(required = false)
            private Set<ZuulFallbackProvider> zuulFallbackProviders = Collections.emptySet();
    
            @Bean
            @ConditionalOnMissingBean
            public RibbonCommandFactory<?> ribbonCommandFactory(
                    SpringClientFactory clientFactory, ZuulProperties zuulProperties) {
                return new RestClientRibbonCommandFactory(clientFactory, zuulProperties,
                        zuulFallbackProviders);
            }
        }
    
        @Configuration
        @ConditionalOnRibbonOkHttpClient
        @ConditionalOnClass(name = "okhttp3.OkHttpClient")
        protected static class OkHttpRibbonConfiguration {
    
            @Autowired(required = false)
            private Set<ZuulFallbackProvider> zuulFallbackProviders = Collections.emptySet();
    
            @Bean
            @ConditionalOnMissingBean
            public RibbonCommandFactory<?> ribbonCommandFactory(
                    SpringClientFactory clientFactory, ZuulProperties zuulProperties) {
                return new OkHttpRibbonCommandFactory(clientFactory, zuulProperties,
                        zuulFallbackProviders);
            }
        }
    
        @Configuration
        @ConditionalOnRibbonHttpClient
        protected static class HttpClientRibbonConfiguration {
    
            @Autowired(required = false)
            private Set<ZuulFallbackProvider> zuulFallbackProviders = Collections.emptySet();
    
            @Bean
            @ConditionalOnMissingBean
            public RibbonCommandFactory<?> ribbonCommandFactory(
                    SpringClientFactory clientFactory, ZuulProperties zuulProperties) {
                return new HttpClientRibbonCommandFactory(clientFactory, zuulProperties, zuulFallbackProviders);
            }
        }
    
        @Target({ ElementType.TYPE, ElementType.METHOD })
        @Retention(RetentionPolicy.RUNTIME)
        @Documented
        @Conditional(OnRibbonHttpClientCondition.class)
        @interface ConditionalOnRibbonHttpClient { }
    
        private static class OnRibbonHttpClientCondition extends AnyNestedCondition {
            public OnRibbonHttpClientCondition() {
                super(ConfigurationPhase.PARSE_CONFIGURATION);
            }
    
            @Deprecated //remove in Edgware"
            @ConditionalOnProperty(name = "zuul.ribbon.httpclient.enabled", matchIfMissing = true)
            static class ZuulProperty {}
    
            @ConditionalOnProperty(name = "ribbon.httpclient.enabled", matchIfMissing = true)
            static class RibbonProperty {}
        }
    
        @Target({ ElementType.TYPE, ElementType.METHOD })
        @Retention(RetentionPolicy.RUNTIME)
        @Documented
        @Conditional(OnRibbonOkHttpClientCondition.class)
        @interface ConditionalOnRibbonOkHttpClient { }
    
        private static class OnRibbonOkHttpClientCondition extends AnyNestedCondition {
            public OnRibbonOkHttpClientCondition() {
                super(ConfigurationPhase.PARSE_CONFIGURATION);
            }
    
            @Deprecated //remove in Edgware"
            @ConditionalOnProperty("zuul.ribbon.okhttp.enabled")
            static class ZuulProperty {}
    
            @ConditionalOnProperty("ribbon.okhttp.enabled")
            static class RibbonProperty {}
        }
    
        @Target({ ElementType.TYPE, ElementType.METHOD })
        @Retention(RetentionPolicy.RUNTIME)
        @Documented
        @Conditional(OnRibbonRestClientCondition.class)
        @interface ConditionalOnRibbonRestClient { }
    
        private static class OnRibbonRestClientCondition extends AnyNestedCondition {
            public OnRibbonRestClientCondition() {
                super(ConfigurationPhase.PARSE_CONFIGURATION);
            }
    
            @Deprecated //remove in Edgware"
            @ConditionalOnProperty("zuul.ribbon.restclient.enabled")
            static class ZuulProperty {}
    
            @ConditionalOnProperty("ribbon.restclient.enabled")
            static class RibbonProperty {}
        }   
    }
    

### 总结

前面带领大家分析了一小段源码，Spring Cloud
很庞大，不可能一一分析，本文的主要目的就是教大家如何分析源码，从何处下手，以便大家可以按照这种思路继续跟踪下去。

