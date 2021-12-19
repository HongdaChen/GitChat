### 问题引入

#### **现状分析**

统一了接口的应答格式，那么需不需要对请求做统一的处理呢？这个问题需要从以下几个方面进行分析：

代码重复性：

  * 接口文档中大多数接口都需要用户上传 Token 进行身份校验，是否每个接口都要处理 Token 参数？

安全性：

  * 所有的接口方法都对外开放会不会有安全性问题？
  * 需不需要将一部分接口对外隔离，仅限内部访问？

可扩展性：

  * 现在服务端架构发展的趋势是微服务，如果项目需要改为微服务架构，是否每个接口都要进行改造？
  * 是否能在改动少许代码的情况下将项目做集群部署以支持大用户量？

目前的架构还不能满足以上这些要求，因此，本篇带领大家做统一的请求处理，使项目安全稳定、扩展性强、开发简单，满足现有开发需求的同时为以后大型项目做准备。

#### **期望结果**

针对以上考虑，我们期望：

  * 有一个代理的接口对外接收所有请求，并验证用户身份及访问权限
  * 验证通过后将请求分发给相应的接口进行业务处理，拿到处理结果后返回给客户端
  * 否则，直接拒绝访问

这样做的好处是：

  * 所有实际业务接口只负责处理业务，不需要关心用户身份和访问权限等问题，业务代码量大大减小
  * 仅代理接口对外开发，避免了项目过多接口的暴露，安全性得到保证
  * 以后做集群部署或微服务改造时仅需改动代理接口相关的代码，业务代码不需要改动，项目扩展性比较好

### 知识准备

上一篇，使用 Spring-web 组件中 org.springframework.http 包中的一些类实现了统一应答，本篇同样用到了 Spring-
web 组件中的内容。

#### **RestTemplate**

RestTemplate 在 Spring-web 组件的 org.springframework.web.client 包中，用于快捷方便地访问
RESTful 接口，在 Spring Cloud 项目中经常会用它进行不同组件间的 HTTP 访问，源码如图：

