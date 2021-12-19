上一节中我们学习了如何使用 REST Assured 调用一个简单的 GET 请求接口，本节课将带领大家学习如何使用 REST Assured
调用各种类型接口并生成html格式的测试报告。

为了完成今天的课程目标，我分了 4 个 Task.

  * Task1：使用 data-driven 编写测试案例
  * Task2：调用包含查询参数接口
  * Task3: 调用 POST 请求且设置了 BasicAuth 的接口
  * Task4: 获取测试报告

## 使用data-driven编写测试案例

在“ WireMock 模拟接口”课程中，我们学习了模拟一个 GET 请求接口，Url
是`http://localhost:9090/api/getBook/xxx`，xxx 为任何 a-z
的字母组合，接下来我们将以这个接口为例编写调用测试用例。

备注：如果没有学习前面课程的同学，这里是 WireMock 的 Repo 地址（<https://github.com/tlqiao/wiremock-
demo.git>), 可以 clone 代码，然后启动 WireMock 服务，接口测试中需要用到的接口都在这个 Repo 上。

针对上面的接口假如测试场景是`/getBook`后面输入不同的书名都能成功返回正确 Response。针对这种“测试场景相同、传入参数不同”的 Case
我们可以利用 spock 提供的 data-driven 来实现， 创建名为 SecondDemo 的 Class，写入如下代码：

    
    
    package github.com.fourCourse
    
    import spock.lang.Specification
    import static io.restassured.RestAssured.given
    
    class SecondDemo extends Specification {
    
        def "should get user details by user name successfully"() {
            given: "no given"
            when: "call get user by name api"
            given().baseUri("http://localhost:9090/")
                    .when()
                    .pathParam("bookName",bookName)
                    .get("api/getBook/{bookName}")
                    .then()
                    .assertThat().statusCode(200)
            then: "no then"
            where:                            //固定写法，where：后面跟测试用例需要的测试数据
            bookName|placeHolder             //固定写法，多个参数之间用“|”隔开，且至少要有一个“|”，所以如果只有一个输入参数“|”后面可以写个placeHolder
            "TOM"|""
            "Dave"|""
        }
    }
    

编写好代码后，命令行中运行该 Class，如下图所示

