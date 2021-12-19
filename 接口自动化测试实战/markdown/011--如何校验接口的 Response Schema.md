上一节课程中我们学习了接口测试 Response Body 的校验，本次课程将带领大家学习如何校验接口response
schema。为了完成本次课程目标，我分了两个 Task

  * 使用 REST Assrued 校验 Response Schema
  * Schema 文件定义详解

## 使用 REST Assrued 校验 Response Schema

这里仍然使用上一节课中的 getResume 接口，以下是 getResume 接口 Response
部分内容，接口地址是：http://localhost:9090/api/getResume

    
    
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
       .......   // 这里没有贴出getResume接口的所有response，因为上次课程已mock好了这个接口，此次课程直接使用即可，这里贴出来主要作用是与schema文件进行match
    

假设 getResume 接口 name 和 age 字段是必填字段，name 是 string 类型，age 是 number 类型，且 age 范围是
20-50。

针对这样的接口如果要对 Response Schema 进行校验，我们就需要 check reponse中确实包含 name 和 age field，且
name 的类型是 string，age 的类型是number，且 age 的值是 20-50 之间。为了校验 Response
Schema，首先需要定义Schema 文件。

以下是编写的 Schema 文件，properties 中定义需要校验的 field 类型，默认值，取值范围等内容，required 中存放必填
field，Schema 后面带 Schema 版本，Schema 文件编写稍后会详细解释：

    
    
    {
      "type": "object",
      "$schema": "http://json-schema.org/draft-07/schema",
      "title": "test",
      "description": "test",
      "properties": {
        "name": {
          "type": "string"
        },
        "age": {
          "type": "number",
          "minimum": 20,
          "maximum": 50
        },
        "test":{
          "type": "string"
        }
      },
      "required": [
        "name",
        "age",
        "test"
      ]
    }
    

定义好 Schema 文件后，如何编写代码实现 Schema 的校验呢？实际 REST Assrued 提供了 Schema
校验功能。为了实现该场景，首选需要在`pom.xml`文件中引入该 jar包。

    
    
      <dependency>
          <groupId>io.rest-assured</groupId>
          <artifactId>json-schema-validator</artifactId>
          <version>3.0.5</version>
        </dependency>
    

下载依赖后，在 ResumeClient 中编写校验 Schema 的代码，REST Assrued 的`body()`接受
`matchesJsonSchema()` 和 `matchesJsonShemaInClasspath()`。
`matchesJsonSchema()`接受 string、File、InputStream
等类型作为参数，`matchesJsonSchemaInClasspath()`接受 string 类型的 filePath 作为参数，以下案例采用
`matchesJsonSchemaInClasspath()`，大家也可以尝试使用另外一种方式完成 Schema 校验。

    
    
       def getResumeSchemaValidate(filePath) {
            given().baseUri("http://localhost:9090")
                    .when()
                    .get("/api/getResume")
                    .then().assertThat()
                    .body(matchesJsonSchemaInClasspath((String)filePath))    //在调用接口时通过matchsJsonSchemaInClassPath校验接口的response schema
        }
    

schema file 存放在 `resources/com/github/schema`
目录下，故`filePath=“com/github/schema/getResumeSchema.json”`，以下是 Case 代码。

    
    
       def "call the api"() {
            given: "no given"
            when: "call the get resume api"
             resumeClient.getResumeSchemaValidate(filePath)    //编写case调用上面的getResumeSchemaValidate方法
            then: "no then"
            where:
            filePath|placeHolder
            "com/github/schema/getResumeSchema.json"|""        //这里请输入自己存放的shema文件地址
        }
    

执行该 case 能运行成功则说明 Schema 校验流程正确，此时我们尝试修改 Schema 文件，properties 中新增一个接口 Response
中不存在的 filed ”test“，并添加到required 中，再次运行，Case 执行失败，可以看到会打印详细的错误信息。

![](media/15731159117013/callSchemaValidate.gif)

上面的 Schema 中只对 name，age 做了定义说明，实际项目中定义的 schema 可能非常复杂的，以下是对 schema
的一些规则进行说明，便于大家自己能灵活定义所测接口的JSON Schema。

## Schema 文件定义详解

  * properties 中 type 总共包含六种类型（integer、string、number、object、array、boolean），number 和 integer 的区别是 integer 只可以匹配整数类型
  * type 为 number 的数据，可以通过 maxinum，minimum 指定数据取值范围
  * type 为 string 的数据，可以通过 maxLength，minLength 指定字符串长度范围，通过pattern正则表达式定义该field值
  * type为array的数据，可以通过 minItems,maxItems 指定数组长度，uniqueItems是个布尔值，true表示数组值之间不可重复
  * type为object的数据，可以通过 maxProperties，minProperties 指定该object 包含的属性个数范围，required 中指定 object 的哪些属性是必须的，properties 里面指定每个属性的类型，长度范围等 根据上面的规则，我们重新定义 getResume 这个接口的 Response Schema，假设接口的 response 中每个字段有如下规则
  * name feild：type 是 string，假设长度范围是1-100的字符，必须是大写字母
  * age feild：type 是 number，假设长度范围是20-50
  * birthPlace field：type 是 object，包含四个属性country，city，state，street都是必填项
  * contacts filed：type 是 array，假设数组长度范围是 1-3，每个 item 又是个object，object 中的 phone，address 都是必填字段 按照上面的假设，定义的 schema 如下，大家在定义 schema 的时候可以参考到getResume 的 Response Body 一起看

    
    
    {
      "type": "object",
      "properties": {
        "name": {
          "type": "string",
          "minLength": 1,
          "maxLength": 3,
          "pattern": "[A-Z]"    //定义name字段的值长度是1-3个字符，只能是A-Z的字母组合
        },
        "age": {
          "type": "number",    //定义age字段值是20-50   
          "minimum": 20,
          "maximum": 50
        },
        "birthPlace": {        //定义birthPlace中包含四个字段，country，city，state，street
          "type": "object",
          "required": [
            "country",
            "city",
            "state",
            "street"
          ],
          "properties": {
            "country": {
              "type": "string"
            },
            "city": {
              "type": "string"
            },
            "state": {
              "type": "string"
            },
            "street": {
              "type": "string"
            }
          }
        },
        "contacts": {         //定义contacts是一个数组，数组中每个值是一个object对象，每个object包含phone，address两个字段，
          "type": "array",
          "minItems": 1,
          "maxItems": 3,
          "uniqueItems": true,
          "items": {
            "type": "object",
            "required": [
              "phone",
              "address"
            ],
            "properties": {
              "phone": {
                "type": "string",
                "minLength": 8,
                "maxLength": 11,
                "pattern": "[0-9]"        //定义phone长度是8-11位，只能是0-9数字组合
              },
              "address": {
                "type": "string"
              }
            }
          }
        }
      },
      "required": [
        "name",
        "age",
        "birthPlace",
        "contacts"
      ]
    }
    

