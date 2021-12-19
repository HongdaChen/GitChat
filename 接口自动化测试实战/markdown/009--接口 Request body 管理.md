上三节课中我们学习了 Groovy 的内容，本次课程将带领大家学习如何管理接口测试的request body。一个项目生命周期中一个接口的Request
Body 可能会进行多次改动，本节课程将介绍如何通过模板引擎工具 velocity 有效管理 Request Body，降低编写和维护 Case
的成本。为了完成上述目标我拆分了两个 Task

  * 直接通过文件方式管理 Reqeust Body
  * 通过 velocity 管理 Request Body

## 直接通过文件方式管理 Reqeust Body

首先我们通过 WireMock模拟一个 POST 接口，接口的 Url
为`http://localhost:9090/api/addUserDetails`，以下是模拟接口的 mapping 文件，mapping 文件只限制了
Request Body 是 JSON 格式。

    
    
    {
      "request": { 
        "method": "POST",
        "urlPath": "/api/addUserDetails",
        "headers": {
          "Content-Type": "application/json; charset=UTF-8"
        }
      },
      "response": {
        "status": 200,
        "body": "add user successfully"
      }
    }
    

假如我们的 Request Body 如下,包含`name`、`age`，两个 contact 信息和backgroud，一个 description
信息：

    
    
     {
      "name": "TOM",
      "age": 10,
      "contacts": [
        {
          "city": "chengdu",
          "street": "huaxi-Street",
          "phone": "11122222222"
        },
        {
          "city": "meijing",
          "street": "qinghua-street",
          "phone": "33333444"
        }
      ],
      "background": {
        "degree": "doctor",
        "educate school": "Beijing univercity",
        "graduate Date": "2019-7"
      },
      "otherDescription": "any comment"
    }
    

我们先通过文件方式传入 Request Body 调用接口。接口调用的代码存放在 UserClient Class中，Request Body的构建和
Response 的解析在其他 Class 中完成。

UserClient Class 代码如下

    
    
    class UserClient {
        def addUserWithFile(File file) {
            def res = given().baseUri("http://localhost:9090")
                    .when()
                    .body(file)                                    //body参数中传入File对象
                    .post("/api/addUserDetails")
                    .then().assertThat().statusCode(200)
                    .extract().response().getBody().asString()     //获取接口的response body
            res
        }
    
        def addUserWithString(String body) {
            def res = given().baseUri("http://localhost:9090")
                    .when()
                    .body(body)                                  //body参数中传入接口的request body字符串
                    .post("/api/addUserDetails")
                    .then().assertThat().statusCode(200)
                    .extract().response().getBody().asString()  //获取接口的response body
            res
        }
    }
    

