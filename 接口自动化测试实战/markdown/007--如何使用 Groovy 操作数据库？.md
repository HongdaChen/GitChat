上一节课程中我们学习了 Groovy 的部分语法、闭包和集合处理的知识，本次课程将带领大家学习如何使用 Groovy 操作数据库、编写 Groovy
脚本。为了完成本次课程目标，我划分了两个 task：

  * Groovy 操作数据库
  * 编写和执行 Groovy 脚本

## groovy操作数据库

在开始学习前请安装 MySQL服务器和客户端并创建对应的 Database。

  * MySQL 和 navicat 安装文档 ：[Windows 安装 MySQL 以及客户端](https://www.jianshu.com/p/6c1b62b002f4)
  * 创建 DataBase，创建名为 apitestdb 的 DataBase，并初始化一些表和数据。

以下是在DataBase中创建一张 user 表和 address 表并初始化一些数据的 SQL：

    
    
    CREATE TABLE IF NOT EXISTS `user`(
      `id` INT UNSIGNED AUTO_INCREMENT,
      `username` VARCHAR(100) NOT NULL,
      `age` INTEGER NOT NULL,
      `create_date` DATE,
      PRIMARY KEY ( `id` )
    )ENGINE=InnoDB DEFAULT CHARSET=utf8;
    
    CREATE TABLE IF NOT EXISTS `address`(
      `id` INT UNSIGNED AUTO_INCREMENT,
      `userId` INT ,
      `address` VARCHAR(100) NOT NULL,
      `create_date` DATE,
      PRIMARY KEY ( `id` )
    )ENGINE=InnoDB DEFAULT CHARSET=utf8;
    
    INSERT INTO USER (username,age,create_date)VALUES ('TOM',10,'2019-10-11 12:10:10');
    INSERT INTO USER (username,age,create_date)VALUES ('DONE',15,'2019-10-10 12:10:10');
    INSERT INTO USER (username,age,create_date)VALUES ('ECHO',20,'2019-10-15 12:10:10');
    INSERT INTO USER (username,age,create_date)VALUES ('MARY',10,'2019-10-10 12:10:10');
    INSERT INTO address (userId,address,create_date)VALUE (1,'chengdu','2019-11-12 00:00:00');
    INSERT INTO address (userId,address,create_date)VALUE (2,'beijing','2019-11-12 00:00:00');
    INSERT INTO address (userId,address,create_date)VALUE (3,'shanghai','2019-11-12 00:00:00');
    INSERT INTO address (userId,address,create_date)VALUE (4,'hangzhou','2019-11-12 00:00:00');
    

通过客户端工具例如 navicat 执行上述 SQL，执行完后确保数据是完整的，因为后面会编写代码来获取数据库数据，如果数据不正确，测试 Case
会运行失败。

如下图所示，笔者使用命令查看数据库中数据确实已初始化成功。

![](https://images.gitbook.cn/15749193727197)

为了在接口测试项目中编写连接数据库的代码，需要添加mysql依赖。接口测试项目pom.xml文件中需添加内容如下。

properties中添加 version 信息

    
    
     <mysql.version>5.1.44</mysql.version>
    

dependencies下添加依赖

    
    
     <dependency>
          <groupId>mysql</groupId>
          <artifactId>mysql-connector-java</artifactId>
          <version>${mysql.version}</version>
          <scope>test</scope>
        </dependency>
    

配置好`pom.xml`文件后，在命令框或者 IntellJ 中执行`mvn clean install`下载依赖。

下载依赖后就可以开始编写操作数据库的代码了，创建DataSource，ConstantSql，DataRepository 三个 Class。

  * DataSources 负责数据库的连接
  * ConstantSql 存放 SQL 常量
  * DataRepository 存放操作数据库的方法

以下是DataSources的代码，实际项目中如果需要连接多个数据库或者多种类型数据库，例如mongoDB,oracle等都可以放到该Class中

    
    
    class DataSource {
        Sql sql
        Sql getSql() {
            if (!sql) {
                def mysqlDB = [
                        driver  : 'com.mysql.jdbc.Driver',
                        url     : 'jdbc:mysql://127.0.0.1:3306/apitestdb',
                        user    : 'root',          //这里写入安装mysql是设置的用户名
                        password: 'root12345'     //这里写入安装mysql时设置的密码
                ]
                sql = Sql.newInstance(mysqlDB.url, mysqlDB.user, mysqlDB.password, mysqlDB.driver)
                // 连接mysql数据库固定写法
            }
            sql
        }
    }
    

ConstantSql代码，包含增删该查 SQL

    
    
    class ConstantSql {
        static final getUserInfo="select * from user"
        static final getAddressInfoByUserName ="select a.username,a.age,b.address from user a,address b where a.id=b.userId and a.username=?"
        static final addUser = "insert into user (username,age,create_date) values (?,?,'1019-10-11:11:12:13')"
        static final getUser="select * from user where username=?"
        static final updateAge="update user set age=? where username=?"    //sql查询语句固定写法，需要传递参数的地方用？
    }
    

DataRepository代码，负责执行 SQL

    
    
    class DataRepository extends DataSource{
        def getUserInfo() {
            def userInfo = sql.rows(ConstantSql.getUserInfo)   //查询多行数据
            userInfo ? userInfo : ''  //这里做了空保护
        }
    
        def getAddressByUserName(userName) {
            def address = sql.firstRow(ConstantSql.getAddressInfoByUserName, [userName])  //只获取第一行数据
            // 上面调用的getAddressInfoByUserName sql语句需要传递一个参数，所以在后面带了userName参数
            address ? address : ''
        }
    
        def addUser(userName,age) {
            sql.execute(ConstantSql.addUser,[userName,age])   //传递两个参数
            // 非查询类操作使用execute方法
        }
    
        def getUser(userName) {
            sql.firstRow(ConstantSql.getUser,[userName])
        }
    
        def updateAddress(userName,age) {
            sql.execute(ConstantSql.updateAge,[age,userName])
        }
    }   
    

编写case检查是否能获取正确数据

    
    
    class Case extends Specification {
        DataRepository dataRepository
    
        void setup() {
            dataRepository = new DataRepository()
        }
    
        def "should get user info successfully"() {
            given: "no given"
            when: "query user table to get info"
            def userInfo = dataRepository.getUserInfo()
            then: "should get user info"
            userInfo.each { it -> println it.username + ":" + it.age + ":" + it.create_date }   //打印从数据库获取到的所有user信息
        }
    
        def "should get user address successfully"() {
            given: "no given"
            when: "query user and address table"
            def addressInfo = dataRepository.getAddressByUserName(userName)
            then: "should get correct user address info"
            Assert.assertEquals(addressInfo.address, address)  //校验从数据库获取的数据是否正确
            where:
            userName | address
            "TOM"    | "chengdu"
            "DONE"   | "beijing"
            "ECHO"   | "shanghai"
            "MARY"   | "hangzhou"
        }
    
        def "should add user successfully"() {
            given: "no given"
            when: "add user"
            dataRepository.addUser(userName, age)    //添加信息到user表
            then: "should get added user successfully"
            Assert.assertEquals(dataRepository.getUser(userName).username, userName)  //只有成功添加了“Dave”这个user，校验才会成功
            where:
            userName | age
            "Dave"   | 88
        }
    
        def "should update address successfully"() {
            given: "no given"
            when: "update user's address"
            dataRepository.updateAddress(userName, age)  //修改user表信息
            then: "should update address successfully"
            Assert.assertEquals(dataRepository.getUser(userName).age, age)   //校验修改后的user的age是否正确
            where:
            userName | age
            "MARY"   | 55
        }
    }
    

执行上述的case，全部执行成功如下图所示，说明正确从数据库获取到了数据。

![](https://images.gitbook.cn/15749193727221)

以上就是 Groovy 操作数据库的所有知识点，可以看到非常简单，配置好数据库连接信息后编写 SQL 增删该查语句并调用 Groovy 自带的
execute、firstRow、rows 等方法执行即可。接下来将给大家介绍下 Groovy 脚本。

## groovy脚本文件

以上讲解的内容中都是通过创建 Class 使用 Groovy
，实际上面所写的所有代码还可以放到`xxx.groovy`文件中，且可直接运行`xxx.groovy`，即通过执行脚本文件运行编写的代码。

测试工作中编写合理脚本可协助进行手动测试提升测试效率，例如一些测试数据的准备，测试结果的校验等。

要运行 Groovy 脚本需要单独安装 Groovy，[Groovy
安装方法](https://blog.csdn.net/accp_fangjian/article/details/51479505)。

安装好后创建一个 first.groovy 的文件，文件中写入`println "hello world"`，然后在命令行工具上运行`groovy
./test.groovy`，就可以打印出"hello world"，如下图所示。

![](https://images.gitbook.cn/15749193727234)

大家可以把前面写过的一些内容copy到 Groovy 脚本上尝试运行感受下，这里介绍 Groovy
脚本内容是期望大家能多掌握一些技能，这样在日常手动测试中，可以编写一些脚本辅助手动测试，提升测试效率。

另外如今越来越火的构建工具 Gradle 就是 Groovy 写的，掌握好 Groovy 脚本可以更好的使用Gradle。

至此 Groovy 操作数据库的内容就结束了，下节课会带领大家学习如何通过 Groovy 操作各种文件，然后重写 DataSource，将数据库连接信息放到
yaml 文件中统一管理。

