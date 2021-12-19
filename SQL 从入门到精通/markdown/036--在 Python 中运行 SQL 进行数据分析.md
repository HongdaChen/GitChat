上一篇我们学习了如何在关系数据库中存储 JSON 文档，以及使用 SQL 实现关系数据与 JSON 数据的相互转换。

今天我们介绍如何在 Python 程序中访问数据库，执行 SQL 语句进行数据分析操作。

### Python 数据库连接

Python 以其优雅、准确、 简单的语言特性，在云计算、Web 开发、自动化运维、数据科学以及机器学习等人工智能领域获得了广泛应用。如果想要学习
Python，可以参考菜鸟教程上的 [《Python 3
教程》](https://www.runoob.com/python3/python3-tutorial.html)。

为了连接数据库，Python 定义了操作数据库的标准接口 Python DB API。不同的数据库在此基础上实现了特定的驱动，这些驱动都实现了标准接口。

![pythondb](https://img-blog.csdnimg.cn/20191111094855971.png?x-oss-
process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly90b255ZG9uZy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

常用的 Python 驱动包括但不限于：

  * Python 连接 Oracle 数据库的 cx_Oracle；
  * Python 连接 MySQL 数据库的 MySQL Connector/Python；
  * Python 连接 SQL Server 数据库的 pyodbc；
  * Python 连接 PostgreSQL 数据库的 Psycopg。

接下来我们以 MySQL 为例，介绍如何在 Python 程序中使用 MySQL Connector/Python 驱动访问 MySQL
数据库。其他数据库的连接和操作也都大同小异。

### 安装 MySQL Connector/Python

首先，我们需要验证已经安装了 Python，在命令行中输入以下命令查看：

    
    
    C:\Users\dongx>python -V
    Python 3.8.0
    

我们使用的是 Python 3.8.0 版本。如果没有安装
Python，可以在[官方网站](https://www.python.org/downloads/)下载安装。同时可以安装一个
IDE（集成开发环境），例如 JetBrains 出品的 [Pycharm Community
Edition](https://www.jetbrains.com/pycharm/) 。

接下来我们需要安装 MySQL Connector/Python 驱动，在命令行中输入以下命令：

    
    
    C:\Users\dongx>pip install mysql-connector-python
    

执行该命令之后会自动安装 mysql-connector-python 驱动和相关依赖，并且输出以下信息：

    
    
    Successfully installed mysql-connector-python-8.0.18
    

表示驱动已经安装成功，然后我们就可以利用这个驱动连接 MySQL 数据库了。

### 连接 MySQL 数据库

首先，在 Pycharm 中新建一个项目。创建一个数据库连接的配置文件 dbconfig.ini，添加以下内容：

    
    
    [mysql]
    host = 192.168.56.104
    port = 3306
    database = hrdb
    user = tony
    password = Tony!123
    

该配置文件中存储了数据库的连接信息：主机、端口、数据库、用户以及密码。你需要按照自己的环境进行配置。

然后新建一个 Python 文件 mysql_connection.py，输入以下内容测试数据库连接：

    
    
    # 导入 MySQL Connector/Python 模块和 Error 对象
    import mysql.connector
    from mysql.connector import Error
    from configparser import ConfigParser
    
    def read_db_config(filename='dbconfig.ini', section='mysql'):
        """ 读取数据库配置文件，返回一个字典对象
        """
        # 创建解析器，读取配置文件
        parser = ConfigParser()
        parser.read(filename)
    
        # 获取 mysql 部分的配置
        db = {}
        if parser.has_section(section):
            items = parser.items(section)
            for item in items:
                db[item[0]] = item[1]
        else:
            raise Exception('文件 {1} 中未找到 {0} 配置信息！'.format(section, filename))
    
        return db
    
    db_config = read_db_config()
    connection = None
    
    try:
        # 使用 mysql.connector.connect 方法连接 MySQL 数据库
        connection = mysql.connector.connect(**db_config)
    
        # 验证连接是否成功，并且输出 MySQL 版本
        if connection.is_connected():
            db_version = connection.get_server_info()
            print("连接成功，MySQL 服务器版本：", db_version)
    
    except Error as e:
        print("连接 MySQL 失败：", e)
    finally:
        # 释放数据库连接
        if (connection.is_connected()):
            connection.close()
            print("MySQL 数据库连接已关闭。")
    

以上程序主要包括：

  * **import mysql.connector** ，导入MySQL Connector/Python 模块，该模块包含了连接和操作 MySQL 数据库的接口；
  * **from mysql.connector import Error** ，导入 Error 对象，用于显示与数据库操作相关的错误消息；
  * **def read_db_config** ，创建一个读取数据库连接配置的函数，利用 ConfigParser 对象读取文件；
  * **mysql.connector.connect()** ，调用该方法建立数据库连接并返回一个连接对象 connection；
  * **connection.is_connected()** ，使用 is_connected() 方法判断是否成功连接数据库；
  * **connection.get_server_info()** ，使用 get_server_info() 方法获取 MySQL 服务器的版本；
  * **connection.close()** ，关闭数据库连接。

如果数据库的连接信息正确，执行该程序将会输出以下信息：

    
    
    连接成功，MySQL 服务器版本： 8.0.18
    MySQL 数据库连接已关闭。
    

接下来我们就可以在 Python 程序中执行各种 SQL 查询和修改操作。

### 执行查询操作

在 Python 中执行 SQL 查询语句的流程大体如下：

  1. 使用 mysql.connector.connect() 方法连接 MySQL 数据库，返回一个连接对象；
  2. 使用连接对象的 cursor() 方法获取一个游标；
  3. 通过游标的 execute() 方法执行 SQL 语句；
  4. 利用游标的 fetchone()、fetchall() 或者 fetchmany() 方法获取查询结果；
  5. 在 Python 程序中处理返回的结果；
  6. 关闭游标和连接资源。

我们新建一个 Python 文件 mysql_query.py，输入以下内容：

    
    
    # 导入 MySQL Connector Python 模块和 Error 对象
    import mysql.connector
    from mysql.connector import Error
    from configparser import ConfigParser
    
    def read_db_config(filename='dbconfig.ini', section='mysql'):
        """ 读取数据库配置文件，返回一个字典对象
        """
        # 创建解析器，读取配置文件
        parser = ConfigParser()
        parser.read(filename)
    
        # 获取 mysql 部分的配置
        db = {}
        if parser.has_section(section):
            items = parser.items(section)
            for item in items:
                db[item[0]] = item[1]
        else:
            raise Exception('文件 {1} 中未找到 {0} 配置信息！'.format(section, filename))
    
        return db
    
    db_config = read_db_config()
    connection = None
    
    try:
        # 使用 mysql.connector.connect 方法连接 MySQL 数据库
        connection = mysql.connector.connect(**db_config)
    
        sql_str = "select * from department"
        cursor = connection.cursor()
        cursor.execute(sql_str)
        results = cursor.fetchall()
        print("查询结果记录总数：", cursor.rowcount)
    
        # 打印查询结果
        for dept in results:
            print("部门编号：", dept[0], "- 部门名称：", dept[1])
    except Error as e:
        print("读取部门信息失败：", e)
    finally:
        # 释放数据库连接
        if (connection.is_connected()):
            cursor.close()
            connection.close()
            print("MySQL 数据库连接已关闭。")
    

其中， **connection.cursor()** 方法用于获取一个游标对象，游标对象代表了查询的结果集；
**cursor.execute(sql_str)** 用于执行查询语句； **cursor.fetchall()** 方法获取了所有的查询结果，也可以使用
fetchone() 或者 fetchmany(n) 返回一条或者多条记录；然后通过一个 for 循环打印所有的结果。该程序输出的内容如下：

    
    
    查询结果记录总数： 6
    部门编号： 1 - 部门名称： 行政管理部
    部门编号： 2 - 部门名称： 人力资源部
    部门编号： 3 - 部门名称： 财务部
    部门编号： 4 - 部门名称： 研发部
    部门编号： 5 - 部门名称： 销售部
    部门编号： 6 - 部门名称： 保卫部
    MySQL 数据库连接已关闭。
    

除了查询语句之外，我们也可以通过接口执行数据的修改操作。

### 执行 DML 操作

在 Python 中执行 DML 语句的流程与查询流程类似，以下示例（mysql_insert.py）使用 INSERT 语句为员工表增加一个新员工：

    
    
    # 导入 MySQL Connector Python 模块和 Error 对象
    import mysql.connector
    from mysql.connector import Error
    from configparser import ConfigParser
    
    def read_db_config(filename='dbconfig.ini', section='mysql'):
        """ 读取数据库配置文件，返回一个字典对象
        """
        # 创建解析器，读取配置文件
        parser = ConfigParser()
        parser.read(filename)
    
        # 获取 mysql 部分的配置
        db = {}
        if parser.has_section(section):
            items = parser.items(section)
            for item in items:
                db[item[0]] = item[1]
        else:
            raise Exception('文件 {1} 中未找到 {0} 配置信息！'.format(section, filename))
    
        return db
    
    db_config = read_db_config()
    connection = None
    
    try:
        # 使用 mysql.connector.connect 方法连接 MySQL 数据库
        connection = mysql.connector.connect(**db_config)
    
        sql_str = """INSERT INTO employee(emp_id, emp_name, sex, dept_id, manager, hire_date, job_id, salary, bonus, email)
                     VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)"""
        employee = (26, '李四', '男', 5, 18, '2019-12-31', 10, 6000, None, 'lisi@shuguo.com')
        cursor = connection.cursor(prepared=True)
        cursor.execute(sql_str, employee)
        connection.commit()
        print("成功插入一条员工记录：")
    
        cursor.execute("SELECT emp_name, salary FROM employee WHERE emp_id = 26")
        result = cursor.fetchone()
    
        # 打印查询结果
        if result is not None:
            print(result)
    except Error as e:
        print("插入员工信息失败：", e)
    finally:
        # 释放数据库连接
        if (connection.is_connected()):
            cursor.close()
            connection.close()
            print("MySQL 数据库连接已关闭。")
    

其中，sql_str 中使用了许多占位符 %s，这些占位符的值在执行时进行替换；这种查询称为参数化查询（parameterized
query）。参数化查询可以进行预编译，也就是说数据库服务器可以将语句的分析编译结果存储在内存中，实现重复利用。connection.cursor(prepared=True)
表示创建一个预编译的语句。cursor.execute(sql_str, employee) 替换参数并且执行语句；connection.commit()
提交事务，因为默认的 auto-commit 连接参数为 False；最后再次查询并打印该员工的信息。

> 预编译语句（prepared statement）只需要编译一次，使用不同参数值运行多次时可以提高性能；同时它还可以防止 SQL 注入攻击，我们将会在第
> 38 篇中讨论 SQL 注入的预防。

该示例运行的结果如下：

    
    
    成功插入一条员工记录：
    ('李四', Decimal('6000.00'))
    MySQL 数据库连接已关闭。
    

UPDATE 和 DELETE 语句的使用方法与示例中的 INSERT 相同，需要修改的只是 SQL 语句。

### 调用 MySQL 存储过程

除了直接执行 SQL 语句之外，我们也可以在 Python 中利用游标对象的 **callproc()** 方法调用 MySQL
存储过程。首先创一个存储过程 get_employee_by_name，通过姓名查找员工的信息：

    
    
    DELIMITER $$
    
    CREATE PROCEDURE get_employee_by_name(IN p_emp_name VARCHAR(50))
    BEGIN
      SELECT emp_name, sex, email
        FROM employee
       WHERE emp_name = p_emp_name;
    END$$
    
    DELIMITER ;
    

然后创建一个新的模块 mysql_proc.python，调用该存储过程返回数据：

    
    
    # 导入 MySQL Connector Python 模块和 Error 对象
    import mysql.connector
    from mysql.connector import Error
    from configparser import ConfigParser
    
    def read_db_config(filename='dbconfig.ini', section='mysql'):
        """ 读取数据库配置文件，返回一个字典对象
        """
        # 创建解析器，读取配置文件
        parser = ConfigParser()
        parser.read(filename)
    
        # 获取 mysql 部分的配置
        db = {}
        if parser.has_section(section):
            items = parser.items(section)
            for item in items:
                db[item[0]] = item[1]
        else:
            raise Exception('文件 {1} 中未找到 {0} 配置信息！'.format(section, filename))
    
        return db
    
    db_config = read_db_config()
    connection = None
    
    try:
        # 使用 mysql.connector.connect 方法连接 MySQL 数据库
        connection = mysql.connector.connect(**db_config)
    
        cursor = connection.cursor(prepared=True)
        emp_name =  ['关羽']
        cursor.callproc('get_employee_by_name', emp_name)
        results = cursor.stored_results()
    
        # 打印查询结果
        for employee in cursor.stored_results():
            print(employee.fetchone())
    except Error as e:
        print("读取员工信息失败：", e)
    finally:
        # 释放数据库连接
        if (connection.is_connected()):
            cursor.close()
            connection.close()
            print("MySQL 数据库连接已关闭。")
    

其中，cursor.callproc() 方法用于调用存储过程，参数是存储过程名和定义时的参数。cursor.stored_results()
存储了执行结果对象；然后打印返回的结果：

    
    
    ('关羽', '男', 'guanyu@shuguo.com')
    MySQL 数据库连接已关闭。
    

Python 连接数据库还支持其他的选项，例如连接池（connection pool）和认证方式等。如果想要进一步深入了解相关内容，可以参考 [MySQL
Connector/Python 官方文档](https://dev.mysql.com/doc/connector-python/en/)。

### 小结

在 Python 程序可以通过标准的 DB API
接口和相应的驱动连接各种数据库，实现对数据库的操作和存取。除了本文介绍的方法之外，实际开发中我们也可以利用对象关系映射（ORM）工具简化数据库的操作，例如
SQLAlchemy 和 Django ORM。

**练习题** ：创建两个 Python 模块 mysql_update.py 和
mysql_delete.py，分别用于更新和删除员工信息，输入参数为员工姓名。

