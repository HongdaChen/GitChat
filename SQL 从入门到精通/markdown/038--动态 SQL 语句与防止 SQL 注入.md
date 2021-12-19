上一篇我们学习了如何在 Java 程序中通过 JDBC 接口连接和操作数据库。

今天我们来讨论一下 SQL 中的动态语句，以及由此带来的 SQL 注入问题。

### 什么是动态 SQL 语句？

在我们的专栏中，使用的基本都是静态 SQL 语句（Static
SQL），因为这些语句的内容在编译时就已经完全确定。与此相反，数据库还支持另一类语句：动态语句（Dynamic SQL）。

**动态 SQL 语句是指内容在运行时才最终确定的语句**
。这种语句可以用于构建更加灵活通用的查询；无论数据库自身的存储过程，还是应用程序的驱动（Python DB-API、Java JDBC
等），都提供了创建动态 SQL 的功能。

#### 数据库动态 SQL

在数据库服务器中执行动态 SQL 的方式如下：

  * MySQL 使用 **PREPARE** 、 **EXECUTE** 和 **DEALLOCATE PREPARE** 语句的组合运行预编译的动态 SQL 语句；
  * Oracle 使用 **EXECUTE IMMEDIATE** 语句或者 **DBMS_SQL** 包编写动态 SQL，后者支持的功能更加强大；
  * SQL Server 使用 **EXECUTE** 语句和存储过程 **sp_executesql** 运行动态 SQL；
  * PostgreSQL 支持与 MySQL 相同的方式运行动态 SQL，或者使用 PL/pgSQL 中的 **EXECUTE** 语句运行动态 SQL。

我们先来看一个 MySQL 中的动态 SQL 语句示例：

    
    
    -- MySQL
    SET @s = 'SELECT emp_name, sex, email FROM employee WHERE emp_id = ?';
    PREPARE ps FROM @s;
    SET @empid = 1;
    EXECUTE ps USING @empid;
    emp_name|sex|email            |
    --------|---|-----------------|
    刘备     |男 |liubei@shuguo.com|
    
    SET @empid = 2;
    EXECUTE ps USING @empid;
    emp_name|sex|email            |
    --------|---|-----------------|
    关羽     |男 |guanyu@shuguo.com|
    
    DEALLOCATE PREPARE ps;
    

首先使用 SET 语句定义了一个字符串变量，其中的问号（?）表示预编译语句中的占位符，这一点和 JDBC 类似；然后使用 PREPARE
创建一个预编译语句（ps）；接着定义了一个员工编号变量（@empid）并赋值；使用 EXECUTE 执行该语句，USING
表示运行之前使用参数值替换占位符，第一次运行输出了“刘备”的信息；接下来使用新的变量值再次运行该语句，返回了“关羽”的信息；最后使用 DEALLOCATE
PREPARE 释放该语句的资源。

> 预编译语句可以实现一次编译多次运行，减少 SQL 语句的解析时间。

对于 Oracle 而言，可以使用 EXECUTE IMMEDIATE 执行动态语句：

    
    
    -- Oracle
    DECLARE
      ln_empid INTEGER;
      lv_sql VARCHAR2(100) := 'SELECT emp_name, sex, email FROM employee WHERE emp_id = :empid';
      lv_empname VARCHAR2(50);
      lv_sex VARCHAR2(10);
      lv_email VARCHAR2(100);
    BEGIN
      ln_empid := 1;
      EXECUTE IMMEDIATE lv_sql INTO lv_empname,lv_sex,lv_email USING ln_empid;
      DBMS_OUTPUT.PUT_LINE(lv_empname || ',' || lv_sex || ',' || lv_email);
    
      ln_empid := 2;
      EXECUTE IMMEDIATE lv_sql INTO lv_empname,lv_sex,lv_email USING ln_empid;
      DBMS_OUTPUT.PUT_LINE(lv_empname || ',' || lv_sex || ',' || lv_email);
    END;
    

上面的语法和第 31 篇介绍的 PL/SQL 存储过程相同。DECLARE 声明了一些变量，lv_sql 中的 :empid
是占位符（绑定变量）；EXECUTE IMMEDIATE 执行动态语句，INTO 返回查询结果，USING 用于替换参数值；系统存储过程
DBMS_OUTPUT.PUT_LINE 用于打印输出信息；然后修改员工编号的值再次执行动态语句。以上程序的输出结果如下：

    
    
    刘备,男,liubei@shuguo.com
    关羽,男,guanyu@shuguo.com
    

