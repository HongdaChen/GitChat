上一节课程中学习了如何通过 Groovy 对数据库数据进行增删该查，本次课程将带领大家学习如何通过 Groovy 操作各类文件。例如读取
CSV、yml、JSON、XML、TXT 文件，因为接口测试大部分情况下都会遇到处理各类文件场景。

为了完成本次课程目标，我按文件类型分为了 5 个 Task

  * 读取写入 TXT 文件
  * 读取 yml 文件
  * 读取 CSV 文件
  * 读取 JSON 文件
  * 读取 XML 文件

为了解析 CSV 文件，需要在`pom.xml`文件中引入 groovycsv 包；为了解析 yml 文件，需要引入 snakeyaml
包，以下是`pom.xml`文件配置

    
    
    <properties>
     <groovycsv.version>1.3</groovycsv.version>
     <snakeyaml.version>1.17</snakeyaml.version>
     </properties>
     <dependencies>
        <dependency>
          <groupId>com.xlson.groovycsv</groupId>
          <artifactId>groovycsv</artifactId>
          <version>${groovycsv.version}</version>
        </dependency>
     <dependency>
          <groupId>org.yaml</groupId>
          <artifactId>snakeyaml</artifactId>
          <version>${snakeyaml.version}</version>
        </dependency>   
    </dependencies>    
    

创建 FileService Class 将文件相关的处理存放在该 Class 中

    
    
    package tl86.fourCourse.file
    
    import com.xlson.groovycsv.CsvParser
    import groovy.json.JsonSlurper
    import org.yaml.snakeyaml.Yaml
    
    class FileService {
        JsonSlurper jsonSlurper
        XmlSlurper xmlSlurper
        CsvParser csvParser
    
        FileService() {
            jsonSlurper = new JsonSlurper()
            xmlSlurper = new XmlSlurper()
            csvParser = new CsvParser()
        }
        // 如果不存在则创建，如果已存在则直接返回存在的文件对象
        def createFile(path) {
            new File(path)
        }
    
       //这里通过引入snackyaml包来解析yaml文件内容
        private def yml(String text) {
            new Yaml().load(text)
        }
    
        // 获取yml文件内容
        def getConfigs(String ymlFilePath) {
            def configs = yml(createFile(ymlFilePath).text)
            configs
        }
    
        // 将xml文件内容转换为groovy中数据集合
        // xmlSlurper是groovy自带的一款强大的解析xml文件的工具
        def getCollectionFromXMLFile(String xmlFilePath) {
            xmlSlurper.parse(createFile(xmlFilePath))
        }
    
        // 将json文件内容转换为groovy中数据集合
        // JsonSlurper是groovy自带的一款强大的解析json文件的工具
        def getCollectionFromJsonFile(String jsonFilePath) {
            jsonSlurper.parse(createFile(jsonFilePath))
        }
    
       // 获取csv文件内容，并支持不同类型的分隔符
       // 这里通过引入groovycsv包使用CsvParser
        def getCsvFileContent(String csvFilePath, separator) {
           csvParser.parse(new FileReader(createFile(csvFilePath)),separator:separator)
        }
    }
    

### 读取和写入 TXT 文件

创建名称为 Case 的 Class，写入如下内容，以下是操作 TXT 文件代码。通过调用前面FileService 中 createFile
方法获取文件对象，然后对文件进行写入、读取和删除操作。

    
    
    class Case extends Specification {
        FileService fileService
    
        def setup() {
            fileService = new FileService()
        }
    
        def "should create and read txt file successfully"() {
            given: "create txt file"
            def file = fileService.createFile("./src/test/resources/com/github/data/test.txt")    //这里请写入自己创建的文件路径
            when: "write some content to the file"
            // 支持通过<<写入文件内容
            file << "name,age,address\n"
            file << "Tom,100,chengdu\n"
            then: "print file content"
            // 读取txt文件内容
            def lines = file.readLines()
            lines.each{println it}
            and: "delete file"
            file.delete()
        }
    }
    

执行上述 Case 如果打印的内容是写入的内容则说明写入的内容都正确读取出来了，后面提供了运行本次课程所有代码的小视频。

### 读取 yaml 文件

以下是读取yml文件的代码，通过调用前面 FileService 中的 getConfigs 方法，读取 yml 文件内容极其简单。

创建名称为 `config.yml`的文件，内容如下，可以看到该文件配置的是前面使用的数据库连接信息。

接口测试中为了在不同环境中都能进行自动化测试，通常会把与环境相关的配置信息统一放到 yml 文件中管理。关于配置信息的管理后续还会详细介绍。

    
    
    active: dev
    dev:
      db:
        url: jdbc:mysql://127.0.0.1:3306/apitestdb
        user: root
        password: root
    stable:
      db:
        url: jdbc:mysql://127.0.0.1:3306/apitestdb
        user: root
        passowrd: root
    

通过编写 Case 校验是否正确获取 yml 文件内容

    
    
     def "should read yml file successfully"() {
            given: "no given"
            when: "get the csv data "
            def configs = fileService.getConfigs('./src/test/resources/com/github/config/config.yml')     //这里请写入自己创建的文件路径
            then: "print data"
            println configs.stable.db.url   //打印的值与config.yml文件中的值相同则说明成功获取到了yml文件内容
            println configs.active
        }
    

执行上述 Case 打印的 db.url，acitve 值与 yaml 文件中一致则说明获取数据正确，后面提供了运行本次课程所有代码的小视频。

## 读取 CSV 文件内容