![](https://images.gitbook.cn/15749142766590)

运行该 Case 你会发现 Assert 失败，报 404 错误，什么原因呢？

看代码感觉没有问题呀，是 url 错误呢？还是通过 pathParam 传递参数的方式不正确呢？

针对接口调用的调试，REST Assured 提供了很简单的调试方式，`given()`后面加入`.log().all()`即可打印 Request
的详细信息，在`then()`后面加入`.log().all()`即可打印 Response
的详细信息。接下来我们就利用该方法调试一下，以下是添加`log().all()`后的代码：

    
    
     package github.com.secondCourse
    
    import spock.lang.Specification
    import static io.restassured.RestAssured.given
    
    class SecondDemo extends Specification {
    
      def "should get user details by user name successfully"() {
            given: "no given"
            when: "call get user by name api"
            given().baseUri("http://localhost:9090/").log().all()      //添加.log().all()后会打印详细的reqeust信息
                    .when()
                    .pathParam("bookName",bookName)
                    .get("api/getBook/{bookName}")
                    .then().log().all()                 //会添加详细的response信息
                    .assertThat().statusCode(200)
            then: "no then"
            where:           
            bookName|placeHolder  
            "TOM"|""
            "Dave"|""
        }
    }
    

再次在 IntelliJ 中运行上述代码，可以看到调用接口的 Url 以及详细的 Response，发现 Url 的拼接没有错误，但 Response
是`Request Not Found`，如下图所示

![](https://images.gitbook.cn/15749142766606)

反查一下 WireMock
的服务，发现我们mock的api定义的参数是`([a-z]*)`，上面的参数`TOM,Dave`包含大写字母，哦，原来是大小写的原因，修改 bookName
都为小写字母再次运行 Case 。 Case 运行成功如下图所示

![](https://images.gitbook.cn/15749142766619)

上面讲解了 data-driven 方式编写 Case 和接口调用失败后如何进行 Debug，接下来将讲解如何调用带查询参数的接口。

## 调用包含查询参数接口

在“ WireMock
模拟接口”课程中，我们已经模拟过带查询参数的接口，例如：/api/getBookByPathPatter/test?后面可以跟任意的查询参数，对于queryParam的接口，
REST Assured 调用方式如下

    
    
     class ThirdDemo extends Specification{
        def "should call get user by name and age api successfully"() {
            given: "no given"
            when: "call mock api api"
            given().baseUri("http://localhost:9090")
                   .queryParam("name","sanguo")         //设置接口的查询参数
                   .queryParam("price",18)              //设置接口的查询参数
                    .when()
                    .get("api/getBookByPathPatter/test")
                    .then()
                    .assertThat().statusCode(200)
            then: "no then"
        }
    }
    

运行该案例，运行成功，查看发送的请求，接口地址正确，如下图所示

![](https://images.gitbook.cn/15749142766634)

## 调用post请求且设置了BasicAuth的接口

前面学习了如何调用“带查询参数的get请求接口”，接下来将学习如何调用post请求接口，且接口设置了basicAuth和header。先利用
WireMock 模拟一个带header，cookie且是basic认证的接口，mapping文件如下,可以看到header中定义了content-
type，cookies中定义了session的取值，另外还配置了basicAuth的用户名和密码。

    
    
    {
      "request": {
        "method": "POST",
        "urlPath": "/api/addUser",
        "headers": {
          "Content-Type" :{"equalTo" : "application/json; charset=UTF-8"}     //设置conent-type
        },
        "cookies": {
          "session": {
            "matches": ".*12345.*"    //设置cookie，cookie中包含session属性，且session的值需要包含12345
          }
        },
        "basicAuth": {
          "username": "root",
          "password": "root123"     //设置调用接口的用户名和密码
        },
        "bodyPatterns": [
          {
            "matchesJsonPath": "$.name",
            "matchesJsonPath": "$..mainName",
            "matchesJsonPath": "$..alias",
            "matchesJsonPath": "$.age"
          }
        ]
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "application/json"
        },
        "body": "add user successfully"
      }
    }
    

接口测试代码中创建名称为 FourDemo 的 Class 并写入如下代码

    
    
     class FourDemo extends Specification {
    
        def "should add user successfully"() {
            given: "no given"
            def body = "{\"name\": {\"mainName\": \"zhangshang\",\"alias\": \"zhangshangalias\"},\"age\":123}"
            when: "call get user by name api"
            given().log().all()
                    .baseUri("http://localhost:9090")
                    .auth().preemptive().basic("root", "root123")  //接口调用时，对basicAuth的的设置
                    .header("Content-Type", "application/json")   //mock的接口header中定义了Content-Type，所以接口调用是设置header值
                    .cookie("session", "123456")                 //mock的接口中定义了名称为“session”的cookie，所以接口调用时设置该cookie
                    .body(body)                                 //这里body是采用传入Body Sring方式，后续课程会详细讲解如何管理接口测试的request body
                    .when()
                    .post("/api/addUser")
                    .then()
                    .log().all()
                    .assertThat().statusCode(200)
            then: "no then"
        }
    }
    

编写好代码后执行上述 Case，可以看到接口调用成功，发起的 POST 请求中确实带了header、cooki、header 的 Content-Type
是`application/json`。

![](https://images.gitbook.cn/15749142766652)

以上就是如何使用 REST Assured 调用 GET 和 POST 请求接口的所有知识点，感觉怎么样？是不是很简单。

当你熟悉后，编写一个接口调用的 Case 只需几分钟。接下来我们将学习如何获取测试报告。

## 获取测试报告

前面我们已配置spock-report依赖，所以获取测试报告非常简单。在项目的build目录下有个spock-
report的目录，打开index.html文件，即可看到所有case运行情况。

下图是测试报告目录，可以看到有多个html文件，实际每执行一个测试 Class 就会为该Class 生成一个测试 HTML
文件。`index.html`是汇总所有的测试报告。

实际项目中打开`index.html`即可。

![](https://images.gitbook.cn/15749142766665)

下图是两个 Case 执行结果汇总图即`index.html`文件内容

![](https://images.gitbook.cn/15749142766678)

点击失败的 Case ，会显示具体的错误原因，实际项目中当测试用例非常多的情况下，通过测试报告可以快速定位失败的 Case 并查看失败原因。

![](https://images.gitbook.cn/15749142766693)

至此今天的课程就结束啦，今天我们学习了如何通过 REST Assured 调用接口并查看测试报告。

由于后续课程中会使用 Groovy语言，下节课会带领大家学习 Groovy 相关知识，保证即便你无任何编程基础也能完成课程的所有实践。

