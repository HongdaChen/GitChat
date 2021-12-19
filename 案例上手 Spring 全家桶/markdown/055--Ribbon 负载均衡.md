### 前言

在前面的课程中我们已经通过 RestTemplate
实现了服务消费者对服务提供者的调用，这只是实现了最基本的需求，如果在某个具体的业务场景下，对于某服务的调用需求激增，这时候我们就需要为该服务实现负载均衡以满足高并发访问，在一个大型的分布式应用系统中，负载均衡(Load
Balancing)是必备的。

### 什么是 Ribbon？

Spring Cloud 提供了实现负载均衡的解决方案：Spring Cloud Ribbon，Ribbon 是 Netflix 发布的负载均衡器，而
Spring Cloud Ribbon 则是基于 Netflix Ribbon 实现的，是一个用于对 HTTP 请求进行控制的负载均衡客户端。

Spring Cloud Ribbon 官网地址：

> <http://cloud.spring.io/spring-cloud-netflix/multi/multi_spring-cloud-
> ribbon.html>

Ribbon 的使用同样需要结合 Eureka Server，即需要将 Ribbon 在 Eureka Server 进行注册，注册完成之后，就可以通过
Ribbon 结合某种负载均衡算法，如轮询、随机、加权轮询、加权随机等帮助服务消费者去调用接口。除了 Ribbon
默认提供的这些负载均衡算法外，开发者也可以根据具体需求来设计自定义的 Ribbon 负载均衡算法。实际开发中，Spring Cloud Ribbon
需要结合 Spring Cloud Eureka 来使用，Eureka Server 提供所有可调用的服务提供者列表，Ribbon
基于特定的负载均衡算法从这些服务提供者中挑选出要调用的实例，如下图所示。