以下是读取csv文件内容代码，接口测试中测试数据通常都存放在csv文件中进行统一管理。创建名称为`test.csv`文件， CSV 文件内容如下

    
    
    name,age,address
    TOM,10,China
    Dave,20,American
    

编写代码读取 CSV 文件内容

    
    
     def "should read csv file successfully"() {
            given: "no given"
            when: "get the csv file data"
            def csvContent = fileService.getCsvFileContent('./src/test/resources/com/github/data/test.csv', ',')   //这里请写入自己创建的文件路径
            then: "println the data"
            csvContent.each{ it -> println it.name + ":" + it.age + ":" + it.address }  
            //这里使用了groovy自带的处理数据集闭包each{}，打印csv文件中的所有name、age、address列内容
            //打印的值与csv文件内容一致则说明获取到了csv文件内容
        }
    

## 读取 JSON 文件

创建名称为`test.json`的文件，内容如下

    
    
    {"pipelineName": "myPipelineName",
      "env": "dev",
      "branch": "master",
      "stages": [
        {
          "id": "15000000000000",
          "name": "stage1",
          "status": "success",
          "duration": 66408
        },
        {
          "id": "15000000000001",
          "name": "stage2",
          "status": "success",
          "duration": 81462
        }
      ],
      "sonar": {
        "sqaleDebtRatio": 5.7,
        "sqaleRating": "B",
        "coverage": 16.7,
        "testNumber": 1.0,
        "link": "http://xxx.yyyy"
      }
    }
    

通过调用 FileService 中的方法快速查找任何期望的字段值

    
    
    def "should read json file successfully"() {
            given: ""
            when: "get json file data"
            def jsonContent = fileService.getCollectionFromJsonFile('./src/test/resources/com/github/data/test.json')   //这里请写入自己创建的文件路径
            then: "println the data"
            println jsonContent.pipelineName   //打印的值与json文件内容相同则说明正确获取到文件内容
            println jsonContent.sonar.coverage
            def stage = jsonContent.stages.find { it -> it.name == "stage2" }  //通过find方法查找json文件中stages对象下name等于“stage2”的对象
            println stage.id
            println stage.duration
        }
    

## 读取 XML 文件

一些应用的接口采用 XML 格式定义 request 和 response，所以掌握 XML 文件内容解析还是很有必要的。

创建名称为`test.xml`的文件，内容如下：

    
    
    <?xml version='1.0' ?>
    <doc>
        <person>
            <name>TOM</name>
            <age>21</age>
        </person>
        <person>
            <name>DAVE</name>
            <age>18</age>
        </person>
    </doc>
    

读取 XML 文件内容的代码如下

    
    
    def "should read xml file successfully"() {
            given: ""
            when: "get json file data"
            def xmlContent = fileService.getCollectionFromXMLFile('./src/test/resources/com/github/data/test.xml')   //这里请写入自己创建的文件路径
            then: "println the data"
            xmlContent.person.each{ println it }
            println xmlContent.person.find{ it -> it.name == "DAVE" }.age  //通过find方法查找XML文件中name等于“DAVE”的person对象，然后获取该对象下age的值
        }
    

运行上述 Case 打印的 age 是 18 则说明获取文件内容方式正确。

运行所有 Case 小视频，可以看到每个 Case 输出的内容与期望一致，说明都成功获取到了各类文件中的内容。

![](https://images.gitbook.cn/15749300966842)

前面学习了如何从`config.yml`文件中获取内容，上一节课程中学习使用Groovy 读取数据库中数据时，我们创建了 DataSource
Class，该 Class 中数据库连接信息是写死在代码中的，现在我们修改 DataSource 内容，从 config 中获取数据库配置信息，
DataSourceNew 内容如下

    
    
    import groovy.sql.Sql
    import com.github.fourCourse.file.FileService
    
    class DataSourceNew {
        Sql sql
        FileService fileService
        def configs
    
        DataSourceNew() {
            fileService = new FileService()
            configs = fileService.getConfigs('./src/test/resources/com/github/config/config.yml')
        }
    
        Sql getSql() {
            if (!sql) {
                def mysqlDB = [
                        driver  : 'com.mysql.jdbc.Driver',
                        url     : configs.dev.db.url,
                        user    : configs.dev.db.user,
                        password: configs.dev.db.password
                ]
                sql = Sql.newInstance(mysqlDB.url, mysqlDB.user, mysqlDB.password, mysqlDB.driver)
            }
            sql
        }
    }
    

修改上一届课程中创建的 DataRepository Class，继承 DataSourceNew。代码如下图所示

    
    
    class DataRepository extends DataSourceNew{}
    

重新运行上一节课程中获取数据库数据的 Case，能成功运行如下图所示，说明正确获取到数据库数据。
![](https://images.gitbook.cn/15749300966867)

在接口测试中为了让同一套测试代码能在不同的环境中自动切换运行，通常会把与环境相关的配置信息统一放到 yaml 文件中，当需要切换环境时，修改 config
文件中环境值即可。

后续课程中会详细介绍如何通过修改环境变量或者 config 中值切换到不同环境运行相同的case。

至此 Groovy 部分的介绍全部结束了，后续在接口测试场景中还会多次使用这两次课所学内容。

如果之前无 Java 或者 Groovy 编程经验的同学，强烈建议多练习这两节课程的内容，做到能独立利用 Groovy
对数据库中表信息进行增删该查，获取各类文件信息，因为在后续接口测试项目实战课程中需要大家有一定编程基础才能完成课程目标。

