上一节课中给大家介绍了如何搭建 WireMock 服务并模拟了一个简单的 Get
请求接口。为了后续在学习接口测试过程中能模拟各种类型的接口，本次课程将带领大家学习如何通过 WireMock
模拟更复杂的接口，为了完成本次课程目标，我将课程内容拆分成了两个小的 Task：

  * Task1：模拟 JSON 格式的接口
  * Task2：模拟 XML 格式接口

## 模拟 JSON 格式接口

模拟一个接口，总的来说就是配置 mapping 文件，mapping 文件中又分为 Request 的配置和 Response 的配置，我们先从
Request 配置进行讲解。Request 配置我们主要介绍method，urlpath， QueryParameters ，BodyPatterns
的配置。method作用是配置接口的请求方法，值包含 GET、POST、PUT、DELETE 等。

urlPath 作用是配置接口的路径参数，这里可以有两种方式进行控制，一种是 urlPattern 和urlPathPattern。

如下`mapping`文件配置了一个 GET 请求的接口，路径参数使用 urlPattern 方式：

    
    
    {
      "request": {
        "method" : "GET",
        "urlPattern": "/api/getBook/([a-z]*)"
      },
      "response": {
        "status": 200,
        "body": "get book with url pattern successfully"
      }
    }
    

从mapping文件看，我们模拟了一个 GET 请求的接口，接口的地址是`/api/getBook/([a-z]*
)`,这里`[a-z]*`是正则表达式，表示可以输入多个`a-z`的任意字符。配置好 mapping 文件后，启动 WireMock
服务（如果服务已经启动，修改 mapping 文件后需要重启 WireMock 服务，新配置的 mapping 文件才会生效）。

下图是 Postman 调用刚刚模拟的接口视频

