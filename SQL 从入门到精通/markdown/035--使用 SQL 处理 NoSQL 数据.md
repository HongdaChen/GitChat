上一篇我们介绍了一些常见的 SQL 优化技巧，包括创建合适的索引和各种查询语句的优化。

今天我们来学习如何在关系数据库中存储 NoSQL 数据（JSON 文档），以及使用 SQL 实现数据行与 JSON 文档的相互转换。

### 什么是 JSON？

JSON（JavaScript Object Notation、JavaScript
对象表示法）是一种轻量级的数据交换格式，采用完全独立于编程语言的文本格式来存储和表示数据。JSON
易于阅读和编写，同时也方便机器解析和生成，并且能够有效地提升网络传输效率。在网络数据传输领域，JSON 已成为了 XML 强有力的替代者。

> 如果想要学习和了解 JSON，推荐 [JSON 官方网站](http://www.json.org/)和 W3Cschool 上的 [JSON
> 教程](https://www.w3cschool.cn/json/)。

除了 Web 领域之外，NoSQL 数据库也是 JSON 广泛应用的场景。例如，著名的文档数据库 MongoDB 就是采用 JSON
格式进行数据的存储。与此同时，SQL:2016 标准也增加了以下 JSON 功能：

  * **JSON 对象的存储与检索** ；
  * **将 JSON 对象表示成 SQL 数据** ；
  * **将 SQL 数据表示成 JSON 对象** 。

这些功能可以表示为以下示意图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191121112710263.png?x-oss-
process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly90b255ZG9uZy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

如今主流的关系数据库也都增加了 JSON 数据类型的支持，我们来看看如何在关系数据库中存储 JSON 文档。

### 创建 JSON 字段

首先，我们来创建一个新的员工表 employee_json，使用 JSON 字段存储员工的信息：

    
    
    -- MySQL 实现
    CREATE TABLE employee_json(
      emp_id    INTEGER NOT NULL PRIMARY KEY,
      emp_info  JSON NOT NULL
    );
    
    -- Oracle
    CREATE TABLE employee_json(
      emp_id    INTEGER NOT NULL PRIMARY KEY,
      emp_info  VARCHAR2(4000) NOT NULL CHECK (emp_info IS JSON)
    );
    
    -- SQL Server
    CREATE TABLE employee_json(
      emp_id    INTEGER NOT NULL PRIMARY KEY,
      emp_info  VARCHAR(MAX) NOT NULL CHECK ( ISJSON(emp_info)>0 )
    );
    
    -- PostgreSQL 实现
    CREATE TABLE employee_json(
      emp_id    INTEGER NOT NULL PRIMARY KEY,
      emp_info  JSONB NOT NULL
    );
    

其中：

  * MySQL 提供了原生的 JSON（二进制格式）数据类型，支持自动的格式校验；
  * Oracle 没有提供原生的 JSON 类型，使用 VARCHAR2 或者 CLOB 类型；同时利用 IS JSON 检查约束验证数据的格式；
  * SQL Server 没有提供原生的 JSON 类型，使用 VARCHAR 类型；同时利用 ISJSON 函数验证数据的格式；
  * PostgreSQL 提供了原生的 JSONB（二进制格式）数据类型，支持自动的格式校验；同时还提供了 JSON（文本格式）数据类型，但是支持的功能不如 JSONB 强大。

### 使用 SQL 插入 JSON 数据

使用 INSERT 语句可以将文本数据插入相应的 JSON 字段中，我们先插入一个不符合规范的 JSON 数据：

    
    
    INSERT INTO employee_json VALUES (1,'{"emp_name":  "刘备" ');
    
    -- Oracle
    SQL Error [2290] [23000]: ORA-02290: check constraint (TONY.SYS_C0016284) violated
    
    -- MySQL
    SQL Error [3140] [22001]: Data truncation: Invalid JSON text: "Missing a comma or '}' after an object member." at position 23 in value for column 'employee_json.emp_info'.
    
    -- SQL Server
    SQL Error [547] [23000]: The INSERT statement conflicted with the CHECK constraint "CK__employee___emp_i__1E3A7A34". The conflict occurred in database "hrdb", table "dbo.employee_json", column 'emp_info'.
    
    -- PostgreSQL
    SQL Error [22P02]: ERROR: invalid input syntax for type json
      Detail: The input string ended unexpectedly.
      Position: 37
      Where: JSON data, line 1: {"emp_name":  "刘备" 
    

由于数据库执行了 JSON 格式校验，以上不符合规范的数据（缺失一个大括号）无法插入成功。如果使用以下有效的 JSON 数据，可以成功插入：

    
    
    INSERT INTO employee_json 
    VALUES (1, '{"emp_name": "刘备", "sex": "男", "dept_id": 1, "manager": null, "hire_date": "2000-01-01", "job_id": 1, "income": [{"salary":30000}, {"bonus": 10000}], "email": "liubei@shuguo.com"}');
    

其中，income 节点是一个数组，包含了 salary 和 bonus 两个对象。完整的插入脚本可以点击 GitHub
[链接](https://github.com/dongxuyang1985/thinking_in_sql/blob/master/35/employee_json.sql)进行下载，最终该表的数据如下：

![employee_json](https://img-blog.csdnimg.cn/2019111916292111.png?x-oss-
process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly90b255ZG9uZy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

接下来我们介绍如何使用 SQL 语句操作 JSON 数据。

### 使用 SQL 查询 JSON 数据

JSON 字段的查询与普通字段相同，我们主要介绍如何查询 JSON 内部的元素。SQL 标准使用 **JSON_VALUE** 函数查询 JSON
元素的值，使用 **JSON_QUERY** 函数查询元素中的对象和数组。以下语句从 emp_info 字段中获取员工的姓名和月薪：

    
    
    -- Oracle 和 SQL Server 实现
    SELECT emp_id,
           JSON_VALUE(emp_info, '$.emp_name') emp_name,
           JSON_VALUE(emp_info, '$.income[0].salary') salary,
           JSON_VALUE(JSON_QUERY(emp_info, '$.income[0]'),'$.salary') salary
      FROM employee_json
     WHERE JSON_VALUE(emp_info, '$.emp_name') = '刘备';
    
    EMP_ID|EMP_NAME|SALARY|SALARY|
    ------|--------|------|------|
         1|刘备     |30000 |30000 |
    

该查询返回了“刘备”的信息；JSON_VALUE 和 JSON_QUERY 函数使用 SQL/JSON 路径表达式查找数据中的元素，其中：

  * $ 代表整个文档；
  * $.emp_name 表示获取 JSON 对象的 emp_name 元素；
  * $.income[0].salary 表示获取 income 数组中的第一个对象的 salary 元素，数组的下标从 0 开始；
  * JSON_QUERY 函数使用 $.income[0] 返回了一个 JSON 对象，然后使用 JSON_VALUE 函数返回该对象的 salary 元素。

MySQL 使用 -> 和 ->> 运算符获取 JSON 文档中的元素值：

    
    
    -- MySQL 实现
    SELECT emp_id,
           emp_info->'$.emp_name' emp_name,
           emp_info->>'$.income[0].salary' salary
      FROM employee_json
     WHERE emp_info->>'$.emp_name' = '刘备';
    
    emp_id|emp_name|salary|
    ------|--------|------|
         1|"刘备"   |30000 |
    

其中，-> 返回的类型是 JSON ；->> 返回的类型是字符串；这两个操作符同样使用 SQL/JSON 路径表达式查找数据中的元素值。

PostgreSQL 使用 ->、#> 以及 ->>、#>> 运算符获取 JSON 文档中的元素值：

    
    
    -- PostgreSQL 实现
    SELECT emp_id,
           emp_info->'emp_name' emp_name,
           emp_info->>'emp_name' emp_name,
           emp_info#>'{income, 0, salary}' salary,
           emp_info#>>'{income, 0, salary}' salary
      FROM employee_json
     WHERE emp_info->>'emp_name' = '刘备';
    
    emp_id|emp_name|emp_name|salary|salary|
    ------|--------|--------|------|------|
         1|"刘备"   |刘备     |30000 |30000 |
    

其中，-> 和 #> 返回的类型是 JSONB；->> 和 #>> 返回的类型是字符串；#> 和 #>> 操作符支持多层路径，使用逗号分隔。

#### 将 JSON 数据转换为 SQL 数据

**JSON_TABLE** 函数可以将 JSON 格式的数据转换为 SQL 行数据，以下语句将 employee_json 表中的数据转换为行格式：

    
    
    -- Oracle 和 MySQL 实现
    SELECT emp_id, jt.*
      FROM employee_json,
           JSON_TABLE(emp_info, '$'
             COLUMNS (emp_name  VARCHAR(50) PATH '$.emp_name',
                      sex       VARCHAR(10) PATH '$.sex',
                      dept_id   INTEGER PATH '$.dept_id',
                      manager   INTEGER PATH '$.manager',
                      hire_date DATE PATH '$.hire_date',
                      job_id    INTEGER PATH '$.job_id',
                      salary    INTEGER PATH '$.income[0].salary',
                      bonus     INTEGER PATH '$.income[1].bonus',
                      email     VARCHAR(100) PATH '$.email')
           ) jt;
    

其中，$ 表示将整个 emp_info 作为数据行的来源；COLUMNS 定义了字段类型及其数据的来源，PATH 同样使用 SQL/JSON
路径表达式。该语句的结果如下（显示部分内容）：

![employee](https://img-blog.csdnimg.cn/20191120145320414.png?x-oss-
process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly90b255ZG9uZy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

SQL Server 和 PostgreSQL 目前没有提供该函数，可以利用上文中的 JSON_VALUE 和 JSON_QUERY 函数或者 -> 和
->> 运算符获取 employee_json 中的 JSON 节点数据。

#### 将 SQL 数据转换为 JSON 数据

**JSON_OBJECT** 和 **JSON_ARRAY** 函数可以将表中的数据构造成 JSON 对象和数组。以下语句使用员工表中的部分字段构造一个
JSON 对象：

    
    
    -- Oracle 实现
    SELECT JSON_OBJECT('emp_name' VALUE emp_name,
                       'sex' VALUE '男',
                       'income' VALUE JSON_ARRAY(JSON_OBJECT('salary' VALUE 30000), JSON_OBJECT('bonus' VALUE 10000))
                      ) jo
      FROM employee
     WHERE emp_id <= 3;
    JO                                                                     |
    -----------------------------------------------------------------------|
    {"emp_name":"刘备","sex":"男","income":[{"salary":30000},{"bonus":10000}]}|
    {"emp_name":"关羽","sex":"男","income":[{"salary":30000},{"bonus":10000}]}|
    {"emp_name":"张飞","sex":"男","income":[{"salary":30000},{"bonus":10000}]}|
    
    -- MySQL 实现
    SELECT JSON_OBJECT('emp_name', emp_name,
                       'sex', sex,
                       'income', JSON_ARRAY(JSON_OBJECT('salary', salary), JSON_OBJECT('bonus', bonus))
                      ) AS jo
      FROM employee
     WHERE emp_id <= 3;
    jo                                                                                   |
    -------------------------------------------------------------------------------------|
    {"sex": "男", "income": [{"salary": 30000.00}, {"bonus": 10000.00}], "emp_name": "刘备"}|
    {"sex": "男", "income": [{"salary": 26000.00}, {"bonus": 10000.00}], "emp_name": "关羽"}|
    {"sex": "男", "income": [{"salary": 24000.00}, {"bonus": 10000.00}], "emp_name": "张飞"}|
    

Oracle 和 MySQL 都提供了这两个函数，但是输入参数的方式不同。PostgreSQL 使用 **jsonb_build_object** 和
**jsonb_build_array** 函数构造 JSON 对象和数组：

    
    
    -- PostgreSQL 实现
    SELECT jsonb_build_object('emp_name', emp_name,
                             'sex', sex,
                             'income', jsonb_build_array(jsonb_build_object('salary', salary),jsonb_build_object('bonus', bonus))
                            ) AS jo
      FROM employee
     WHERE emp_id <= 3;
    jo                                                                                   |
    -------------------------------------------------------------------------------------|
    {"sex": "男", "income": [{"salary": 30000.00}, {"bonus": 10000.00}], "emp_name": "刘备"}|
    {"sex": "男", "income": [{"salary": 26000.00}, {"bonus": 10000.00}], "emp_name": "关羽"}|
    {"sex": "男", "income": [{"salary": 24000.00}, {"bonus": 10000.00}], "emp_name": "张飞"}|
    

SQL Server 提供了一个将表转换为 JSON 数组的查询选项 **FOR JSON** ：

    
    
    SELECT emp_name, sex, salary AS "income.salary", bonus AS "income.bonus"
      FROM employee
     WHERE emp_id <= 3
       FOR JSON PATH;
    [{"emp_name":"刘备","sex":"男","income":{"salary":30000.00,"bonus":10000.00}},{"emp_name":"关羽","sex":"男","income":{"salary":26000.00,"bonus":10000.00}},{"emp_name":"张飞","sex":"男","income":{"salary":24000.00,"bonus":10000.00}}]
    

该选项将所有的结果合并成一个 JSON 数组；其中，FOR JSON PATH 用于构造嵌套的对象。

### 使用 SQL 对 JSON 数据进行 DML 操作

如果需要更新表中的 JSON 字段，可以直接使用 UPDATE 语句。

另外，MySQL 提供了几个用于修改 JSON 元素的函数。以下语句为表 employee_json 中的“刘备”增加 10% 的月薪：

    
    
    -- MySQL 实现
    UPDATE employee_json
       SET emp_info = JSON_SET(emp_info, '$.income[0].salary', emp_info->>'$.income[0].salary' *1.1)
     WHERE emp_id = 1;
    
    SELECT emp_info->'$.income'
      FROM employee_json
     WHERE emp_id = 1;
    emp_info->'$.income'                   |
    ---------------------------------------|
    [{"salary": 33000.0}, {"bonus": 10000}]|
    

其中， **JSON_SET** 函数用于往 JSON 元素中插入数据；如果该元素存在则进行更新。另外，MySQL 还提供了
JSON_INSERT、JSON_REPLACE 和 JSON_REMOVE 函数，分别用于插入、替换和删除 JSON 元素。

SQL Server 提供了修改 JSON 元素的 **JSON_MODIFY** 函数。以下语句为表 employee_json 中的“刘备”增加 10%
的月薪：

    
    
    -- SQL Server 实现
    UPDATE employee_json
       SET emp_info = JSON_MODIFY(emp_info, '$.income[0].salary', CAST(JSON_VALUE(emp_info, '$.income[0].salary') AS NUMERIC) *1.1)
     WHERE emp_id = 1;
    
    SELECT JSON_QUERY(emp_info, '$.income') AS income
      FROM employee_json
     WHERE emp_id = 1;
    income                                |
    --------------------------------------|
    [{"salary":33000.0}, {"bonus": 10000}]|
    

对于 JSON_MODIFY 函数，如果修改的节点不存在，就会增加相应的节点；如果将节点设置为 NULL，就会删除相应的节点。

PostgreSQL 提供了修改 JSON 元素的 **jsonb_set** 函数。以下语句为表 employee_json 中的“刘备”增加 10%
的月薪：

    
    
    -- PostgreSQL 实现
    UPDATE employee_json
       SET emp_info = jsonb_set(emp_info, '{income,0,salary}', to_jsonb((emp_info#>>'{income, 0, salary}')::numeric*1.1))
     WHERE emp_id = 1;
    
    SELECT emp_info->'income' AS income
      FROM employee_json
     WHERE emp_id = 1;
    income                                 |
    ---------------------------------------|
    [{"salary": 33000.0}, {"bonus": 10000}]|
    

jsonb_set 函数设置的对象必须是 JSONB 类型，所以我们使用 to_jsonb 函数进行类型转换。另外，PostgreSQL 还提供了
jsonb_insert 函数用于插入 JSONB 元素。

最后，我们汇总一下各种数据库对于 JSON 数据的支持：

| Oracle | MySQL | SQL Server | PostgreSQL  
---|---|---|---|---  
JSON 数据类型 | VARCHAR2  
CLOB | JSON | VARCHAR | JSONB  
JSON  
构造 JSON 数据 | 字符串常量  
JSON_OBJECT  
JSON_ARRAY | 字符串常量  
JSON_OBJECT  
JSON_ARRAY | 字符串常量  
FOR JSON | 字符串常量  
jsonb_build_object  
jsonb_build_array  
查询指定元素 | JSON_QUERY  
JSON_VALUE | -> 或者 JSON_EXTRACT  
->> 或者 JSON_UNQUOTE + JSON_EXTRACT | JSON_QUERY   
JSON_VALUE | ->、#> 或者 jsonb_extract_path  
->>、#>> 或者 jsonb_extract_path_text  
JSON 数据转换为 SQL 数据 | JSON_TABLE | JSON_TABLE | OPENJSON |  
SQL 数据转换为 JSON 数据 | SON_OBJECT  
JSON_ARRAY | SON_OBJECT  
JSON_ARRAY | FOR JSON | jsonb_build_object  
jsonb_build_array  
更新 JSON 数据 | 更新整个文档 | JSON_SET  
JSON_INSERT  
JSON_REPLACE  
JSON_REMOVE | JSON_MODIFY | jsonb_set  
jsonb_insert  
格式验证 | IS [NOT] JSON | JSON_VALID | ISJSON | N/A  
  
还有一些我们没有介绍到的 JSON 函数和功能，可以参考以下 JSON 文档：

  * [Oracle 官方文档](https://docs.oracle.com/en/database/oracle/oracle-database/18/adjsn/index.html)；
  * [MySQL 官方文档](https://dev.mysql.com/doc/refman/8.0/en/json.html)；
  * [SQL Server 官方文档](https://docs.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server?view=sql-server-2017)；
  * [PostgreSQL 官方文档](https://www.postgresql.org/docs/current/datatype-json.html)。

### 小结

关系数据库对于 JSON 数据类型的支持可以方便我们将 SQL 的强大功能与 NoSQL
的灵活性相结合；当我们需要为应用增加文档数据支持的时候，除了使用专门的 NoSQL 数据库之外，也可以考虑直接在现有的数据库中使用 JSON 数据类型。