SQL Server 中使用 EXECUTE 语句和存储过程 sp_executesql 运行动态语句：

    
    
    -- SQL Server
    DECLARE @empid INT
    DECLARE @s NVARCHAR(100)
    
    SET @s = N'SELECT emp_name, sex, email FROM employee WHERE emp_id = @emp_id'
    SET @empid = 1
    EXECUTE sp_executesql @s, N'@emp_id INT', @emp_id = @empid
    
    SET @empid = 2
    EXECUTE sp_executesql @s, N'@emp_id INT', @emp_id = @empid
    

首先，DECLARE 声明了一些变量；SET 为变量赋值中，其中 @emp _id 是变量占位符；EXECUTE sp_ executesql
执行动态语句，后面是变量的定义和赋值，使用逗号进行分隔。以上程序的输出结果如下：

    
    
    emp_name|sex|email            |
    --------|---|-----------------|
    刘备     |男 |liubei@shuguo.com|
    
    emp_name|sex|email            |
    --------|---|-----------------|
    关羽     |男 |guanyu@shuguo.com|
    

PostgreSQL 中可以使用与 MySQL 相同的方式执行预编译的语句：

    
    
    -- PostgreSQL
    PREPARE ps (int) AS
    SELECT emp_name, sex, email FROM employee WHERE emp_id = $1;
    
    EXECUTE ps(1);
     emp_name | sex |       email       
    ----------+-----+-------------------
     刘备     | 男  | liubei@shuguo.com
    (1 row)
    
    EXECUTE ps(2);
     emp_name | sex |       email       
    ----------+-----+-------------------
     关羽     | 男  | guanyu@shuguo.com
    (1 row)
    
    DEALLOCATE PREPARE ps;
    

首先使用 PREPARE 创建一个预编译语句（ps），接收一个 INT 类型的绑定参数，在语句中使用位置进行引用（$1）；然后使用 EXECUTE
传入参数并执行该语句，两次不同的参数调用分别返回了“刘备”和“关羽”的信息；最后使用 DEALLOCATE PREPARE 释放该语句的资源。

> 动态 SQL 语句不仅能够执行 SELECT，也可以支持其他 DML 和 DDL 等语句。

根据定义，我们在第 31 篇中介绍的存储过程和函数示例都属于动态语句。与此类似，应用程序驱动通常也是采用动态语句的方式执行数据库操作。

#### 应用程序动态 SQL

我们在第 36 篇和第 37 篇分别介绍了如何在 Python 应用和 Java 程序中执行 SQL 语句。以 Java 为例，分别可以利用
Statement、PreparedStatement 和 CallableStatement 语句对象执行普通语句、预编译语句和存储过程。对于
Statement 语句，可以先拼接出一个字符串对象：

    
    
    int deptId = 2;
    String sql_str = "SELECT emp_name, hire_date, salary " +
                             "  FROM employee " +
                             " WHERE dept_id = " + deptId;
    

然后执行语句的 executeQuery 方法运行查询：

    
    
    ResultSet rs = stmt.executeQuery(sql_str)
    

我们可以创建一个类的方法，将 deptId 作为参数传入，实现根据部门编号查询员工信息。除此之外，JDBC
预编译语句和存储过程调用也可以通过占位符的方式实现相同语句不同参数的重复运行。

动态 SQL 语句的最大优势就是它的灵活性，但是这种字符串拼接的方法可能带来一个严重的安全问题：SQL 注入（SQL Injection）。

### SQL 注入与预防

简而言之，SQL 注入都是利用了一个漏洞：应用程序未对客户端的输入进行验证，而是将其直接拼接到动态 SQL
语句中；从而获得数据库中未授权访问的数据，甚至恶意的数据破坏操作。

举例来说，假设我们有一个网站，用户登录时需要输入账号（userName）和密码（passWord）。当我们直接将用户的输入拼接成以下 SQL 语句时：

    
    
    String sql_str = "SELECT * FROM t_user WHERE user_name  = '"
                     + userName + "' AND password = '" + MD5(passWord) + "'";
    