使用新的 Schema 文件，调用 getResume 接口再次进行校验，可以看到运行成功如下图所示

![](media/15731159117013/callSchemaValidate2.gif)

以上讲解的是接口的response
body是个json对象，实际项目中有些接口的response中某个字段值才是需要关注的内容，且字段值是一个string类型的json字符串，对于这类接口如何做shema的校验呢？
还是同样的配方，先用 WireMock 模拟一个这样的接口。mapping 文件和 Response file 文件内容如下

    
    
    {
      "request": {
        "method": "GET",
        "urlPath": "/api/getResume2"
      },
      "response": {
        "status": 200,
        "bodyFileName": "sexCourse/resume2.json",
        "headers": {
          "Content-Type": "application/json; charset=UTF-8"
        }
      }
    }
    

`resume2.json`文件内容如下图所示，payLoad 中的内容实际就是 getResume 中的Response Body。

    
    
    {
      "item_count": 100,
      "properties": {"content-type":"application/json"},
      "payLoad":"{\"name\":\"TOM\",\"age\":30,\"birthPlace\":{\"country\":\"China\",\"city\":\"meijing\",\"state\":\"chaoyang state\",\"street\":\"wangfujing street\"},\"contacts\":[{\"phone\":\"12345678901\",\"address\":\"chengdu-tianfu-state\"},{\"phone\":\"11223344556\",\"address\":\"chengdu-jingjiang-state\"}],\"working\":{\"workingProject\":[{\"projectName\":\"aProject\",\"jobTitle\":\"DEV\",\"responsibility\":\"devTest\"},{\"projectName\":\"bProject\",\"jobTitle\":\"QA\",\"responsibility\":\"qaTest\"},{\"projectName\":\"cProject\",\"jobTitle\":\"TL\",\"responsibility\":\"tlTest\"}]},\"skills\":{\"language\":\"pass CET6\",\"tech\":[{\"language\":\"Java\",\"level\":\"good\"},{\"language\":\"C++\",\"level\":\"poor\"}]},\"comment\":\"other info\"}"
    }
    

用 Postman 调用`http://localhost:9090/getResume2`，返回接口如下图所示，可以看到需要校验的内容是payload
的值，而 payload 的值是一个 JSON 字符串。

![](media/15731159117013/postman.png)

对于该接口，我们真正关注的内容在payload里面，所以我们需要调用接口获取到payload的值，然后采用 hamcrest 中的 assertThat
方法进行 Schema 验证。

以下是调用 getResume2 接口获取 payLoad 值的代码：

    
    
        def getResume2() {
           def payLoad=  given().baseUri("http://localhost:9090")
                    .when()
                    .get("/api/getResume2")
                    .then().assertThat().statusCode(200)
                    .extract()
                    .response()
                    .path("payLoad")            //返回接口response中payLoad的值
            payLoad
        }
    

以下是调用 hamcrest 的 assertThat 方法对 Schema 进行校验，大家可以尝试修改schema file 中 name 的
type为number，再次运行如果报错，说明整个 schema 校验流程正确。

    
    
       def "validate schema of getResume2 api"() {
            given: "no given"
            when: "call the get resume2 api"
            def payLoad= resumeClient.getResume2()        //获取接口中payLoad的值
            then: "check the schema"
            assertThat(payLoad, matchesJsonSchemaInClasspath(filePath))    //assertThat中使用matchesJsonSchemaInClasspath校验payLoad的值是否与getResumeSchema2中定义的schema一致
            where:
            filePath|placeHolder
            "com/github/schema/getResumeSchema2.json"|""
        }
    

运行上述 Case 如下图所示，能运行成功，修改 getResumeSchema2 中任意一个 field 名字，再次运行，运行失败说明整个校验生效。

![](media/15731159117013/callSchemaValidate3.gif)

另外如果 IDE 工具不能自动 import 依赖的包，请手动写上 import 包的代码

    
    
    import static org.hamcrest.MatcherAssert.assertThat
    import static io.restassured.module.jsv.JsonSchemaValidator.matchesJsonSchemaInClasspath
    

至此 JSON 格式的 Response Schema
校验的讲解就完成了，在实际项目中如果是项目自己调用的接口，我们关注的通常是整个业务场景是否正确，而不会单独去验证系统使用的所有接口的 Response
Schema。

而对于提供给第三方的接口，因为接口的 Schema 是双方提前约定好的，第三方也会按照这个 Schema 来获取接口 Response 中的值，所以
Scehma 的校验是必须的。

下章节将给大家讲解如何编写 XML 格式接口测试代码。