![](https://images.gitbook.cn/15748289856130)

可以看到输入地址`http://localhost:9090/api/getBook/xxxx`能成功调用该接口，例如getBook 后面输入
test、abc、edf 都能成功调用该接口，说明与 mapping 中定义的规则一致。

如果此时想模拟一个带查询参数的接口，使用上面的mapping文件还会工作么？例如输入`http://localhost:9090/api/getBook/edf?name=qiaotl`，从上面的小视频发现当接口的路径中带了查询参数后调用失败，失败的
Response 中返回`URL does not match`的错误信息。

那如何模拟带有查询参数的接口呢？对于路径中有查询参数的接口，建议使用 urlPathPattern 配置 mapping 文件，mapping
文件内容如下，此 mappping
文件定义的接口含义是：`getBookByPathPatter`可以输入`a-z`的任意字符，且还可以带任意多个查询参数。

    
    
    {
      "request": {
        "method" : "GET",
        "urlPathPattern": "/api/getBookByPathPatter/([a-z]*)" 
      },
      "response": {
        "status": 200,
        "body": "get book with url path pattern successfully"
      }
    }
    

为了验证 mapping 文件是否正确，我们再次用 Postman 进行验证，验证结果如下图所示：

![](https://images.gitbook.cn/15748289856147)

从上面验证结果可以知道，`getBookByPatter/xxx` 后面任意查询参数都能调用成功，例如输入 name 和 age
两个查询参数能成功调用接口。

如果期望模拟的接口的查询参数只能是固定的名称应该如何配置呢？例如：GET 请求接口中只能输入 name 和 Price 两个查询参数，对于这样的接口可以采用
QueryParameters 进行配置。

如下是新的 mapping 文件内容,因为配置了 QueryParameters ，所以查询参数只接收 name 和 price 两个查询参数，且 name
的值只能是 a-z 的任意字符，price 的值只能是 0-5 的任意数字组合

    
    
    {
      "request": {
        "method": "GET",
        "urlPathPattern": "/api/getBookByQueryParam/([a-z]*)",
        "QueryParameters": {
          "name": {
            "matches": "[a-z]*"
          },
          "price": {
            "matches": "[0-5]*"
          }
        }
      },
      "response": {
        "status": 200,
        "body": "get book with url path pattern and  QueryParameters  successfully"
      }
    }
    

还是相同的方式，使用 Postman 验证新 Mock 的接口。

![](https://images.gitbook.cn/15748289856165)

可以看到当 price 的值是 8 时调用接口失败，因为 mapping 文件中设置了 price 只能是 0-5 的数字。当添加 age
这个查询参数时，调用接口也失败，因为 mapping 文件中设置了只接收 name 和 price 两个查询参数。

以上就是模拟 GET 请求接口的几种方式，我们总结一下。

  * 如果模拟的接口无查询参数，那么使用 urlPattern 即可。
  * 如果模拟的接口有查询参数，但对查询参数的名字和个数无任何限制，那么使用 urlPathPattern 即可。
  * 如果模拟的接口有查询参数，且对查询参数的名称和值都有限制，那么需要在 mapping 文件中添加 QueryParameters 进行控制。

上面都是对GET请求接口的配置，POST 请求的接口如何配置对应的 mapping 文件呢？例如我们需要模拟一个 POST 请求的接口，接口的
Request Body 体如下，price 的价格小于或等于 200

    
    
    {
      "books": [{
       "name":"sanguo",
       "price":100,
       "author":"罗贯中"
      },
      {
        "name":"hongloumeng",
        "price":120,
        "author":"曹雪芹"
      }],
      "comment":"test"
    }
    

可以看到上面的 Body 体是 JSON 格式，一级属性中有 books 和 comment，books 的值是一个数组对象，数组对象里面每个值又是一个
JSON 对象。对于该类接口，我们可以使用bodyPatterns 进行模拟，mapping 文件内容如下

    
    
    {
      "request": {
        "method": "POST",
        "urlPathPattern": "/api/addBookWithBodyPatter/([a-z]*)",
        "bodyPatterns":[{
          "matchesJsonPath": "$.books",
          "matchesJsonPath": "$.comment",
          "matchesJsonPath": "$..name",
          "matchesJsonPath": "$..price",
          "matchesJsonPath": "$..author",
          "matchesJsonPath": "$..[?(@.price<200)]"
        }]
      },
      "response": {
        "status": 200,
        "body": "add book with bodyPatterns successfully"
      }
    }
    

"matchesJsonPath": `\(.books`中`\)`表示 JSON 对象根目录，`\(.books`表示根目录中存在名称为 books
的一级属性，`\)..[?(@.price<200)]`表示存在名称为 price 的属性，且该属性的值小于200.

使用 Postman 验证结果如下图所示，因为此次模拟的是一个Post请求接口，所以使用 Postman 调用接口时，首先需要选择 POST 请求方式。

![](https://images.gitbook.cn/15748289856184)

可以看到 Request Body 中 price 等于 200 时调用失败，因为 mapping 文件中设置了price 需要小于 200，说明
mapping 文件的配置生效了。

上面只是配置 Request Body ，除配置 Request Body
外还可以配置request中的headers和接口的访问权限。如下配置了Request 的 headers 和接口访问权限，header 中Content-
Type 是`application/json`，接口权限是 basicAuth。

    
    
    {
      "request": {
        "method": "POST",
        "urlPathPattern": "/api/addBookWithBasicAuth/([a-z]*)",
        "bodyPatterns":[{
          "matchesJsonPath": "$.books",
          "matchesJsonPath": "$.comment",
          "matchesJsonPath": "$..name",
          "matchesJsonPath": "$..price",
          "matchesJsonPath": "$..author",
          "matchesJsonPath": "$..[?(@.price<200)]"
        }],
        "headers": {
          "Content-Type": {"equalTo":"application/json"}
        },
        "basicAuth": {
          "username": "apiUsername",
          "password": "apiPassword"
        }
      },
      "response": {
        "status": 200,
        "body": "add book with bodyPatterns successfully"
      }
    }
    

下图是 Postman 调用模拟接口视频

![](https://images.gitbook.cn/15748289856207)

可以看到调用接口过程中如果没有设置 header 或者输入了错误的用户名或者密码，则调用接口失败。

当然如果需要模拟的 POST 请求接口对 Request Body ，header 和接口访问权限无任何限制，那么可以采用很简单的方式定义接口的
mapping 文件，文件内容如下图所示：

    
    
    {
      "request": {
        "method": "POST",
        "urlPathPattern": "/api/addBookWithAnyBody/([a-z]*)"
      },
      "response": {
        "status": 200,
        "body": "add book with any body successfully"
      }
    }
    

Postman 调用上述接口视频如下，可以看到任意修改 Request Body 的内容都能成功调用接口。

![](https://images.gitbook.cn/15748289856222)

上面讲解的都是 mapping 文件中 Request 的配置，接下来我们看如何配置接口的Response。相比 Request，Response
的配置简单很多，对于 Response 的 Body 有两种写法，一种是 Body 后面直接写内容，如下所示

    
    
     "response": {
        "status": 200,
        "body": "add book with any body successfully"
      }
    

另一种是 bodyFileName 的方式，这里 fileName 指在`__files` 目录下的文件名称。如下是采用 bodyFileName
的方式定义 Response，表示 Response Body内容为 WireMock
服务的`src/resources/mock/__files/firstCourse`目录中`bookDetails.json`文件内容。

实际上节课中模拟的第一个接口就是采用 BodyFileName 方式定义的，所以这里不再进行重复调用演示。

    
    
    {
      "request": {
        "method": "GET",
        "urlPathPattern": "/api/getBookResponse/([a-z]*)"
      },
      "response": {
        "status": 200,
        "bodyFileName": "firstCourse/bookDetails.json",
        "headers": {
          "Content-Type": "application/json"
        }
      }
    }
    

至此 JSON 格式的模拟就讲解完成了，通过上面的学习我们掌握了如何通过 wiremok 模拟get和 POST
请求的接口，另外还学习了如何设置接口的访问权限和 content-type。接着我们将学习如何模拟 XML 格式接口。

## 模拟 XML 格式接口

对于 XML 格式的接口，配置 urlPathPattern、headers 和 reponseFileName 与 JSON
格式的接口相同，这里就不再重复。

有区别的地方是 BodyPatterns 的配置，XML 格式的接口在 mapping 文件中配置 bodyPatterns 时有两种方式，第一种是使用
equalToxml 方式，第二种是使用 matchesXmlPath 方式。

假如我们需要 Mock 一个 Post 请求接口，接口的 Body 体如下，且 Body 中每个字段以及字段的值都是固定的。

    
    
    <person>
        <firstName>Done</firstName>
        <lastName>Jone</lastName>
    </person>
    

针对这样的接口我们可以采用 equalToxml 的方式，mapping 文件内容如下，可以看到和 Mock JSON 格式的接口相比，只是把 header
中 Content-Type 定义为 `application/xml`，BodyPattern 中使用 equalToXml 方式定义 Request
Body 内容。

    
    
    {
      "request": {
        "method": "POST",
        "urlPathPattern": "/api/addPersonByXml/([a-z]*)",
        "headers": {
          "Content-Type": {
            "equalTo": "application/xml"
          }
        },
        "bodyPatterns": [{
          "equalToXml": "<person><firstName>Done</firstName><lastName>Jone</lastName></person>"
        }]
      },
      "response": {
        "status": 200,
        "body": "add person with equalToXml successfully"
      }
    }
    

还是相同的方式，定义好 mapping 文件后，启动 WireMock 服务，用 Postman 调用 Mock 的接口。

![](https://images.gitbook.cn/15748289856245)

可以看到修改 Request Body 中的字段值或者字段名称或者增加新的字段，接口调用都会失败。equalToXml 方式是严格控制 Body
体的内容，假如只想控制接口的局部字段值，应该如何模拟呢？

例如如下接口，我们只想控制 Request Body 中 price 字段的值包含数字 1，Body体内容如下

    
    
    <book>
        <bookName>test</bookName>
        <price>12</price>
    </book>
    

对于这样的接口我们使用 matchesXmlPath 配置 mapping 文件，文件内容如下

    
    
    {
      "request": {
        "method": "POST",
        "urlPathPattern": "/api/addXmlUser/([a-z]*)",
        "headers": {
          "Content-Type": {
            "equalTo": "application/xml"
          }
        },
        "bodyPatterns": [
            "matchesXPath": {
              "expression": "//price/text()",
              "contains": "1"
            }
          }
        ]
      },
      "response": {
        "status": 200,
        "body": "add user with xpath bodyPatterns successfully"
      }
    }
    

上面的 mapping 文件我们定义了 content-type 必须是`application/xml`，其次我们定义了 Request Body
中存在节点 price，且 price 节点的值需要包含数字 1.

模拟好接口后，使用 Postman 调用接口进行验证，如下图所示

![](https://images.gitbook.cn/15748289856267)

可以看到，只要满足 Price 节点值包含 1 就可以成功调用接口，我们可以新增或修改任意节点名称或者值。

以上就是模拟 XML 格式接口的所有知识点，这里我们再总结一下

  * 如果对接口的 Body 体内容有严格要求（字段名称，字段值都有限制）那么可以采用equalToXml方式定义 Request Body 。
  * 如果只对接口 Body 体局部字段值有控制，那么可以采用matchesXpath的方式进行定义。
  * 如果对 Request Body 无任何要求，那么不配置bodyPatterns，只需把 Header 中content-Type设置为`application/xml` 即可。

至此《使用 WireMock 模拟被测接口》的内容都讲解完了，通过前面的两节课程，大家学习了如何搭建 WireMock 服务并模拟 JSON 格式和 XML
格式的接口，在后续的接口测试学习中已足够。如果你想了解更多关于 WireMock
的内容，可以查看官网资料（[http://WireMock.org/docs/）](http://WireMock.org/docs/%EF%BC%89%E3%80%82)

下节课将带领大家学习如何通过 REST Assured 完成接口调用。

我们为本专栏 **付费读者** 创建了微信交流群，以方便更有针对性地讨论专栏相关的问题。入群方式请添加 GitChat
小助手伽利略的微信号：GitChatty6（或扫描以下二维码），然后给小助手发「379」消息，即可拉你进群~

![](https://images.gitbook.cn/FsONnMw_1O_6pkv-U-ji0U1injRm)