![RestTemplate](https://images.gitbook.cn/fdf31d60-7250-11ea-b8cb-b108c7f0b146)

使用方式也比较简单，不需要单独再引入 jar 包，直接在项目的启动类中定义一个 Bean 就可以了，代码如下：

    
    
    @SpringBootApplication
    public class KellerApplication {
        @Bean
        /**
         * 引入RestTemplate Bean
         * 用来进行服务间的Http通信
         * 同时重新定义其解析时用到的字符集，防止中文乱码
         */
        RestTemplate restTemplate(){
              //创建一个 RestTemplate 实例
            RestTemplate restTemplate = new RestTemplate();
              //清除掉原有的消息转换器，因为这些转换器处理中文字符的能力有限，比较容易出现乱码
            restTemplate.getMessageConverters().clear();
              //为 RestTemplate 实例指定 FastJson 的消息转换器
            restTemplate.getMessageConverters().add(new FastJsonHttpMessageConverter());
              //返回配置好的 RestTemplate 实例
            return restTemplate;
        }
    
        public static void main(String[] args) {
            SpringApplication.run(KellerApplication.class, args);
        }
    }
    

在这里需要注意的是，RestTemplate 默认使用 jackson 或 gson 处理 JSON 数据，可能产生中文乱码，最好替换为 FastJson。

配置好后，按如下方式就可以快捷地访问 RESTful 接口了：

    
    
    //GET 
    restTemplate.getForEntity("URL", String.class,params);
    //POST 
    restTemplate.postForEntity("URL",request,String.class,params);
    //PUT 
    restTemplate.put("URL",request,params);
    //DELETE 
    restTemplate.delete("URL",params);
    

#### **FastJson**

FastJson 是阿里巴巴的开源 JSON 解析库，它可以解析 JSON 格式的字符串，支持将 Java Bean 序列化为 JSON 字符串，也可以从
JSON 字符串反序列化到 JavaBean。在本项目中将其作为 JSON 解析库，引入方式为：

    
    
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.51</version>
    </dependency>
    

FastJson 中文文档地址：

> <https://github.com/alibaba/fastjson/wiki/Quick-Start-CN>，如图：

![FastJson](https://images.gitbook.cn/0905e480-7251-11ea-972c-452d07ff8220)

#### **ApplicationRunner**

SpringBoot 项目由启动类进行启动，有时希望项目在启动的时候加载一些系统参数（如：项目运行的端口等），就要用到 ApplicationRunner。

具体用法为：

  1. 新建一个类，实现 ApplicationRunner 接口
  2. 重写 run 方法，该方法中写希望在项目启动时执行的代码
  3. 在类上添加 @Component 注解，这样该类就会被添加到 Spring 容器中
  4. 如果有多个类都想在启动时执行，需要在每个类上添加 @Order 注解，该注解指定了多个类启动的顺序

示例代码如下：

    
    
    @Component
    @Order(value = 1)
    public class KellerRunner implements ApplicationRunner {
        @Resource
        private CommonConfig config;
    
        @Override
        public void run(ApplicationArguments args) throws Exception {
            RequestUtil.port = config.port;
            RequestUtil.address = config.address;
        }
    }
    

### 方案分析

#### **流程设计**

统一请求的流程图如下：

![流程图](https://images.gitbook.cn/12002be0-7251-11ea-8377-13f07d2f46fb)

**1\. 客户端的所有请求由代理类统一处理；这里的请求要区分多种情况：**

  * 需要身份认证后才能发起的请求，如：获取个人信息
  * 不需要身份认证就能发起的请求，如：注册、登录
  * Web 表单格式的请求
  * JSON 格式的请求

**2\. 代理类对请求进行校验，若校验成功，将请求转发到实际的业务类处理；这时需要：**

  * 服务运行的 IP 地址、端口号
  * 获取到请求的方式（GET/POST/PUT/DELETE）
  * 获取到请求的参数名和值
  * 获取到请求的 URL

**3\. 若校验不成功，直接返回相应的错误码给客户端。**

**4\. 代理类获取到业务类的返回结果，并将其返回给客户端。**

按照以上流程，做出以下解决方案。

#### **总体方案**

  1. 通过 **ApplicationRunner 机制** 获取到项目启动时的系统参数
  2. 创建不同的代理类分别处理 **JSON** 和 **表单** 格式的请求
  3. 使用 **RestTemplate** 对请求进行转发，并处理返回结果
  4. 记录必要的请求日志

### 代码实现

#### **获取系统参数**

Spring Boot 项目的系统配置都在 application.properties 文件中，如下：

    
    
    ## 端口
    server.port=8080
    

在项目中要读取这些配置，需要按照以下步骤：

  * 新建一个类，使用 `@PropertySource` 注解指定要读取的配置文件名称
  * 在属性上使用 `@Value` 注解指定要对应的配置项及默认值
  * 在类上使用 `@Configuration` 注解将其添加到 Spring 容器中

在本项目中，新建一个 **CommonConfig** 类用于读取所有的系统配置，具体代码如下：

    
    
    /**
     * 项目配置文件，从 application.properties 中加载
     * @author yangkaile
     * @date 2019-05-30 09:38:05
     */
    @Configuration
    @PropertySource("classpath:application.properties")
    public class CommonConfig {
        @Value("${server.port:8080}")
        public String port;
    
        @Value("${server.address:http://127.0.0.1}")
        public String address;
            …… ……
    }
    

到这一步，就可以在项目的所有类中通过 `@Resource` 注解注入 **CommonConfig**
类的实例从而实现读取所有系统配置的功能了，但是不能作为静态变量使用。

因为系统配置会有很多，而统一请求中需要的只是其中的两个（项目运行的 IP 地址 address、项目运行的端口号
port），且想作为静态变量使用。因此，可以新建一个 **RequestUtil**
类用于处理所有与请求有关的工作，如：address、port、请求日志记录、请求参数解析等。

    
    
    public class RequestUtil {
        public static String port;
        public static String address;
    }
    

在 **RequestUtil** 类中声明好静态变量后，需要在项目启动时赋值，就用到了 **ApplicationRunner** 。新建
**KellerRunner** 类，代码如下：

    
    
    /**
     * 继承 Application 接口后项目启动时会按照执行顺序执行 run 方法
     * 通过设置 Order 的 value 来指定执行的顺序
     */
    @Component
    @Order(value = 1)
    public class KellerRunner implements ApplicationRunner {
    
          //引入 CommonConfig 实例，就可以读取到所有的系统配置了
        @Resource
        private CommonConfig config;
    
          //重写 ApplicationRunner 中的 run 方法，在该方法中给 RequestUtil 类中的静态变量赋值 
        @Override
        public void run(ApplicationArguments args) throws Exception {
            RequestUtil.port = config.port;
            RequestUtil.address = config.address;
        }
    }
    

在 **KellerRunner** 类中，通过注入 **CommonConfig** 实例的方式获取到系统配置中的项目运行地址及端口号，并用其初始化
**RequestUtil** 中的静态变量。

至此，就可以在项目的代码中使用 RequestUtil.port 的方式读取到系统配置中与请求处理相关的参数了。

#### **创建代理类**

获取到必要的参数后，就可以创建代理类了，通过上面分析过：代理类要区分 JSON 格式和表单格式的请求，因此，在这里新建两个代理类：

**ApiController** 类用于处理 JSON 格式的请求：

    
    
    @RestController
    @RequestMapping("/api")
    @CrossOrigin(origins = "*",allowedHeaders="*", maxAge = 3600)
    public class ApiController {
        @Resource
        HttpServletRequest request;
    
        @Autowired
        RestTemplate restTemplate;
    
          @GetMapping
        public ResponseEntity get(@RequestBody Map<String,String> params){
            ... ...
        }
    }
    

**FormController** 类用于处理表单格式的请求：

    
    
    @RestController
    @RequestMapping("/form")
    @CrossOrigin(origins = "*",allowedHeaders="*", maxAge = 3600)
    public class FormController {
        @Resource
        HttpServletRequest request;
    
        @Autowired
        RestTemplate restTemplate;
          @GetMapping
        public ResponseEntity get(){
            ... ...  
        }
    }
    

这两个类的主要区别在与对请求的处理方式不同：Spring 处理 json 格式的请求时，需要用 `@RequestBody` 方式注解将其参数转换为相应的
Java 类型，在这里，将其转换为 Map；Spring 处理表单格式的请求时可以直接从 **HttpServletRequest**
对象中获取到请求的参数。同时 **HttpServletRequest** 中可以获取到不同请求格式的请求头。

在这里需要注意：

  * 要使用 `@RestController` 注解，而不是 `@Controller` 注解，因为这是 RESTful 格式的接口，返回的是 Json 格式的数据
  * 注意添加 `@CrossOrigin` 注解，避免因为服务端导致的跨域问题
  * 可以通过注入 **HttpServletRequest** 的方式读取请求

#### **解析请求**

通过 **HttpServletRequest** 读取到请求之后，需要对请求进行解析，解析的内容包括请求头和请求参数。我们可以将相关方法写在
**RequestUtil** 类中，将其封装为处理 Request 的工具类。

使用 **HttpServletRequest** 对象中的一些方法可以快捷地获取到相应相求头的内容，如：请求地址、请求方式、访问者的 IP
地址和端口号、自定义的请求头等。代码如下：

    
    
    public static HashMap<String,String> getHeader(HttpServletRequest request){
        HashMap<String,String> headerMap = new HashMap<>(16);
        //请求的 URL 地址
        headerMap.put(RequestConfig.URL,request.getRequestURL().toString());
        //请求的资源
        headerMap.put(RequestConfig.URI,request.getRequestURI());
        //请求方式 GET/POST
        headerMap.put(RequestConfig.REQUEST_METHOD,request.getMethod());
    
        //来访者的 IP 地址
        headerMap.put(RequestConfig.REMOTE_ADDR,request.getRemoteAddr());
        //来访者的 HOST
        headerMap.put(RequestConfig.REMOTE_HOST,request.getRemoteHost());
        //来访者的端口
        headerMap.put(RequestConfig.REMOTE_PORT,request.getRemotePort() + "");
        //来访者的用户名
        headerMap.put(RequestConfig.REMOTE_USER,request.getRemoteUser());
    
        //自定义的 Header （接口名）
            headerMap.put(RequestConfig.METHOD,
                                        request.getHeader(RequestConfig.METHOD));
        //自定义的 Header （TOKEN）
        headerMap.put(RequestConfig.TOKEN,
                                        request.getHeader(RequestConfig.TOKEN));
        return headerMap;
    }
    

在 **ApiController** 代理类中，通过 `@RequestBody` 注解将 Json 格式的请求参数转换为了 Map，在这里需要处理的是
**FormController** 代理类中表单格式的请求。通过 **HttpServletRequest** 对象中的
`getParameterMap()` 方法将请求中的参数获取到，将其转换为方便处理的 `Map<String,String>` 类型。具体代码如下：

    
    
    public static Map<String,String> getParam(HttpServletRequest request){
        Map<String,String> paramMap = new HashMap<>(16);
        //request 对象封装的参数是以 Map 的形式存储的
        Map<String, String[]> map = request.getParameterMap();
        for(Map.Entry<String, String[]> entry :map.entrySet()){
            String paramName = entry.getKey();
            String paramValue = "";
            String[] paramValueArr = entry.getValue();
            for (int i = 0; paramValueArr!=null && i < paramValueArr.length; i++) {
                if (i == paramValueArr.length-1) {
                    paramValue += paramValueArr[i];
                }else {
                    paramValue += paramValueArr[i]+",";
                }
            }
            paramMap.put(paramName,paramValue);
        }
        if(paramMap.size() == 0){
            return null;
        }
        return paramMap;
    }
    

获取到请求头和请求参数后就可以根据两者拼接出实际的请求地址，代码如下：

    
    
    public static String getUrl(Map<String,String> params,HttpServletRequest request){
        //读取请求头
        HashMap<String,String> headers = getHeader(request);
        StringBuilder builder = new StringBuilder();
            //拼接请求地址、端口号、请求方法
        builder
                .append(address).append(":")
                .append(port).append("/")
                .append(headers.get(RequestConfig.METHOD));
        if(params == null){
            return builder.toString();
        }
          //拼接请求参数
        builder.append("?");
        for(String key :params.keySet()){
            builder.append(key)
                    .append("={")
                    .append(key)
                    .append("}&");
        }
        Console.info(builder.toString());
        return builder.toString();
    }
    

至此，完成了代理类中请求地址、请求头、请求参数的解析，以及源请求到目标请求的转换工作。

#### **转发请求**

从源请求转换到目标请求后，只需要将转发到目标请求并获取到应答，将应答返回给客户端即可，这就用到了 **RestTemplate** 对象。下面是一个使用
**RestTemplate** 对象发送 GET 请求的例子：

    
    
    /**
     * JSON 格式的 GET 请求
     * @param params
     * @return
     */
    @GetMapping
    public ResponseEntity get(@RequestBody Map<String,String> params){
            //定义应答
        ResponseEntity responseEntity;
        try {
              //转发 GET 请求并获取到应答
            responseEntity = restTemplate.getForEntity(RequestUtil.getUrl(params,request), String.class,params);
              //记录请求处理成功的日志  
              RequestUtil.successLog(request,params,responseEntity);
        }catch (Exception e){
            //解析请求失败的应答
              responseEntity = ResponseUtils.getResponseFromException(e);
            //记录请求处理失败的日志
              RequestUtil.errorLog(request,params,responseEntity);
        }
          //返回应答
        return responseEntity;
    }
    

在这里需要说明几点：

  1. 使用 **RestTemplate** 对象发送 HTTP 请求时，如果应答码不是 `200 OK`，会抛出 HttpClientErrorException 异常，需要对该异常进行处理，返回相应的错误码。
  2. 作为比较完善的服务端系统，业务日志是需要记录下来的，以利于后续的系统分析及问题定位。

#### **处理异常请求**

`getResponseFromException(e)` 方法的完整代码如下：

    
    
    public static ResponseEntity getResponseFromException(Exception exception){
        ResponseEntity response;
          // 如果异常属于 HttpClientErrorException 则其状态码代表请求状态码，按照请求状态码返回相应的 ResponseEntity 即可
        if(exception instanceof HttpClientErrorException){
            HttpClientErrorException errorException = (HttpClientErrorException) exception;
            switch (errorException.getStatusCode()){
                case FORBIDDEN:  response = Response.forbidden(); break;
                case BAD_REQUEST: response = Response.badRequest();break;
                case UNAUTHORIZED: response = Response.unauthorized();break;
                case INTERNAL_SERVER_ERROR: response = Response.error();break;
                default:{
                    ResultData resultData = ResultData.error("ERROR");
                    response = ResponseEntity.status(errorException.getStatusCode()).contentType(MediaType.APPLICATION_JSON).body(resultData);
                }
            }
        }else {
            response = Response.badRequest();
        }
        return  response;
    }
    

首先对异常类型进行判断：如果异常属于 HttpClientErrorException 则其状态码代表请求状态码，按照请求状态码返回相应的
ResponseEntity 即可；否则，返回 `400 BadRequest`。

#### **记录日志**

记录日志的时候要注意可能的并发问题，如下代码：

    
    
    public static void successLog(HttpServletRequest request, Map<String,String> params, ResponseEntity response){
          //生成 UUID 标记唯一请求
        String requestId = StringUtils.getUUID();
        //记录请求头
        Console.info("api header",requestId,getHeader(request));
          //记录请求参数
        Console.info("param",requestId,params);
          //记录应答
        Console.info("response success",requestId,response.getBody());
    }
    

用户量稍大的系统中，请求往往是有大量的并发的，因此需要一个唯一的标识来保证请求的唯一性，以保证在日志记录中能将同一个请求的请求头、请求参数、应答对应起来。

至此，统一请求开发完成，所有的接口都可以通过走代理类的方式访问，极大程度上保证了内部接口的安全性。

### 效果测试

测试上一篇 **UserController** 中的 `getByEmail(String email)`
接口，接口不变，将请求方式改为代理类的方式，如下：

    
    
    URL:
        localhost:8080/api
    Headers:
        method:"user/getByEmail"
      Content-Type:"application/json"
    Body:
    {
        "email": "Fwwu9pHew6"
    }
    

在 **Postman** 中测试正常，如图：

![postman](https://images.gitbook.cn/47316b80-7251-11ea-9415-437c2ee98aa7)

### 源码地址

本篇完整的源码地址如下：

> <https://github.com/tianlanlandelan/KellerNotes/tree/master/10.统一请求/server>

### 小结

本篇带领大家完成了对项目统一的请求处理，通过统一的代理类进行请求的转发和应答处理。使项目架构安全稳定、扩展性强、开发便捷，满足现有开发需求的同时为以后大型项目做准备。

