上一节课讲解了接口的 Request Body 管理，本节课将带领大家学习如何解析并校验接口的Response。为了完成课程目标，分了两个 Task

  * Response Body 校验 *header/cookie 校验

## Response Body 校验

为了练习 Resposne 的解析，首先需要通过 WireMock 模拟一个具备复杂 Reponse 的接口。以下是模拟一个 GET
请求的接口，、esponse 的 content-type 是 JSON。 mapping 文件内容如下

    
    
    {
      "request": {
        "method": "GET",
        "urlPath": "/api/getResume"
      },
      "response": {
        "status": 200,
        "bodyFileName": "resume.json",
        "headers": {
          "Content-Type": "application/json; charset=UTF-8"
        }
      }
    }
    

Request Body 文件

    
    
    {
      "name": "TOM",
      "age": 30,
      "birthPlace": {
        "country": "China",
        "city": "meijing",
        "state": "chaoyang state",
        "street": "wangfujing street"
      },
      "contacts": [
        {
          "phone": "12345678901",
          "address": "chengdu-tianfu-state"
        },
        {
          "phone": "11223344556",
          "address": "chengdu-jingjiang-state"
        }
      ],
      "working": {
        "workingProject": [
          {
            "projectName": "aProject",
            "jobTitle": "DEV",
            "responsibility": "devTest"
          },
          {
            "projectName": "bProject",
            "jobTitle": "QA",
            "responsibility": "qaTest"
          },
          {
            "projectName": "cProject",
            "jobTitle": "TL",
            "responsibility": "tlTest"
          }
        ]
      },
      "skills": {
        "language": "pass CET6",
        "tech": [
          {
            "programming language": "Java",
            "level": "good"
          },
          {
            "programming language": "C++",
            "level": "poor"
          }
        ]
      },
      "comment": "other info"
    }
    

REST Assured 提供了多种对 Response Body 校验的方式，这里会重点讲解“获取Response Body 的 string
值，接着通过 JsonSlurp 或者 XmlSlurp 转换为数据集，然后进行校验的方式，其他的 Body 校验方式只做简单的介绍。

因为上述接口 Response 的 Content-Type 是 JSON ，所以采用 JsonSlurp 进行处理。

假设针对上面的 GET 请求接口需要测试这样的场景：获取候选人简历中出生地，如果出生地一致则打印出来，如果出生地不一致则打印“No person”。

以下是调用接口并校验返回内容的代码，为了完成这个 Case，创建三个 class。

  * ResumeClient：负责接口调用
  * ResumeService：接口 Response 的解析以及校验
  * Case：编写的测试脚本

    
    
    // ResumeClient代码
    class ResumeClient {
       def getResumeDetails(){
            def res = given().baseUri("http://localhost:9090")
                    .when()
                    .get("/api/getResume")
                    .then().assertThat().statusCode(200)
                    .extract().response().getBody().asString()       //获取接口的response body
            res
        }
    }
    // ResumeService代码
     class ResumeService {
        JsonSlurper jsonSlurper
    
        ResumeService() {
            jsonSlurper = new JsonSlurper()
        }
    
        def getPersonByCountry(String res, country) {
            def resumeDetails = jsonSlurper.parseText(res)          //将String类型的json字符串转换为数据对象，转换为数据对象后才能轻松获取对应的属性值
            resumeDetails.birthPlace.country == country ? resumeDetails.name: "no person"
            //获取接口response body中的contry信息，如果与传入的country一致则返回该值，否则返回“no person”
        }
     }
     //case代码,where部分采用data-driven方式，传入两个测试数据“China”和“USA”
     class Case extends Specification {
        ResumeClient resumeClient
        ResumeService resumeService
    
        def setup() {
            resumeClient = new ResumeClient()
            resumeService = new ResumeService()
        }
    
        def "get person from different country"() {
            given: "no given"
            when: "call the get resume api"
            def res = resumeClient.getResumeDetails()
            then: "println  out the person name from different country"
            println resumeService.getPersonByCountry(res, country)
            where:
            country | placeHolder
            "China" | ""
            "USA" | ""
        }
     }
    

运行该 Case 如下图所示，可以看到第一个打印出了“TOM”，第二个打印出了“No Person"，因为 Mock 接口的时候，Response Body
中 country 的值是“China”

