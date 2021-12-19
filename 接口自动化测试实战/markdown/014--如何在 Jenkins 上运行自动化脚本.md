上一节课程中我们学习了如何统一管理配置信息和测试数据，此次课程将带领大家学习如何运行 Jenkins ，然后把前面写的测试脚本放到 Jenkins
上运行。最后介绍如何通过划分testSuite管理测试案例。为了完成此次课程目标，我分了2个task。

  * 配置 Jenkins 并让测试脚本在 Jenkins 上运行
  * 划分testSuite

## 安装配置 Jenkins

安装 Jenkins 有多种方式，本课程使用运行jar包的方式启动 Jenkins 。

  * 首先下载 Jenkins.war， 

Jenkins.war下载地址 <https://updates. Jenkins -ci.org/download/war/> ，

下载后执行命令`java -jar Jenkins .war`启动 Jenkins 。成功安装后会看到“Installation successful:
Mailer”的信息，在浏览器中输入地址（[http://127.0.0.1:8080/）](http://127.0.0.1:8080/%EF%BC%89)
能显示 Jenkins UI界面则表示安装成功。

首次访问时需要输入安装时初始化的密码, Jenkins UI界面上会显示密码文件地址，界面如图所示

![](https://images.gitbook.cn/15758727916727)

从文件中查看并输入密码后，选择安装默认的插件，插件安装完成后就可以开始创建job了，安装好后的 Jenkins 如下图所示

![](https://images.gitbook.cn/15758727916741)

创建一个freestyle的job，进行job配置前，需要先配置credential，配置步骤如下所示

  * 自动化脚本的repo地址（在配置 Jenkins job前请将自己的自动化脚本上传到自己的github仓库）
  * 在 Jenkins 上创建credential key,配置步骤如下图所示，进入Credential菜单，然后选择add credential

![](https://images.gitbook.cn/15758726882671)

这里输入的用户名、密码为github的账号与密码，scope保持默认即可。配置好credential后，后面配置job的github信息时即可选择配置好的credential。

![](https://images.gitbook.cn/15758726882711)

配置好credentials后就可以进行job的配置了，配置步骤如下所示

  * 配置job命令，因为项目使用maven作为构建工具，所以可以使用mvn test：执行所有的case或使用mvn test -Dtest="testClassName"执行某一个具体的test class或者使用mvn test -Dtest="./src/test/firstCourse/*Test"执行某个目录下所有以Test结尾的test class。这里我们先使用第二种方式执行本课程中具体的test class，配置的命令是mvn test -Dtest="GetDataClient"。job配置的详细信息如下

  * 配置从github拉取代码，选择的credential为上一步创建的credential

![](https://images.gitbook.cn/15758726882724)

  * 配置job的构建命令

![](https://images.gitbook.cn/15758726882736)

完成job配置后，执行job，应该能得到执行成功的信息，查看job执行日志，日志信息如下

![](https://images.gitbook.cn/15758726882747)

可以看到确实执行了GetDataClient中的两个测试方法。前面我们讲过可以在 Jenkins
上配置ACTIVE_ENV来模拟切换到不同环境进行自动化测试，这里我们修改job执行命令，添加环境变量的配置，修改 Jenkins
上Command内容如下所示

    
    
    export ACTIVE_ENV="dev" && mvn test -Dtest="GetDataClient"
    

再次执行job应该能够运行成功，因为mock-server默认的端口就是dev环境的端口。

修改job命令指定环境为stable

    
    
    export ACTIVE_ENV="stable" && mvn test -DTest="GetDataClient"
    

再次运行job，会运行失败，查看job运行日志，详情如下

![](https://images.gitbook.cn/15758726882761)

可以看到接口调用失败，因为stable环境指定的接口地址与实际mock-server启动的接口地址不一致。

通过上面的学习，我们掌握了如何通过管理测试数据、配置信息以及环境变量值来达到相同的自动化case在多个环境中执行的目标。除此之外，在实际项目中我们为了灵活运行不同范围的test
case，还会把case划分到不同的testSuite中，这样可以根据修改代码的影响范围执行对应的自动化case，缩短反馈时间。接下来将带领大家学习如何把case拆分到不同的testSuite中。

## testSuite让运行更加灵活

这里我们采用junit中的category方式将case划分到不同的testSuite，采用category的优势是可以基于method划分，即同一个class中的不同method可以划分到不同的testSuite。
为了实现通过category将不同的case划分到不同的testSuite，需要做如下步骤

  * 创建Interface，这里我们创建名称为FirstCategory，SecondCategory的interface

    
    
    interface FirstCategory {
    
    }
    
    interface SecondCategory {
    
    }
    

  * 创建Test Class，这里创建名称为TestClassA，TestClassB的class，每个class中存在两个method。TestClassA中firstMethod属于FirstCategory，secondMethod属于SecondCategory，TestClassB中Category注解放在class级别，即TestClassB中所有method都属于FirstCategory，代码细节如下

    
    
    class TestClassA {
        @Category([FirstCategory])    //这里表示firstMethod属于FirstCategory
        @Test()
        void firstMethod() {
            println("this is first method from TestClassA")
        }
        @Category([SecondCategory])   //这里表示secondMethod属于SecondCategory
        @Test()
        void secondMethod() {
            println("this is second method from TestClassA")
        }
    }
    @Category([FirstCategory])     //这里表示整个Class中的所有case都属于FistCategory
    class TestClassB {
    
        @Test
        void firstMethod() {
            println "this is first method from TestClassB"
        }
    
        @Test
        void secondMethod() {
            println "this is second method from TestClassB"
        }
    }
    

  * 创建TestSuite，这里创建名称为FirstTestSuite的class，TestSuite中指定IncludeCategory为FirstCategory，SuiteClasses中指定执行的class包含TestClassA和TestClassB

    
    
    @RunWith(Categories.class)
    @Categories.IncludeCategory(FirstCategory.class)
    @Suite.SuiteClasses([TestClassA,TestClassB])
    class FirstTestSuite {
    }
    

通过上面的定义，执行该TestSuite的时候实际是执行TestClassA中的firstMethod和TestClassB中的两个method。在实际项目中可以根据业务场景划分不同的TestSuite，这样可以灵活指定每次执行的测试集。

执行命令：mvn test
-Dtest="**/testSuite/FirstTestSuite"如下图所示，可以看到对应的三个method被成功执行，说明testSuite配置生效。

![](https://images.gitbook.cn/15758726882777)

至此，本次课程的讲解就结束了，本次课程中主要学习了如何在 Jenkins
上设置环境变量让自动化脚本在多环境中切换运行，以及如何配置TestSuite。截止目前所有关于接口自动化的知识点讲解就都完成了。下次课程将带领大家搭建一个真实的web应用，然后利用前面所学编写web应用的自动化case。如果对前面所学掌握的不够熟练，强烈建议大家多多练习前面的内容。

