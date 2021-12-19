### 问题引入

#### **现状分析**

经过前两篇的准备工作，我们有了属于自己的 MyBatis 工具包，接下来进入业务功能开发流程。回顾我们的第一个接口，也就是在第 6
篇《后端架构选择及环境搭建》中，在 UserController 类中写了用以验证 Spring Boot + MyBatis 环境是否运行正常以及
RESTful 的测试接口，代码如下：

    
    
            @GetMapping
        public ResponseEntity getAll(){
            List<UserInfo> list = userService.getAll();
            return ResponseEntity.ok() // 状态码
                    .contentType(MediaType.APPLICATION_JSON) //返回内容类型
                    .body(list); // 实际返回内容
        }
    

分析一下，如果项目中所有接口都按照这样的写法会产生什么样的问题？

每次返回都要使用 `ResponseEntity.status(HttpStatus.BAD_REQUEST)` 的形式定义状态码。HttpStatus
定义了 70 个状态码，有许多不常用的，特别是在大项目中，多人协作的情况下容易引起状态码使用混乱。

每次返回都要用 `contentType(MediaType.APPLICATION_JSON)` 的格式指定返回数据的格式，有两个问题：

  1. 频繁书写相同的代码，浪费时间在无意义的事情上
  2. 如果项目需要更换返回类型，就要挨个修改代码，可扩展性比较差

#### **期望结果**

因为 `return ResponseEntity` 比较常用，我们更希望用简短的类似 `Response.ok(list);` 的语句代替下面的形式：

    
    
    return ResponseEntity.ok()
                                        .contentType(MediaType.APPLICATION_JSON)
                             .body(list);
    

为了达到期望的效果，需要了解 Spring 关于 HTTP 请求处理的相关知识，具体到源码就是
ResponseEntity、HttpStatus、MediaType 类的功能和使用场景。

### 知识准备

#### **ResponseEntity**

    
    
            @GetMapping
        public ResponseEntity getAll(){
            List<UserInfo> list = userService.getAll();
            return ResponseEntity.ok() // 状态码
                    .contentType(MediaType.APPLICATION_JSON) //返回内容类型
                    .body(list); // 实际返回内容
        }
    

在这段代码中返回值是 ResponseEntity， ResponseEntity 是 Spring 中标准的 RESTful 接口返回格式，它包含了
HTTP 应答的如下内容：

  1. HttpStatus：HTTP 状态码，表示 HTTP 响应的状态
  2. ContentType：HTTP 响应实际返回的内容的内容类型
  3. Body：HTTP 响应实际返回的内容

在浏览器中访问上面的接口，打开 web 控制台，在网络选项卡中可以看到这三部分内容，如图是火狐浏览器中看到的界面：

