### 前言

上节课我们实现了服务提供者 provider，接下来应该实现消费者 consumer 了，在实现服务消费者之前，我们先来学习 RestTemplate
的使用，通过 RestTemplate 可以实现不同微服务之间的调用。

### 什么是 REST

REST 是当前比较流行的一种互联网软件架构模型，通过统一的规范完成不同终端的数据访问和交互，REST 是一个词组的缩写，全称为
Representational State Transfer，翻译成中文的意思是资源表现层状态转化。

#### 特点

1\. URL 传参更加简洁，如下所示：

  * 非 RESTful 的 URL：http://...../queryUserById?id=1
  * RESTful 的 URL：http://..../queryUserById/1

2\. 完成不同终端之间的资源共享，RESTful 提供了一套规范，不同终端之间只需要遵守该规范，就可以实现数据交互。

Restful 具体来讲就是四种表现形式，HTTP 协议中四种请求类型（GET、POST、PUT、DELETE）分别表示四种常规操作，即 CRUD：

  * GET 用来获取资源
  * POST 用来创建资源
  * PUT 用来修改资源
  * DELETE 用来删除资源

### 什么是 RestTemplate

RestTemplate 是 Spring 框架提供的基于 REST 的服务组件，底层对 HTTP 请求及响应进行了封装，提供了很多访问远程 REST
服务的方法，可简化代码开发。

### 如何使用 RestTemplate

1\. 通过 Spring Boot 快速搭建一个 REST 服务，新建 Maven 工程，pom.xml 引入相关依赖。

    
    
    <parent>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-parent</artifactId>
       <version>2.0.7.RELEASE</version>
    </parent>
    
    <dependencies>
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-web</artifactId>
       </dependency>
    </dependencies>
    

2\. 使用 Lombok 来简化实体类的代码编写，在 pom.xml 中添加 Lombok 依赖。

    
    
    <dependency>
       <groupId>org.projectlombok</groupId>
       <artifactId>lombok</artifactId>
       <optional>true</optional>
    </dependency>
    

3\. 使用 Lombok 需要预先在 IDE 中安装 Lombok 插件，我们以 IDEA 为例，安装步骤如下图所示。