如果用户输入的账号和密码是“admin”和“abc123”，那么数据库执行的语句就是：

    
    
    SELECT * FROM t_user WHERE user_name  = 'admin' AND password = 'e99a18c428cb38d5f260853678922e03';
    

只有用户名和密码正确时才会返回数据。但是如果有人故意输入“admin' OR '1' = '1' -- ”和“abc123”，那么数据库执行的语句就变成了：

    
    
    SELECT * FROM t_user WHERE user_name  = 'admin' OR '1' = '1' -- ' AND password = 'e99a18c428cb38d5f260853678922e03';
    

以上语句中的 OR 永远为 True，而且密码的验证被注释掉了，所以可以得到系统中的所有注册用户。

一些更恶意的行为是在输入数据里面加入删除数据或者表的破坏性操作，例如“admin' ; DROP table t_user --
”和“abc123”会导致以下操作：

    
    
    SELECT * FROM t_user WHERE user_name  = 'admin' ; DROP table t_user -- ' AND password = 'e99a18c428cb38d5f260853678922e03';
    

如果运行，其中的 DROP 语句会删除用户表 t_user。

知道了 SQL 注入的原理，也就不难对其进行预防了。SQL 注入的根本问题是将用户的输入数据和 SQL
语句中的命令混淆在一起，因此我们可以采用以下预防措施：

  * **使用参数化的预编译语句** ，这是预防 SQL 注入最有效的方式。数据库会对这些运行时的参数进行验证处理，不会将它们看作 SQL 中的操作部分；而且，预编译语句还能在一定程度上提高性能；
  * **对页面输入的内容进行校验** ，例如用户名和密码不能包含空白字符，或者使用正则表达式验证输入数据的模式等；
  * **使用 ORM（对象关系映射）框架** ，避免直接编写 SQL 语句；这些框架底层通常也是产生参数化的语句；
  * **严格管理数据库的账号和权限** ，例如应用程序连接数据库的账号只给相应的增删改查权限，即使账号泄漏也无法执行 DROP 这种严重破坏的操作。

我们还是以 MySQL 为例，比较一下不使用和使用参数化查询时的区别：

    
    
    SET @s = CONCAT('SELECT emp_name, sex, email FROM employee WHERE emp_id = ', '1 OR ''1''=''1''');
    PREPARE ps FROM @s;
    EXECUTE ps;
    emp_name|sex|email                   |
    --------|---|------------------------|
    刘备     |男 |liubei@shuguo.com       |
    关羽     |男 |guanyu@shuguo.com       |
    张飞     |男 |zhangfei@shuguo.com     |
    ...
    
    SET @s = 'SELECT emp_name, sex, email FROM employee WHERE emp_id = ?';
    PREPARE ps FROM @s;
    SET @empid = '1 OR ''1''=''1''';
    EXECUTE ps USING @empid;
    emp_name|sex|email            |
    --------|---|-----------------|
    刘备     |男 |liubei@shuguo.com|
    
    DEALLOCATE PREPARE ps;
    

第一个语句中直接拼接了所有的输入字符，由于查询条件中的 OR 永远为 True，返回了所有的员工信息；第二个语句使用了参数绑定，整个 empid
的内容都被看作数据，返回了“刘备”的信息（MySQL 将字符串中的第一个字符转换成数字 1）。

另外，如果使用存储过程或者在应用程序中使用参数化预编译语句也是同样的道理。

### 小结

动态 SQL
语句可以在运行时使用不同的参数生成不同的语句，方便我们编写更加通用灵活的数据处理模块。早期的时候由于使用不当导致了一些安全问题，但是如今已经有了很好的解决措施，我们要做的就是遵循这些标准的方法。

**参考文档** ：

  * [MySQL 官方文档](https://dev.mysql.com/doc/refman/8.0/en/sql-prepared-statements.html)；
  * [Oracle 官方文档](https://docs.oracle.com/en/database/oracle/oracle-database/18/lnpls/dynamic-sql.html#GUID-7E2F596F-9CA3-4DC8-8333-0C117962DB73)；
  * [SQL Server 官方文档](https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-executesql-transact-sql?view=sql-server-ver15)；
  * [PostgreSQL 官方文档](https://www.postgresql.org/docs/12/sql-prepare.html)。

