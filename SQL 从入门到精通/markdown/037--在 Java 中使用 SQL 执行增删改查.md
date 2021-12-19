上一篇我们学习了如何在 Python 应用中通过数据库接口执行 SQL 语句和存储过程。

今天我们来了解一下如何在 Java 程序中通过 JDBC 执行数据的增删改查操作。

### 什么是 JDBC？

JDBC（Java Database Connectivity）是 Java 语言中用于访问数据库的应用程序接口（API）。JDBC
提供了查询和更新关系数据库的标准方法，属于 Java Standard Edition 平台的一部分。以下是 Java 应用程序通过 JDBC
访问数据库的示意图：

![jdbc](https://img-blog.csdnimg.cn/20191102102848470.png?x-oss-
process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly90b255ZG9uZy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

其中各个模块的作用如下：

  * Java 应用程序由开发人员编码完成，用于实现业务处理的逻辑和流程；
  * JDBC API 提供了统一的接口和驱动管理，实现了应用程序和 JDBC 驱动的隔离。同一套应用代码只需要切换驱动程序就可以支持不同的数据库；
  * JDBC 驱动实现了 JDBC API 中定义的接口，用于与不同的数据库进行交互；
  * 数据库提供数据的存储管理和访问控制。

接下来我们就以 Java 程序连接 MySQL 数据库为例，介绍如何通过 JDBC 连接数据库并执行 SQL
语句。如果使用其他数据库，除了驱动不同之外，几乎不需要修改什么代码。

首先，让我们准备好开发环境。

### 安装开发环境

开发环境需要准备三部分内容：Java JDK、MySQL 数据库以及 MySQL JDBC 驱动。

#### 安装 Java JDK

开发 Java 应用需要使用 JDK（Java Development Kit）；使用 java -version 命令检查是否已经安装 JDK，要求至少
JDK 1.8.0 版本以上。

    
    
    C:\Users\dongx>java -version
    java version "13.0.1" 2019-10-15
    Java(TM) SE Runtime Environment (build 13.0.1+9)
    Java HotSpot(TM) 64-Bit Server VM (build 13.0.1+9, mixed mode, sharing)
    

如果显示无法识别以上命令，表示没有安装 JDK。打开浏览器，进入 Oracle 官方 Java SE JDK
[下载地址](https://www.oracle.com/technetwork/java/javase/downloads/index.html)。

![download](https://img-blog.csdnimg.cn/20191102135358919.png?x-oss-
process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly90b255ZG9uZy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

点击页面中的“DOWNLOAD”按钮，进入下载页面。点击同意用户协议后，选择下载相应平台的安装文件（Windows 平台为
jdk-13.0.1_windows-x64_bin.exe），完成后运行文件进行安装。

安装完成之后，还需要设置环境变量。

  * 右键单击“我的电脑”，然后选择“属性”。在“高级”选项卡上，选择“环境变量”，然后新建环境变量“JAVA_HOME”，变量值为 JDK 的安装目录，例如“C:\Program Files\Java\jdk-13.0.1”。
  * 编辑环境变量“Path”，将以下内容追加到最后：%JAVA_HOME%\bin\。

再次使用 java -version 命令查看 JDK 的版本。

#### 安装 MySQL 数据库

安装 MySQL 数据库的过程比较简单，可以参考[我的博客](https://mp.csdn.net/postedit/100089205) 。

#### 安装 MySQL Connector/J

为了方便程序开发，推荐安装一个 IDE（集成开发环境），我们使用 JetBrains 出品的 [IntelliJ
IDEA](https://www.jetbrains.com/idea/) 社区版。IntelliJ IDEA 可以使用 Maven
管理包的依赖，我们创建一个新的 Maven 项目（基于项目模板）：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191125153420648.png?x-oss-
process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly90b255ZG9uZy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

然后在项目的 pom.xml 文件的 <dependencies> 节点中添加以下内容：

    
    
        <dependency>
          <groupId>mysql</groupId>
          <artifactId>mysql-connector-java</artifactId>
          <version>8.0.18</version>
        </dependency>
    

Maven 就会自动下载 MySQL Connector/J 驱动并且进行配置。关于 IntelliJ IDEA 的使用可以参考 W3Cschool
上的[IntelliJ IDEA官方文档](https://www.w3cschool.cn/intellij_idea_doc/)。

### 案例：查询数据

通过 JDBC 连接数据库并执行 SQL 语句的过程主要包括以下内容：

  1. 利用 DriverManager 类的 getConnection() 方法获取一个 Connection 连接对象；
  2. 使用连接对象的 createStatement() 方法创建一个 Statement、PreparedStatement 或者 CallableStatement 语句对象；
  3. 利用语句对象的 executeQuery() 方法执行 SQL 语句或者存储过程，返回一个 ResultSet 结果集对象；
  4. 遍历结果集，获取并处理查询结果；
  5. 释放 ResultSet、Statement 以及 Connection 对象资源。

我们首先在项目目录中创建一个数据库的连接配置文件 db.properties，内容如下：

    
    
    user=tony
    password=Tony!123
    url=jdbc:mysql://192.168.56.104:3306/hrdb
    

其中 user 和 password 分别为连接数据库的用户和密码；url 中指定了数据库的 IP 地址、端口以及目标数据库。

然后将项目默认创建的 App.java 文件修改如下：

    
    
    package com.test;
    
    // 导入 JDBC 和 IO 包
    import java.io.FileInputStream;
    import java.io.IOException;
    import java.sql.*;
    import java.util.Properties;
    
    public class App
    {
        public static void main( String[] args )
        {
            String url = null;
            String user = null;
            String password = null;
            String sql_str = "SELECT emp_name, hire_date, salary " +
                             "  FROM employee " +
                             " WHERE dept_id = 2";
    
            // 读取数据库连接配置文件
            try (FileInputStream file = new FileInputStream("db.properties")) {
    
                Properties p = new Properties();
                pros.load(file);
                url = p.getProperty("url");
                user = p.getProperty("user");
                password = p.getProperty("password");
            } catch (IOException e) {
                System.out.println(e.getMessage());
            }
    
            // 建立数据库连接，创建查询语句，并且执行语句
            try (Connection conn = DriverManager.getConnection(url, user, password);
                 Statement stmt = conn.createStatement();
                 ResultSet rs = stmt.executeQuery(sql_str)) {
    
                // 处理查询结果集
                while (rs.next()) {
                    System.out.println(rs.getString("emp_name") + "\t" +
                            rs.getDate("hire_date")  + "\t" +
                            rs.getBigDecimal("salary"));
                }
    
            } catch (SQLException e) {
                System.out.println(e.getMessage());
            }
        }
    }
    

首先，通过一个 **FileInputStream** 对象读取数据库的连接配置文件；然后使用 **try -with-resources**
方式建立数据库连接，创建查询语句，并且执行该语句；最后使用一个 **while** 循环获取查询结果集， **rs.next()**
获取结果中的下一条记录； **rs.getString()** 、 **rs.getDate()** 和 **rs.getBigDecimal()**
分别用于获取记录中的字符串、日期以及数字字段。该程序执行的结果如下：

    
    
    诸葛亮        2006-03-15  24000.00
    黄忠        2008-10-25  8000.00
    魏延        2007-04-01  7500.00
    

### 案例：修改数据

接下来我们通过 JDBC 修改员工表中的信息，创建一个新的源文件 MySQLUpdate.java：

    
    
    package com.test;
    
    // 导入 JDBC 和 IO 包
    import java.io.FileInputStream;
    import java.io.IOException;
    import java.sql.*;
    import java.util.Properties;
    
    public class MySQLUpdate {
        public static void main(String[] args )
        {
            String url = null;
            String user = null;
            String password = null;
            String sql_str = "UPDATE employee " +
                             "   SET salary = ? " +
                             " WHERE emp_name = ?";
    
            // 读取数据库连接配置文件
            try (FileInputStream file = new FileInputStream("db.properties")) {
    
                Properties p = new Properties();
                p.load(file);
                url = p.getProperty("url");
                user = p.getProperty("user");
                password = p.getProperty("password");
            } catch (IOException e) {
                System.out.println(e.getMessage());
            }
    
            // 建立数据库连接，创建查询语句，并且执行语句
            try (Connection conn = DriverManager.getConnection(url, user, password);
                 PreparedStatement ps = conn.prepareStatement(sql_str)) {
    
                // 设置输入参数
                ps.setInt(1, 25000);
                ps.setString(2, "诸葛亮");
                ps.addBatch();
    
                ps.setInt(1, 8500);
                ps.setString(2, "黄忠");
                ps.addBatch();
    
                ps.setInt(1, 8000);
                ps.setString(2, "魏延");
                ps.addBatch();
    
                // 执行批量更新操作
                int[] rowCount = ps.executeBatch();
                System.out.println(String.format("更新行数: %d", rowCount.length));
    
            } catch (SQLException e) {
                System.out.println(e.getMessage());
            }
        }
    }
    

其中，sql_str 中的两个问号（ **?** ）表示两个占位符，它们的值会在运行时进行替换；然后使用 **prepareStatement()**
方法创建了一个 PreparedStatement 预编译的 SQL 语句，在执行该语句之前使用 **setInt()** 和
**setString()** 方法替换占位符的值，并且使用 **addBatch()** 添加批量操作；最后执行 **executeBatch()**
方法进行批量更新操作，返回被更新的行数。

> 预编译语句可以避免 SQL 语句的重复编译，使用不同的参数多次运行语句可以提高效率，并且能够预防 SQL 注入；我们将会在第 38 篇中讨论 SQL
> 注入的预防。

运行以上示例的输出结果如下：

    
    
    更新行数: 3
    

我们可以再次运行上文中的查询示例，确认修改后的结果。

### 案例：执行存储过程

除了直接执行 SQL 语句之外，也可以在利用 JDBC 中的 CallableStatement 对象执行 MySQL 存储过程。我们创建一个新的源码文件
使用上一篇中创建的存储过程 get_employee_by_name，通过姓名查找员工的信息：

    
    
    package com.test;
    
    // 导入 JDBC 和 IO 包
    import java.io.FileInputStream;
    import java.io.IOException;
    import java.sql.*;
    import java.util.Properties;
    
    public class MySQLProc {
        public static void main(String[] args )
        {
            String url = null;
            String user = null;
            String password = null;
            String sql_str = "{ call get_employee_by_name(?) }";
            ResultSet rs;
    
            // 读取数据库连接配置文件
            try (FileInputStream file = new FileInputStream("db.properties")) {
    
                Properties p = new Properties();
                p.load(file);
                url = p.getProperty("url");
                user = p.getProperty("user");
                password = p.getProperty("password");
            } catch (IOException e) {
                System.out.println(e.getMessage());
            }
    
            // 建立数据库连接，创建查询语句，并且执行语句
            try (Connection conn = DriverManager.getConnection(url, user, password);
                 CallableStatement cs = conn.prepareCall(sql_str)) {
    
                // 设置输入参数
                cs.setString(1, "诸葛亮");
    
                // 执行存储过程，遍历返回结果
                rs = cs.executeQuery();
                while (rs.next()) {
                    System.out.println(rs.getString("emp_name") + " "
                            + rs.getString("sex") + " "
                            + rs.getString("email"));
                }
    
            } catch (SQLException e) {
                System.out.println(e.getMessage());
            }
        }
    }
    

其中，调用存储过程使用 **{ call get_employee_by_name(?) }** ，问号是占位符，表示一个输入参数；
**prepareCall()** 方法返回一个 CallableStatement
对象，代表调用存储过程的语句；设置输入参数后执行存储过程并且处理返回结果。运行该示例输入以下内容：

    
    
    诸葛亮 男 zhugeliang@shuguo.com
    

除了我们介绍的内容之外，JDBC 还提供了许多功能，例如连接池、事务管理以及负载均衡等。关于 MySQL Connector/J 驱动的详细配置，可以参考
[MySQL 官方文档](https://dev.mysql.com/doc/connector-j/8.0/en/)。

在实际的应用开发中，我们不需要直接调用这些底层 JDBC 接口，而是可以利用成熟的框架，例如 Mybatis、Hibernate 和 Spring
JDBC。这些框架可以为我们处理所有的低层细节，包括连接管理、事务控制以及异常处理等；当然，这些框架最后还是调用了 JDBC 接口。

### 小结

JDBC 为我们提供了 Java 访问数据库的标准接口，本文通过几个简单的案例演示了 JDBC
操作数据库的流程。如果打算进一步学习，推荐选择一款框架深入了解。

**练习题** ：创建两个 Java 类 MySQLInsert.java 和 MySQLDelete.java，分别用于插入和删除员工信息。