![ResponseEntityTest](https://images.gitbook.cn/ce3da990-724c-11ea-8f47-a57dd3f092fe)

左边是 Body 中的内容，返回了实际的数据，以 JSON 格式显示在浏览器中；右边上半部分显示了 **状态码 200** ，说明该请求返回
OK；右边下半部分 **响应头** 中显示了 Content-Type:application/json，说明返回的数据类型是 JSON
格式，浏览器将按照该格式解析 Body 中的数据。

ResponseEntity 类在 spring-web 组件中的 org.springframework.http 包下，源码如下图：

![ResponseEntity](https://images.gitbook.cn/d5d5eff0-724c-11ea-964d-61a29639fe46)

从源码中可以看到，该类内置了几个常用的状态码：200 OK、400 BadRequest、404 NotFound 等。

#### **HttpStatus**

HttpStatus 类和 ResponseEntity 类在相同的包下，是一个枚举类，每一个枚举值对应一个 HTTP 状态码，源码如图所示：

![HttpStatus](https://images.gitbook.cn/de8f84d0-724c-11ea-88af-f7ef8761fe69)

HttpStatus 类定义了 70 个 HTTP 状态码，在本项目中常用的状态码有：

  * 200 OK：请求已成功，请求对应的响应头或数据体将随此响应返回。
  * 400 Bad Request：请求语义有误导致无法解析该请求或请求参数有误无法根据请求参数得到正确的数据。
  * 401 Unauthorized：接口要求用户身份验证，但请求中没有用户身份信息，返回该请求提示用户需要进行登录或身份验证。
  * 403 Forbidden：解析出的用户身份不符合接口要求的用户身份，拒绝用户访问。
  * 500 Internal Server Error：服务器遇到了一个无法处理的错误，请求无法得到正确的结果，需要程序员处理。

#### **MediaType**

MediaType 类和 ResponseEntity 类、HttpStatus 类在相同的包下，和 HttpStatus 类同样属于枚举类，源码如下：

![MediaType](https://images.gitbook.cn/e86efc10-724c-11ea-88af-f7ef8761fe69)

在源码中可以找到 HTTP 所有返回数据格式的定义，常见的格式有：

  * application/x-www-form-urlencoded：表示返回的数据是 form 表单的形式
  * application/json：表示返回的数据是 JSON 格式
  * application/xml：表示返回的数据是 XML 格式
  * image/jpeg、image/gif、image/png：表示返回的数据是对应格式的图片
  * text/markdown：表示返回的数据是文本类型的数据，可以按照 Markdown 格式解析

本项目用的是 RESTful 风格的接口，将返回数据格式统一定义为 application/json，即 JSON 格式。

### 方案分析

#### **流程设计**

该方案的流程图如下：

![流程图](https://images.gitbook.cn/f20b9df0-724c-11ea-a207-7f4534b95cc3)

#### **总体方案**

根据流程分析结果，做出如下设计方案：

  1. 自定义 **Response** 类，返回 **ResponseEntity** 类型的数据，包含定义好的 **HttpStatus** 和 **ContentType** ，用于 **Controller** 层的统一应答。
  2. 自定义 **ResultData** 类，包含业务状态、业务数据、失败描述，用于 **Service** 层的统一应答。

### 代码实现

#### **自定义 Response**

新建 Response 类，用于 Controller 层统一的应答格式，代码如下：

    
    
    public class Response {
        /**
         * 返回OK
         * @param object
         * @return
         */
        public static ResponseEntity ok(Object object){
            return ResponseEntity.ok()
                    .contentType(MediaType.APPLICATION_JSON).body(object);
        }
    
        /**
         * 返回 OK
         * @return
         */
        public static ResponseEntity ok(){
            ResultData response = ResultData.success();
            return ResponseEntity.ok()
                    .contentType(MediaType.APPLICATION_JSON).body(response);
        }
    
        /**
         * 异常请求
         * @return
         */
        public static ResponseEntity badRequest(){
            ResultData response = ResultData.error("请求参数异常");
            return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                    .contentType(MediaType.APPLICATION_JSON).body(response);
        }
    }
    

自定义的 Response 类，将每个常用的返回码都对应一个方法，在使用时就可以通过
`Response.ok(list);`、`Response.badRequest();` 等方式简单快速地给出统一的应答了。

#### **自定义 ResultData**

接口文档里面对应答格式做了统一的规定，格式如下：

    
    
    {
        "data": {},
        "success": 0,
          "message":""
    }
    

  * data：具体的业务返回数据
  * success：业务执行状态：0 成功；1 失败
  * message：错误信息

相应的，需要在业务逻辑层中返回指定格式的数据，用于表示业务进行状态及返回的结果，比较好的解决办法就是为此格式定义一个类。新建 ResultData
类，代码如下：

    
    
    public class ResultData {
    
        /**
         * 业务状态 0：成功  1：失败
         */
        private int success ;
        /**
         * 返回数据
         */
        private Object data ;
        /**
         * 文字描述，一般放业务处理失败时返回的错误信息
         */
        private String message ;
    
        public final static int SUCCESS_CODE_SUCCESS = 0;
        public final static int SUCCESS_CODE_FAILED = 1;
    
          /**
           * 业务处理成功，不需要返回结果
           */
          public static ResultData success() {
            ResultData resultData = new ResultData();
            resultData.setSuccess(SUCCESS_CODE_SUCCESS);
            return resultData;
        }
    
          /**
           * 业务处理成功，有返回结果
           */
        public static ResultData success(Object data) {
            ResultData resultData = new ResultData();
            resultData.setSuccess(SUCCESS_CODE_SUCCESS);
            resultData.setData(data);
            return resultData;
        }
    
          /**
           * 业务处理失败，返回失败描述
           */
        public static ResultData error(String message) {
            ResultData resultData = new ResultData();
            resultData.setSuccess(SUCCESS_CODE_FAILED);
            resultData.setMessage(message);
            return resultData;
        }
    
    }
    

在 ResultData 中定义了三个静态方法，分别对应三种不同的业务状态：

  * success()：业务处理成功，无返回结果
  * success(Object data)：业务处理成功，有返回结果
  * error(String message)：业务处理失败，返回失败描述

### 效果测试

在 UserServer 中添加一个方法 `getByEmail(String email)`，返回值为 **ResultData**
，根据邮箱查询用户。查询成功，返回用户信息；否则，返回错误提示：该邮箱未注册。代码如下：

    
    
    public ResultData getByEmail(String email){
            UserInfo userInfo = new UserInfo();
            userInfo.setEmail(email);
            List<UserInfo> list = userMapper.baseSelectByCondition(userInfo);
            if(list != null && list.size() > 0){
                return ResultData.success(list.get(0));
            }
            return ResultData.error("该邮箱未注册");
        }
    

在 UserController 中添加一个方法 `getByEmail(String email)`，返回值为 **ResponseEntity**
，根据邮箱查询用户。

  * 先判断请求参数中邮箱是否为空，如果为空，返回 400 BadRequest；
  * 如果邮箱不为空，调用 Service 层的 `getByEmail(String email)` 方法查询数据，并将查询结果组装成接口文档要求的格式进行返回。

    
    
    @GetMapping("/getByEmail")
    public ResponseEntity getByEmail(String email){
        if(StringUtils.isEmpty(email)){
            return Response.badRequest();
        }
        ResultData resultData = userService.getByEmail(email);
        return Response.ok(resultData);
    }
    

启动项目，使用 PostMan 进行测试。

使用带有 email 参数的请求访问接口，请求不存在的邮箱时返回 HttpStatus 200 OK，但是提示“该邮箱未注册”，如图：

![返回200error](https://images.gitbook.cn/18d71ae0-724d-11ea-b8cb-b108c7f0b146)

请求存在的邮箱时，返回 HttpStatus 200 OK，并成功返回了查询到的用户数据，如图：

![返回200success](https://images.gitbook.cn/2c908b20-724d-11ea-a207-7f4534b95cc3)

当请求中没有 email 参数时，返回 HttpStatus 400 BadRequest，提示”请求参数异常“，如图：

![返回400](https://images.gitbook.cn/2097f7e0-724d-11ea-af73-378397689c85)

至此，说明项目中已经可以使用统一的 HTTP
应答来处理请求，可以保证所有接口应答格式的统一，能大大减少接口应答的工作量和避免接口应答的不统一而造成的开发问题。

### 源码地址

本篇内容完整的源码在 GitHub 上，地址如下：

> <https://github.com/tianlanlandelan/KellerNotes/tree/master/9.统一应答/server>

### 小结

本篇带领大家完成了对项目统一应答的处理，使 **Controller** 层 和 **Service**
层返回的数据格式统一化，保证了接口的统一性。在多人协作及前后端分离项目中意义重大，保证数据格式统一的基础上极大程度上简化了开发及沟通工作。