测试场景代码

    
    
    class Case extends Specification {
        FileService fileService
        UserClient userClient
        def setup() {
            fileService = new FileService()
            userClient = new UserClient()
        }
        def "should add user successfully"() {
            given: "no given"
    
            when: "call the add user api"
              // 创建addUser.json文件，将前面列出的request body内容放到该文件中。
            def file = fileService.createFile("./src/test/resources/com/github/body/addUser.json")  //获取文件对象
    
            then: "get the correct response"
            Assert.assertEquals(userClient.addUserWithFile(file), "add user successfully")         
            //将文件对象传入userClient.addUserWithFile()中，这里可以开启.log().all()查看接口是否返回正确的response body。
        }
    

运行上述 Case 如下图所示，可以看到接口返回了正确的 Response body。

![](media/15731156690628/callInterFace.gif)

以上就是通过文件管理 Request Body 的实现方式，一个接口用了一个文件来存放Request
Body，似乎没什么问题，接下来我们看看这样的测试场景。

假如 addUser 接口的实际业务场景如下：

  * 必填字段是 name 和 age
  * contact 可选字段，contact 包含所在的城市，街道和联系电话
  * 可以输入多个 contact 信息，最多输入两个
  * 可以输入教育背景信息，可选字段
  * 可以输入其他信息，可选字段，是一个富文本

想象一下如果把前面的场景转换为 Case（各种必填和非必填组合），那么 Case 至少在10 个以上，如果每个 Case 都用一个文件管理对应的
Request Body，那么一个业务场景就需要 10 个以上的 Request Body File，如果所有 Case 都这样管理Request
Body，一个复杂系统的接口测试维护成本就非常大。

那有没有更好的方式管理 Request Body 呢？当然有，接下来我们将介绍如何通过 velocity 只需一个 file 即可覆盖上述的所有场景。

## velocity 管理 Request Body

为了使用 velocity，首先在`pom.xml`文件中添加对应的依赖，`pom.xml`配置如下

    
    
    <properties>
    <velocity.version>1.7</velocity.version>
     </properties>
    
    
    
    <dependency>
          <groupId>org.apache.velocity</groupId>
          <artifactId>velocity</artifactId>
          <version>${velocity.version}</version>
        </dependency>
    

以下代码是如何通过 velocity 将数据对象与模板文件进行 merge

    
    
    class TemplateService {
        VelocityEngine velocityEngine = new VelocityEngine()
        VelocityContext velocityContext = new VelocityContext()
        StringWriter stringWriter = new StringWriter()
    
        def getAddUserRequestBody(addUserBody) {
            velocityContext.put("addUserBody",addUserBody)   
            velocityEngine.getTemplate("src/test/resources/com/github/body/template/addUserTemplate.json").merge(velocityContext,stringWriter)
            stringWriter.toString()
          // 上面三行属于固定写法，目的是把数据对象addUserBody和模版文件进行merge
          // 如果对这部分内容不理解，不用急，待后续查看了接口调用时构造出来的request body后再反过来看velocity这个工具，理解起来就会简单很多
        }
    }
    

创建名称为`addUserTemplate.json`的文件并写入如下内容。

因为前面 merge 的是 addUserBody 对象，所以模板文件中所有参数化的变量都是`$addUserBody`开头

    
    
    {
      "name": "TOM",
      "age": 10,
      "contacts": [
        #if ($addUserBody.ifAddMainContact)     // #If...#end表示如果条件为true，那么在body merge中就有此内容，反之则无这段内容。通过这些设置可以根据需要动态组合构造出来的reqeust body
        {
          "city": $addUserBody.mainContact.city,  // 所有以$开头的都是后续可以参数化的内容
          "street": $addUserBody.mainContact.street,
          "phone": $addUserBody.mainContact.phone
        }
       #end
       #if($addUserBody.ifAddBackupContact)
      ,
        {
        "city": $addUserBody.backupContact.city,
        "street": $addUserBody.backupContact.street,
        "phone": $addUserBody.backupContact.phone
        }
      #end
      ]
      #if ($addUserBody.ifAddBackGround)
      "background": {
        "degree": $addUserBody.backGround.degree,
        "educate school": $addUserBody.backGround.school,
        "graduate Date": $addUserBody.backGround.date
      }
      #end
      #if ($addUserBody.ifAddOtherInfo)
      ,
      "others description": "any comment"
      #end
    }
    

定义好模板文件后，接下来就是定义 addUserBody 对象，采用 build 模式来构建该对象，以下是对象构建代码

    
    
    class AddUserBody {
        def mainContact=[:]
        def ifAddMainContact
        def backupContact=[:]
        def ifAddBackupContact
        def backGround=[:]
        def ifAddBackGround
        def otherInfo
        def ifAddOtherInfo
        UserClient userClient
    
        AddUserBody() {
            userClient = new UserClient()
        }
    
        def addMainContact(city, street, phone) {
            this.ifAddMainContact = true
            this.mainContact.city = city
            this.mainContact.street = street
            this.mainContact.phone = phone
            this
        }
    
        def addBackupContact(city, street, phone) {
            this.ifAddBackupContact = true
            this.backupContact.city = city
            this.backupContact.street = street
            this.backupContact.phone = phone
            this
        }
    
        def addBackGround(degree, school, date) {
            this.ifAddBackGround = true
            this.backGround.degree = degree
            this.backGround.school = school
            this.backGround.date = date
            this
        }
    
        def addOtherInfo(otherInfo) {
            this.ifAddOtherInfo = true
            this.otherInfo = otherInfo
            this
        }
    
        def generateBody() {
          new TemplateService().getAddUserRequestBody(this)  //this表示AddUserBody对象本身，将this传递给templateService，那么该对象中设置的所有值就可以用在模版文件中了
        }
    }
    

通过上面的配置就可以实现按需构造 body 了。例如添加 user 时，只输入一个 contact 信息，其他信息都不输入，那么构造该 body
的语句如下.

    
    
    def body = new AddUserBody()
                    .addMainContact(city, street, phone)
                    .generateBody()
    

再例如添加 user 时，需要输入两个 contact 信息和 background 信息，那么构造 Body 的语句如下.

    
    
     def body = new AddUserBody()
                    .addMainContact(mainCity, mainStreet, mainPhone)
                    .addBackupContact(backupCity, backupStreet, backupPhone)
                    .addBackGround(degree, school, date)
                    .generateBody()
    

可以看到通过使用 velocity 和采用 build 模式，轻松实现了按需构造接口的 Request
Body，一个模板文件就可以覆盖所有的测试场景。以下是挑选了其中几个场景编写的测试 Case。

只添加 mainContat 信息时调用接口

    
    
     def "should add user with only inputting main contact successfully"() {
            given: "generate request body"
            def body = new AddUserBody()
                    .addMainContact(city, street, phone)
                    .generateBody()
            when: "call add user api"
            def response = userClient.addUserWithString(body)
            then: "should get correct response"
            Assert.assertEquals("assert add user api response correct", response, "add user successfully")
            where:
            city      | street          | phone
            "chengdu" | "qingyi-street" | 11223344556
        }
    

userClient 中开启`.log().all()`，运行上面的 Case，可以看到调用接口时发送的Request Body 只包含
mainContact

![](media/15731156690628/callInterFaceTwo.gif)

再例如如下例子，两个case在构造接口的request body时添加了不同的信息。

    
    
        def "should add user with inputting main and backup contact successfully"() {
            given: "generate request body"
            def body = new AddUserBody()
                    .addMainContact(mainCity, mainStreet, mainPhone)              //添加mianContact信息
                    .addBackupContact(backupCity, backupStreet, backupPhone)      //添加BackupContact信息
                    .generateBody()
            when: "call add user api"
            def response = userClient.addUserWithString(body)
            then: "should get correct response"
            Assert.assertEquals("assert add user api response correct", response, "add user successfully")
            where:
            mainCity  | mainStreet   | mainPhone   | backupCity | backupStreet | backupPhone
            "chengdu" | "one-street" | 11223344556 | "beijing"  | "two-street" | 00112233445
        }
    
        def "should add user with inputting contacts and background successfully"() {
            given: "generate request body"
            def body = new AddUserBody()
                    .addMainContact(mainCity, mainStreet, mainPhone)           //添加mianContact信息       
                    .addBackupContact(backupCity, backupStreet, backupPhone)   //添加BackupContact信息
                    .addBackGround(degree, school, date)                       //添加BackGround信息
                    .generateBody()
            when: "call add user api"
            def response = userClient.addUserWithString(body)
            then: "should get correct response"
            Assert.assertEquals("assert add user api response correct", response, "add user successfully")
            where:
            mainCity  | mainStreet   | mainPhone   | backupCity | backupStreet | backupPhone | degree   | school    | date
            "chengdu" | "one-street" | 11223344556 | "beijing"  | "two-street" | 00112233445 | "doctor" | "qinghua" | "2019-07"
        }
    

`

运行上面的 Case，查看调用接口时发送的 Request Body，与设置的一致。

![](https://images.gitbook.cn/FuDspy.gif)

至此今天的课程就完成了，本节课程主要为大家讲解了如何通过模板引擎工具更好的管理Request Body，下节课将为大家讲解接口 Response 的解析。

```