![](https://images.gitbook.cn/15749336902150)

上面只是获取 Response 中的 country 信息进行简单判断，如果需要获取其他值或查找符合某些条件的信息如何处理呢？通过 Debug 可以发现
jsonslurp 解析后返回一个 Map 对象，如下图所示.

以 contacts 为例，contacts 本身是个 map 对象，contacts 是 key，具体的联系信息是 value，而 value
本身又是一个 ArrayList，每个数组的 value 又是一个 map 对象

![](https://images.gitbook.cn/15749336902162)

如果要获取 contact 中的 phone 信息，如何处理呢？下面代码实现了获取 contact 中第一个 phone，大家可以尝试编写如何获取所有
phone 信息的代码

    
    
    def getContactPhone(String res) {
            def resumeDetails = jsonSlurper.parseText(res)
            resumeDetails.contacts.size() > 0 ? resumeDetails.contacts[0].phone : "no contact"     
            //这里做了空保护，先判断contacts.size()>0再获取phone信息，在实际项目中为了保证接口测试在流水线上稳定运行，在平时编写脚本时一定要注意进行空保护，否则某些情况下可能就报空指针异常了
        }
    

同理我们还可以利用前面”Groovy“课程中的 find，each 方法对 Reponse 中的内容进行筛选或者遍历。

例如查找候选人简历中是否具备某项编程语言能力，ResumeService代码如下，可以自己完成 Case 部分的代码

    
    
     void printIfPersonWithSpecialSkill(String res, language) {
            def resumeDetails = jsonSlurper.parseText(res)
            if (resumeDetails.skills.tech.size() > 0) {
                def programmingSkill = resumeDetails.skills.tech.find { it -> it.language == language }     //使用了find方法
                println "--programmingSkill:${programmingSkill.language}--level:${programmingSkill.level}"    //使用了groovy中“”中可以带参数的特性
            }
        }
    

通过遍历打印简历中所有的项目经验信息，同上自己完成 Case 部分的代码编写

    
    
          void printWorkingDetails(String res) {
            def resumeDetails = jsonSlurper.parseText(res)
            if (resumeDetails.working.workingProject.size() > 0) {
                resumeDetails.working.workingProject.each { it ->
                        println "--projectName:${it.projectName}--jobTitle:${it.jobTitle}--responseibility:${it.responsibility}" }
                        // 使用each遍历，且使用了groovy中“”中可以带参数的方式打印获取到的内容
            }
        }
    

编写 Case 调用 ResumeService 中的这些方法，如下图所示，打印出来的信息与mock的接口中一致，说明正确获取了 Response Body
中的信息。

![](https://images.gitbook.cn/15749336902191)

至此通过 jsonSlurp 解析接口 Response Body 的内容就讲解完了。

此种方式过程是：接口调用时获取 Body 的 string 值，接着通过 jsonSlurp 将 string 值转换为数据对象，然后通过 Groovy
提供的 find，each 等方法获取数据对象中的任意属性值。

除了上述方式外，REST Assured 还提供了其他校验 Response Body 的方法，例如Body 中直接进行 check 或者
`response.path` 方式。

如果对接口的 Response 只做单一简单的校验，可以采用该种方式，如果是复杂校验建议采用 jsonSlurp 转换方式。

如下是使用 REST Assured 提供的方法校验接口Response Body的内容

    
    
        void getResumeDetails2() {
            given().baseUri("http://localhost:9090")
                    .when()
                    .get("/api/getResume")
                    .then().assertThat().statusCode(200)
                    .body("name", equalTo("TOM"))    //校验response中名称是否是TOM
        }
    
        def getResumeDetails3() {
            given().baseUri("http://localhost:9090")
                    .when()
                    .get("/api/getResume")
                    .then().assertThat().statusCode(200)
                    .extract()
                    .response().path("contacts.phone")   // 返回reponse中的所有phone信息
        }
    

为了运行上述两个方法，这里使用 junit 提供的 @Test 注解（采用 junit 的注解主要是为了简化 Case 编写，因为本节课的重点是学习
Request Body 的解析校验），代码如下：

    
    
    import org.junit.Test
    
    class JunitCase {
        @Test
        void testFunction() {
           ResumeClient resumeClient= new ResumeClient()
            resumeClient.getResumeDetails2()
          println "--phone information is:    "  + resumeClient.getResumeDetails3()
        }
    }
    

运行上述 Case 如下图所示，第一个接口调成功，如果修改 equalTo 中的内容为“TOMS”接口调用失败，说明校验生效了。

第二个接口调用打印的 `contacts.phone` 的信息与 Mock 时定义的信息一致，说明获取信息正确：

![](https://images.gitbook.cn/15749336902210)

## header、cookie校验

上面讲解了 Response Body 的校验，接下来我们学习如何对 header 或者 cookie 进行判断处理。

以登录为例，有些应用会把登录生成的 token 放到response的header中，在调用其他接口前需要先调用登录接口，获取 Response
header 中的 Token，然后传入其他接口进行使用。

故学习如何获取 header 或者 cookie 信息是非常有必要的。

如下是获取header中的content-type,可以自己尝试获取header的其他信息或者mock一个带cookie的接口获取cookie信息

    
    
        def getResumeDetailHeader() {
            given().baseUri("http://localhost:9090")
                    .when()
                    .get("/api/getResume")
                    .then().assertThat().statusCode(200)
                    .extract()
                    .response().getHeader("content-type")   //获取header中content-type值
        }
    

运行上述 Case 如下图所示，可以看到确实打印了 header 值。

![](https://images.gitbook.cn/15749336902224)

至此关于接口的 Response Body 的解析校验就讲解完成了。如果在获取某些信息时不清楚可以查看[ REST
Assured使用手册](https://github.com/rest-assured/rest-assured/wiki/Usage) 。

在实际项目中，很多时候为了控制自动化测试维护成本，我们可能只对接口 Response 中部分核心内容进行校验，那么如何在最小维护成本下保证接口
Response 的其他内容也正确呢？例如开放给第三方的接口，出现了某个必填 field 不存在的 Bug 或者非空 field 返回空值的 Bug。

下节课将带领大家学习接口 Response Schema 的校验。

