上一节中我们学习了利用 WireMock 框架模拟各种类型的接口，本节课将带领大家学习如何使用 REST Assured 调用 Mock 的接口，并生成
HTML 格式的测试报告。为了完成今天的课程目标，我分了两个 Task：

  * Task1：配置接口测试项目
  * Task2：编写一个调用接口 Case

## 配置接口测试项目

在正式开始编写测试代码前，我们还需要对项目做一些设置。我将设置的任务分成了4个步骤，接下来我们将分步完成

  * 设置目录属性
  * 添加`.gitignore`和`README.md`文件
  * 添加和下载依赖
  * 添加一个简单的测试场景验证

首先使用 intelliJ 打开在“初始化项目”课程中创建的接口测试项目，如下图所示

![](https://images.gitbook.cn/15748453060828)

因为接口测试属于测试代码，所以所有的代码应该存放在`src/test`目录下。另外因为采用 Groovy 语言，所以 test 目录下创建 Groovy
目录， Groovy 目录下是项目包名，包名规则是公司域名反写，如果不能写公司的域名请用 Github 的域名，课程中使用的包名是com.github。

创建好目录后选中 Groovy 点击右键，选中「Mark Directory As」,然后选中“Test Sources Root”。

如下图所示：

![](https://images.gitbook.cn/15748453060840)

同理在`src/test`目录下创建 resources 目录，右键将其设置为「Test Resources
Root」。设置完目录属性后，接着添加`.gitignore`文件，该文件与 src 在同一级目录，文件内容如下：

    
    
    target/
    .idea/
    *.iml
    *.ipr
    *.iws
    

`.gitignore`文件作用是在 Push 代码到代码管理仓库（Github）时不会把多余的内容上传到代码管理仓库。

例如 target 目录、.idea 目录等，因为其他人通过代码管理仓库下载你的代码时，根本不需要这些构建产物。添加好`.gitignore`文件后添加
`README.md`， 该文件与 src 也在同一级目录，该文件作用就是对项目做一个简单说明。

设置后的项目代码应该如下图所示：

![](https://images.gitbook.cn/15748453060851)

接下来添加所需依赖,为了完成接口调用和生成报告，需要在项目中引入如下插件和jar包

编号 | 插件名称 | groupId | artifactId | 作用  
---|---|---|---|---  
1 | Groovy 插件 | org.codehaus.gmavenplus | gmavenplus-plugin | 安装该插件后才能创建
Groovy 的class  
2 | surfire插件 | org.apache.maven.surefire | maven-surefire-plugin | 执行测试case需要  
3 | Groovy 包 | org.codehaus.groovy | Groovy-all | 使用 Groovy 语言  
4 | REST Assured | io.rest-assured | REST Assured | 引入该包后才能使用该框架调用接口  
5 | spock-core | org.spockframework | spock-core | 引入该包后才能使用spock（ Groovy
自带的BDD框架，后续会详细介绍）  
6 | spock-report | com.athaydes | spock-reports | 引入该包运行case后才能生成对应的测试报告  
  
因为是maven作为构建工具，故所有依赖相关的信息都配置在pom.xml中，项目初始化后pom.xml文件中有默认的内容，例如properties，dependencies,plugins等，如下图所示

![](https://images.gitbook.cn/15748453060865)

为了引入新的依赖，只需把相关内容添加到对应位置即可。properties
下面统一配置依赖的版本信息，这样可以在统一的位置查看所有的依赖版本是否正确。当然也可以在配置依赖的时候填写版本信息。

properties 部分配置内容如下所示：

    
    
     <properties>
        <rest-assured.version>3.0.5</rest-assured.version>
        <groovy.version>2.4.12</groovy.version>
        <spock-core.version>1.1-groovy-2.4</spock-core.version>
        <spock-report.version>1.5.0</spock-report.version>
        <surefire.version>2.22.0</surefire.version>
      </properties>
    

dependencies下面配置所需依赖的 groupId，artifactId 和 version 信息，内容如下所示

    
    
    <dependencies>
        <dependency>
          <groupId>org.codehaus.groovy</groupId>    
          <artifactId>groovy-all</artifactId>
          <version>${groovy.version}</version>        //因为在properties里面已经配置了groovy.version的值，这里只需引用该值即可
          <scope>test</scope>
        </dependency>
        <dependency>
          <groupId>io.rest-assured</groupId>
          <artifactId>rest-assured</artifactId>
          <version>${rest-assured.version}</version>
        </dependency>
        <dependency>
          <groupId>org.spockframework</groupId>
          <artifactId>spock-core</artifactId>
          <version>${spock-core.version}</version>
        </dependency>
        <dependency>
          <groupId>com.athaydes</groupId>
          <artifactId>spock-reports</artifactId>
          <version>${spock-report.version}</version>
        </dependency>
      </dependencies>
    

plugins下面配置gmavenplus和surefire插件

    
    
    <plugins>
            <plugin>
              <groupId>org.codehaus.gmavenplus</groupId>
              <artifactId>gmavenplus-plugin</artifactId>
              <version>1.5</version>
            </plugin>
           <plugin>
              <groupId>org.apache.maven.plugins</groupId>
              <artifactId>maven-surefire-plugin</artifactId>
              <version>${surefire.version}</version>
                  <dependencies>
                      <dependency>
                          <groupId>org.apache.maven.surefire</groupId>
                          <artifactId>surefire-junit47</artifactId>
                          <version>2.22.0</version>                         //这里是在配置依赖时填写版本信息，没有从properties里面读取
                      </dependency>
                  </dependencies>
           </plugin>     
    </plugins>     
    

配置好pom.xml文件后，打开命令行工具，cd到项目根目录，执行mvn clean install下载所需依赖。

![](https://images.gitbook.cn/15748453060884)

如果得到“Build Success”信息，如下图所示，说明命令执行成功

![](https://images.gitbook.cn/15748453060912)

IntelliJ中点击右边的“Maven”按钮，确认配置的依赖都下载成功，如果Dependencies下面看不到配置的依赖，可以点击上面的刷新按钮（红框标注的按钮）进行同步。如下图所示，配置的依赖在IntelliJ中都能查看到，说明依赖下载成功

![](https://images.gitbook.cn/15748453060929)

依赖下载成功后就可以创建Groovy的Class了，创建方式如下图所示，IntelliJ中右键选择“New”，然后选则Groovy Class，如下图所示

![](https://images.gitbook.cn/15748453060946)

至此项目所有设置工作全部完成，接下来可以开始编写调用接口代码了。

## 编写一个调用接口 Case

创建一个名为 FirstDemo 的 Groovy
Class，然后在该Class上实现调用getUserDetails这个接口（也就是第一节课程中模拟的第一个接口）。以下是调用接口的代码， REST
Assured 的使用规则稍后将详细介绍。

    
    
    package github.com.thirdCourse
    
    import spock.lang.Specification
    import static io.restassured.RestAssured.given
    
    class FirstDemo extends Specification {
    
        def "should call mock api successfully"() {     //spock框架（BDD框架）语法，所有case都是def开头，def后面是该case的描述信息
            given: "no given"                           //spock框架语法，given-when-then三段式写法，given/when/then后是描述信息
            when: "call mock api api"
            given().baseUri("http://localhost:9090")    //这里输入接口的baseUri
                    .when()
                    .get("api/getUserDetails")          //输入接口的地址
                    .then()
                    .assertThat().statusCode(200)      //这里校验调用接口后返回的状态码是200，如果不是200，调用会失败
            then: "no then"
        }
    }
    

编写好代码后，运行编写的测试脚本有三种方式，第一是命令行工具中执行mvn test。第二是IntelliJ中选中FirstDemo
Class，右键运行。第三是IntelliJ中输入mvn test运行case。

如下图所示，演示在命令行工具中运行FirstDemo，首先cd到项目根目录，然后执行mvn test
-DTest=“FirstDemo”（这里指定了只执行FirstDemo这个测试Class）

![](https://images.gitbook.cn/15748453060973)

IntelliJ的Terminal中执行mvn test（该命令会执行项目中所有的测试Class）

![](https://images.gitbook.cn/15748453060991)

第三种方式：IntellJ中右键运行,选中FirstDemo，点击右键，选中“Run FirstDemo”即可，这里就不再演示。

如果在运行过程中遇到错误，请确保IntelliJ中配置的Jdk是1.8版本。如下图所示，新下载的IntelliJ配置的是Jdk11版本。

![](https://images.gitbook.cn/15748453061006)

修改 IntelliJ 配置的 JDK 版本方法是：先下载 JDK1.8，下载地址：[下载
JDK1.8](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)。

如果是 Windows 系统，可以下载 EXE 安装包，如果安装路径选择 C 盘，那么在`C:\Program Files\Java`路径下应该有
JDK1.8的文件夹。

如果是 Mac 系统，Mac 系统自带了安装好的 JDK，在命令框中执行`/usr/libexec/java_home -V`即可知道 JDK
的安装目录。找到该文件夹后，然后选中 IntelliJ 中菜单「File-ProjecStructure」，配置 1.8 的 SDK。

如下图所示演示了如何在 Mac 系统上把 IntelliJ 中 JDK 版本设置为 1.8，配置 SDK 时选 JDK 安装目录：

![](https://images.gitbook.cn/15748453061025)![](https://images.gitbook.cn/15748453061042)

配置 Project 确保选择的是 JDK1.8

![](https://images.gitbook.cn/15748453061058)

配置 Modules 确保选中的是 JDK1.8

![](https://images.gitbook.cn/15748453061074)

设置完后，点击右下角的 Apply，OK 按钮，保存新的设置，此时重新回到项目中，查看 JDK 的版本应该变为了 JDK1.8。

如下图所示,修改 Java11 为 JDK1.8

![](https://images.gitbook.cn/15748453061090)

至此一个接口调用的 Case 就完成了，接下来将给大家解析上面这段代码使用到的框架。实际这段代码中使用了两个框架， REST Assured 和spock。
REST Assured 框架负责接口本身的调用和校验，也是 given-when-then
模式，这里对接口的响应做了最简单的返回状态码验证。下面的代码片段属于 REST Assured 框架的内容

    
    
    given()                      //固定写法， REST Assured 自身也是三段式写法，given-when-then
          .baseUri("http://localhost:9090")   
          .when()               // 固定写法
          .get("api/getUserDetails")     // 支持get，post，delete等，括号里面是接口路径
          .then()              //固定写法
          .assertThat().statusCode(200)    
    

Spock是一个BDD框架，每一个case以def开头，def这里可以添加该case所覆盖的业务场景描述信息，case内容上支持given-when-
then三段式。为了使用spock框架所有测试Class都需要继承Specification。以下代码片段属于spock内容

    
    
    def "should call mock api successfully"() {
            given: "no given"
            when: "call mock api api"
            then: "no then"
    

学到这里你是否会疑虑 REST Assured 已经是三段式（given-when-then）写法了，为什么还要使用 Spock 呢？因为 Spock
的作用不仅仅是支持三段式。一段代码能执行要么将代码放到main方法中，要么就是套用一些测试框架，例如使用很广泛的单元测试框架junit，如果要运行一段代码，那么需要添加@Test注解才能运行。Spock底层实际套用的也是junit，所以这里使用spock的另外一个作用是让编写的代码能运行起来。

至此今天的课程就结束了，怎么样？编写一个接口调用的测试脚本实际就是这么简单。

本次课程中除编写测试脚本外还涉及到 IntelliJ
工具的使用，如果之前未使用过这个工具，在使用过程中遇到任何问题，可以查看官网文档：[https://www.w3cschool.cn/intellij
_idea_ doc/](https://www.w3cschool.cn/intellij_idea_doc/)。

下次课程将带领大家学习 REST Assured 使用规则。