![1](https://images.gitbook.cn/3027f130-cce2-11e9-beb5-a53251e30de8)

4\. 创建 User 类，并添加 Lombok 相关注解。

    
    
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public class User {
       private Long id;
       private String name;
       private Integer age;
    }
    

5\. 创建 UserRepository 接口及实现类，实现 CRUD 方法，这里使用 Map 集合来存储数据。

    
    
    public interface UserRepository {
       public Collection<User> findAll();
       public User findById(Long id);
       public void saveOrUpdate(User user);
       public void deleteById(Long id);
    }
    
    
    
    @Repository
    public class UserRepositoryImpl implements UserRepository {
    
       private Map<Long,User> userMap;
    
       public UserRepositoryImpl(){
           userMap = new HashMap<>();
           userMap.put(1L,new User(1L,"张三",21));
           userMap.put(2L,new User(2L,"李四",22));
           userMap.put(3L,new User(3L,"王五",23));
       }
    
       @Override
       public Collection<User> findAll() {
           return userMap.values();
       }
    
       @Override
       public User findById(Long id) {
           return userMap.get(id);
       }
    
       @Override
       public void saveOrUpdate(User user) {
           userMap.put(user.getId(),user);
       }
    
       @Override
       public void deleteById(Long id) {
           userMap.remove(id);
       }
    }
    

6\. 创建 UserHandler 类，实现 CRUD 相关业务方法。

    
    
    @RestController
    @RequestMapping("/user")
    public class UserHandler {
    
       @Autowired
       private UserRepository userRepository;
    
       @GetMapping("/findAll")
       public Collection<User> findAll(){
           return userRepository.findAll();
       }
    
       @GetMapping("/findById/{id}")
       public User findById(@PathVariable("id") Long id){
           return userRepository.findById(id);
       }
    
       @PostMapping("/save")
       public Collection<User> save(@RequestBody User user){
           userRepository.saveOrUpdate(user);
           return userRepository.findAll();
       }
    
       @PutMapping("/update")
       public void update(@RequestBody User user){
           userRepository.saveOrUpdate(user);
       }
    
       @DeleteMapping("/deleteById/{id}")
       public void deleteById(@PathVariable("id") Long id){
           userRepository.deleteById(id);
       }
    }
    

7\. 创建启动类 Application。

    
    
    @SpringBootApplication
    public class Application {
       public static void main(String[] args) {
           SpringApplication.run(Application.class,args);
       }
    }
    

8\. 创建配置文件 application.yml，指定服务端口。

    
    
    server:
     port: 8080
    

9\. 启动 Application, REST服务搭建完成并访问 http://localhost:8080/user/findAll，看到如下结果。

![2](https://images.gitbook.cn/402b8560-cce2-11e9-beb5-a53251e30de8)

现在使用 RestTemplate 来访问搭建好的 REST 服务。

10\. 实例化 RestTemplate 对象并通过 @Bean 注入 IoC，在启动类 Application 中添加代码。

    
    
    @SpringBootApplication
    public class Application {
       public static void main(String[] args) {
           SpringApplication.run(Application.class,args);
       }
    
       @Bean
       public RestTemplate createRestTemplate(){
           return new RestTemplate();
       }
    }
    

11\. 创建 RestHandler 类，通过 @Autowired 将 IoC 容器中的 RestTemplate 实例对象注入
RestHandler，在业务方法中就可以通过 RestTemplate 来访问 REST 服务了。

    
    
    @RestController
    @RequestMapping("/rest")
    public class RestHandler {
    
       @Autowired
       private RestTemplate restTemplate;
    
       //业务方法...
    
    }
    

下面演示调用 RestTemplate 的相关方法，分别通过 GET、POST、PUT、DELETE 请求，访问服务资源。

#### GET

方法描述：

    
    
    public <T> ResponseEntity<T> getForEntity(String url, Class<T> responseType, Object... uriVariables) throws RestClientException {
       RequestCallback requestCallback = this.acceptHeaderRequestCallback(responseType);
       ResponseExtractor<ResponseEntity<T>> responseExtractor = this.responseEntityExtractor(responseType);
       return (ResponseEntity)nonNull(this.execute(url, HttpMethod.GET, requestCallback, responseExtractor, uriVariables));
    }
    

参数解读：

url 为请求的目标资源，responseType 为响应数据的封装模版，uriVariables 是一个动态参数，可以根据实际请求传入参数。

getForEntity 方法的返回值类型为 ResponseEntity，通过调用其 getBody 方法可获取结果对象，具体使用如下所示。

    
    
    @GetMapping("/findAll")
    public Collection<User> findAll(){
       return restTemplate.getForEntity("http://localhost:8080/user/findAll", Collection.class).getBody();
    }
    

上述代码表示访问 http://localhost:8080/user/findAll，并将结果封装为一个 Collection 对象。

如果需要传参，直接将参数追加到 getForEntity 方法的参数列表中即可，如下所示。

    
    
    @GetMapping("/findById/{id}")
    public User findById(@PathVariable("id") Long id){
       return restTemplate.getForEntity("http://localhost:8080/user/findById/{id}", User.class,id).getBody();
    }
    

方法描述：

    
    
    @Nullable
    public <T> T getForObject(String url, Class<T> responseType, Object... uriVariables) throws RestClientException {
       RequestCallback requestCallback = this.acceptHeaderRequestCallback(responseType);
       HttpMessageConverterExtractor<T> responseExtractor = new HttpMessageConverterExtractor(responseType, this.getMessageConverters(), this.logger);
       return this.execute(url, HttpMethod.GET, requestCallback, responseExtractor, (Object[])uriVariables);
    }
    

getForObject 方法的使用与 getForEntity 很类似，唯一的区别在于 getForObject 的返回值就是目标对象，无需通过调用
getBody 方法来获取，具体使用如下所示。

    
    
    @GetMapping("/findAll2")
    public Collection<User> findAll2(){
       return restTemplate.getForObject("http://localhost:8080/user/findAll", Collection.class);
    }
    

如果需要传参，直接将参数追加到 getForObject 方法的参数列表中即可，如下所示。

    
    
    @GetMapping("/findById2/{id}")
    public User findById2(@PathVariable("id") Long id){
       return restTemplate.getForObject("http://localhost:8080/user/findById/{id}", User.class,id);
    }
    

#### POST

方法描述：

    
    
    public <T> ResponseEntity<T> postForEntity(String url, @Nullable Object request, Class<T> responseType, Object... uriVariables) throws RestClientException {
       RequestCallback requestCallback = this.httpEntityCallback(request, responseType);
       ResponseExtractor<ResponseEntity<T>> responseExtractor = this.responseEntityExtractor(responseType);
       return (ResponseEntity)nonNull(this.execute(url, HttpMethod.POST, requestCallback, responseExtractor, uriVariables));
    }
    

参数解读：

url 为请求的目标资源，request 为要保存的目标对象，responseType 为响应数据的封装模版，uriVariables
是一个动态参数，可以根据实际请求传入参数。

postForEntity 方法的返回值类型也是 ResponseEntity，通过调用其 getBody 方法可获取结果对象，具体使用如下所示。

    
    
    @PostMapping("/save")
    public Collection<User> save(@RequestBody User user){
       return restTemplate.postForEntity("http://localhost:8080/user/save",user,Collection.class).getBody();
    }
    

方法描述：

    
    
    @Nullable
    public <T> T postForObject(String url, @Nullable Object request, Class<T> responseType, Object... uriVariables) throws RestClientException {
       RequestCallback requestCallback = this.httpEntityCallback(request, responseType);
       HttpMessageConverterExtractor<T> responseExtractor = new HttpMessageConverterExtractor(responseType, this.getMessageConverters(), this.logger);
       return this.execute(url, HttpMethod.POST, requestCallback, responseExtractor, (Object[])uriVariables);
    }
    

postForObject 方法的使用与 postForEntity 类似，唯一的区别在于 postForObject 的返回值就是目标对象，无需通过调用
getBody 方法来获取，具体使用如下所示。

    
    
    @PostMapping("/save2")
    public Collection<User> save2(@RequestBody User user){
       return restTemplate.postForObject("http://localhost:8080/user/save",user,Collection.class);
    }
    

#### PUT

方法描述：

    
    
    public void put(String url, @Nullable Object request, Object... uriVariables) throws RestClientException {
       RequestCallback requestCallback = this.httpEntityCallback(request);
       this.execute(url, HttpMethod.PUT, requestCallback, (ResponseExtractor)null, (Object[])uriVariables);
    }
    

参数解读：

url 为请求的目标资源，request 为要修改的目标对象，uriVariables 是一个动态参数，可以根据实际请求传入参数，具体使用如下所示。

    
    
    @PutMapping("/update")
    public void update(@RequestBody User user){
       restTemplate.put("http://localhost:8080/user/update",user);
    }
    

#### DELETE

方法描述：

    
    
    public void delete(String url, Object... uriVariables) throws RestClientException {
       this.execute(url, HttpMethod.DELETE, (RequestCallback)null, (ResponseExtractor)null, (Object[])uriVariables);
    }
    

参数解读：

url 为请求的目标资源，uriVariables 是一个动态参数，可以根据实际请求传入参数，具体使用如下所示。

    
    
    @DeleteMapping("/deleteById/{id}")
    public void delete(@PathVariable("id") Long id){
       restTemplate.delete("http://localhost:8080/user/deleteById/{id}",id);
    }
    

RestHandler 完整代码如下：

    
    
    @RestController
    @RequestMapping("/rest")
    public class RestHandler {
    
       @Autowired
       private RestTemplate restTemplate;
    
       @GetMapping("/findAll")
       public Collection<User> findAll(){
           return restTemplate.getForEntity("http://localhost:8080/user/findAll", Collection.class).getBody();
       }
    
       @GetMapping("/findAll2")
       public Collection<User> findAll2(){
           return restTemplate.getForObject("http://localhost:8080/user/findAll", Collection.class);
       }
    
       @GetMapping("/findById/{id}")
       public User findById(@PathVariable("id") Long id){
           return restTemplate.getForEntity("http://localhost:8080/user/findById/{id}", User.class,id).getBody();
       }
    
       @GetMapping("/findById2/{id}")
       public User findById2(@PathVariable("id") Long id){
           return restTemplate.getForObject("http://localhost:8080/user/findById/{id}", User.class,id);
       }
    
       @PostMapping("/save")
       public Collection<User> save(@RequestBody User user){
           return restTemplate.postForEntity("http://localhost:8080/user/save",user,Collection.class).getBody();
       }
    
       @PostMapping("/save2")
       public Collection<User> save2(@RequestBody User user){
           return restTemplate.postForObject("http://localhost:8080/user/save",user,Collection.class);
       }
    
       @PutMapping("/update")
       public void update(@RequestBody User user){
           restTemplate.put("http://localhost:8080/user/update",user);
       }
    
       @DeleteMapping("/deleteById/{id}")
       public void delete(@PathVariable("id") Long id){
           restTemplate.delete("http://localhost:8080/user/deleteById/{id}",id);
       }
    }
    

重新启动 Application ，通过 Postman 工具来分别测试 RestHandler 的业务方法，如下所示。

  * findAll 接口

![3](https://images.gitbook.cn/99630770-cce2-11e9-8d89-4fa271cb1633)

  * findAll2 接口

![4](https://images.gitbook.cn/a1095790-cce2-11e9-9a11-bbb3551196dc)

  * findById 接口

![5](https://images.gitbook.cn/a7c1c540-cce2-11e9-9f23-07a3e2a236db)

  * findById2 接口

![6](https://images.gitbook.cn/afc06e90-cce2-11e9-beb5-a53251e30de8)

  * save 接口

![7](https://images.gitbook.cn/b73b9000-cce2-11e9-9f23-07a3e2a236db)

  * save2 接口

![8](https://images.gitbook.cn/c1293810-cce2-11e9-beb5-a53251e30de8)

  * update 接口

![8](https://images.gitbook.cn/c84cc3a0-cce2-11e9-8d89-4fa271cb1633)

修改完成之后再次查询，调用 findAll 接口，可以看到数据已经修改。

![9](https://images.gitbook.cn/cffa8dd0-cce2-11e9-8d89-4fa271cb1633)

  * deleteById 接口

![10](https://images.gitbook.cn/d85e6190-cce2-11e9-8d89-4fa271cb1633)

删除完成之后再次查询，调用 findAll 接口，可以看到数据已经被删除。

![3](https://images.gitbook.cn/e00db260-cce2-11e9-beb5-a53251e30de8)

### 总结

本节课我们讲解了 RestTemplate 的使用，RestTemplate 底层对 HTTP 请求及响应进行了封装，提供了很多访问远程 REST
服务的方法，基于它的这个特性，我们可以实现不同微服务之间的调用。下节课我们将一起来实现服务消费者 consumer，并通过 RestTemplate
来调用服务提供者 provider 的相关接口。

[请点击这里查看源码](https://github.com/southwind9801/resttemplate.git)

[点击这里获取 Spring Cloud
视频专题](https://pan.baidu.com/s/1P_3n6KnPdWBFnlAtEdTm2g)，提取码：yfq2