![1](https://images.gitbook.cn/4db698e0-d25b-11e9-bcae-b7c2737c8da6)

Ribbon 常用的负载均衡策略有以下几种：

  * 随机：访问服务时，随机从注册中心的服务列表中选择一个。
  * 轮询：当同时启动两个服务提供者时，客户端请求会由这两个服务提供者交替处理。
  * 加权轮询：对服务列表中的所有微服务响应时间做加权处理，并以轮询的方式来访问这些服务。
  * 最大可用：从服务列表中选择并发访问量最小的那个微服务。

了解完原理，接下来我们一起实现 Ribbon。

1\. 在父工程下创建 Module。

![2](https://images.gitbook.cn/7a240a70-d25b-11e9-bcae-b7c2737c8da6)

2\. 输入 ArtifactId，点击 Next。

![3](https://images.gitbook.cn/8311f5c0-d25b-11e9-b943-9d5bb2abdc80)

3\. 设置工程名和工程存放路径，点击 Finish。

![4](https://images.gitbook.cn/8b809ef0-d25b-11e9-84ba-0bd4ba7d7fb3)

4\. 在 pom.xml 中添加 Eureka Client 依赖。

    
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
    </dependencies>
    

5\. 在 resources 路径下创建配置文件 application.yml，添加 Ribbon 相关配置。

    
    
    server:
      port: 8040
    spring:
      application:
        name: ribbon
    eureka:
      client:
        service-url:
          defaultZone: http://localhost:8761/eureka/
      instance:
        prefer-ip-address: true
    

属性说明：

  * server.port：当前 Ribbon 服务端口。
  * spring.application.name：当前服务注册在 Eureka Server 上的名称。
  * eureka.client.service-url.defaultZone：注册中心的访问地址。
  * eureka.instance.prefer-ip-address：是否将当前服务的 IP 注册到 Eureka Server。

6\. 在 java 路径下创建启动类 RibbonApplication，并通过 @Bean 注解注入 RestTemplate
实例，@LoadBalanced 注解提供负载均衡。

    
    
    @SpringBootApplication
    public class RibbonApplication {
        public static void main(String[] args) {
            SpringApplication.run(RibbonApplication.class,args);
        }
    
        @Bean
        @LoadBalanced
        public RestTemplate restTemplate(){
            return new RestTemplate();
        }
    }
    

注解说明：

  * @SpringBootApplication： 声明该类是 Spring Boot 服务的入口。
  * @LoadBalanced：声明一个基于 Ribbon 的负载均衡。

7\. 接下来让 Ribbon 调用 Provider 服务，首先创建对应的实体类 Student。

    
    
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public class Student {
        private long id;
        private String name;
        private char gender;
    }
    

8\. 创建 RibbonHandler，完成相关业务，通过 RestTemplate 调用 Provider 相关接口，RestTemplate 访问的
URL 不需要指定 IP 和 端口，直接访问 Provider 在 Eureka Server 注册的 application name
即可，比如：http://localhost:8010/ 可替换为 http://provider/。

    
    
    @RequestMapping("/ribbon")
    @RestController
    public class RibbonHandler {
    
        @Autowired
        private RestTemplate restTemplate;
    
        @GetMapping("/findAll")
        public Collection<Student> findAll(){
            return restTemplate.getForObject("http://provider/student/findAll",Collection.class);
        }
    
        @GetMapping("/findById/{id}")
        public Student findById(@PathVariable("id") long id){
            return restTemplate.getForObject("http://provider/student/findById/{id}",Student.class,id);
        }
    
        @PostMapping("/save")
        public void save(@RequestBody Student student){
            restTemplate.postForObject("http://provider/student/save",student,Student.class);
        }
    
        @PutMapping("/update")
        public void update(@RequestBody Student student){
            restTemplate.put("http://provider/student/update",student);
        }
    
        @DeleteMapping("/deleteById/{id}")
        public void deleteById(@PathVariable("id") long id){
            restTemplate.delete("http://provider/student/deleteById/{id}",id);
        }
    }
    

9\. 依次启动注册中心、Provider、Ribbon，启动成功控制台输出如下信息。

![5](https://images.gitbook.cn/d07adac0-d25b-11e9-84ba-0bd4ba7d7fb3)

10\. 打开浏览器，访问 http://localhost:8761，看到如下界面。

![6](https://images.gitbook.cn/d8d16810-d25b-11e9-8d0f-6b56ebcd1907)

11\. 可以看到 Provider 和 Ribbon 已经在注册中心完成注册，接下来用 Postman 工具测试 Ribbon 相关接口，如下图所示。

  * findAll 接口

![7](https://images.gitbook.cn/2f172160-d25c-11e9-b943-9d5bb2abdc80)

  * findById 接口

![8](https://images.gitbook.cn/35b6d6f0-d25c-11e9-b943-9d5bb2abdc80)

  * save 接口

![9](https://images.gitbook.cn/3be41ba0-d25c-11e9-84ba-0bd4ba7d7fb3)

添加完成之后再来查询，调用 findAll 接口，可以看到新数据已经添加成功。

![10](https://images.gitbook.cn/44215490-d25c-11e9-8d0f-6b56ebcd1907)

  * update 接口

![11](https://images.gitbook.cn/4b1a4db0-d25c-11e9-b943-9d5bb2abdc80)

修改完成之后再来查询，调用 findAll 接口，可以看到修改之后的数据。

![12](https://images.gitbook.cn/528fc9d0-d25c-11e9-bcae-b7c2737c8da6)

  * deleteById 接口

![13](https://images.gitbook.cn/58aba960-d25c-11e9-8d0f-6b56ebcd1907)

删除完成之后再来查询，调用 findAll 接口，可以看到数据已经被删除。

![7](https://images.gitbook.cn/5f559820-d25c-11e9-bcae-b7c2737c8da6)

12\. 接下来测试 Ribbon 的负载均衡，在 RibbonHandler 中添加如下代码。

    
    
    @RequestMapping("/ribbon")
    @RestController
    public class RibbonHandler {
    
        @GetMapping("/index")
        public String index(){
            return restTemplate.getForObject("http://provider/student/index",String.class);
        }
    
    }
    

13\. 分别启动注册中心，端口为 8010 的 Provider，端口为 8011 的 Provider、Ribbon，如下图所示。

![14](https://images.gitbook.cn/7034e650-d25c-11e9-bcae-b7c2737c8da6)

14\. 打开浏览器，访问 http://localhost:8761，可看到如下界面。

![15](https://images.gitbook.cn/77a09e70-d25c-11e9-8d0f-6b56ebcd1907)

15\. 可以看到两个 Provider 和 Ribbon 已经在注册中心完成注册，通过 Postman 访问
http://localhost:8040/ribbon/index，交替出现下图所示情况，实现了负载均衡。

![16](https://images.gitbook.cn/90b49a10-d25c-11e9-84ba-0bd4ba7d7fb3)

![17](https://images.gitbook.cn/97597fc0-d25c-11e9-bcae-b7c2737c8da6)

### 总结

本节课我们讲解了使用 Ribbon 来实现服务调用负载均衡的具体操作，Ribbon 是 Netflix 发布的负载均衡器，Ribbon
的功能是结合某种负载均衡算法，如轮询、随机、加权轮询、加权随机等帮助服务消费者去调用接口，同时也可以自定义 Ribbon 的负载均衡算法。

[请点击这里查看源码](https://github.com/southwind9801/myspringclouddemo.git)

[点击这里获取 Spring Cloud
视频专题](https://pan.baidu.com/s/1P_3n6KnPdWBFnlAtEdTm2g)，提取码：yfq2

