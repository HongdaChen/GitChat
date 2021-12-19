上一篇我们介绍了关系数据库和 SQL 语言的一些重要概念，同时准备好了课程所需的环境和初始化数据。现在就让我们正式开始 SQL 的学习！

查询是数据库中最常见的操作，所以我们先来了解一下基本的查询语句。

> **本专栏所有的 SQL 语句默认都适用于 4 种数据库，数据库专用的语法将会进行特殊说明。**

### 查询指定字段

在 employee 表中，存储了关于员工的信息。假设现在打算群发邮件，需要找出所有员工的姓名、性别和电子邮箱。在 SQL
中可以通过一个简单的查询语句来实现：

    
    
    SELECT emp_name, sex, email
      FROM employee;
    

其中 SELECT 表示查询，随后列出需要返回的字段，多个字段使用逗号分隔；FROM 表示要从哪个表中进行查询；分号表示 SQL
语句的结束。该语句执行的结果如下（显示部分数据）：

![avatar](https://images.gitbook.cn/FgxawwbVsrlW26OWanU2njATkYbu)

这种查询表中指定字段的操作在关系运算中被称为 **投影** （Projection），使用 SELECT
子句进行表示。投影是针对表进行的垂直选择，保留需要的字段用于生成新的表。以下是投影操作的示意图：

![avatar](https://images.gitbook.cn/Fp9kX4KvaMytcz1Q-W8kXrycKP9z)

投影操作中包含一个特殊的操作，就是查询表中所有的字段。

### 查询全部字段

查看表中的全部字段可以使用一个简单的写法，就是使用星号（*）表示全部字段。例如，以下语句查询员工表中的所有数据：

    
    
    SELECT * 
      FROM employee;
    

数据库在解析该语句时，会使用表中的字段名进行扩展：

    
    
    SELECT emp_id, emp_name, sex, dept_id, manager,
           hire_date, job_id, salary, bonus, email
      FROM employee;
    

该语句执行的结果如下（显示部分数据）：

![avatar](https://images.gitbook.cn/FmMXJzdq2vMsanToKJaRYN10gMxV)

> **注意**
> ：星号可以便于快速编写查询语句，但是在实际项目中不要使用这种写法。因为应用程序可能并不需要所有的字段，避免返回过多的无用数据；另外，当表结构发生变化时，星号返回的信息也会发生改变。

除了查询表的字段之外，SELECT 语句还支持扩展的投影操作，包括基于字段的算术运算、函数和表达式等。

### 扩展操作

以下示例返回了员工的姓名、一年的工资（12 个月的月薪）以及电子邮箱的大写形式：

    
    
    SELECT emp_name,
           salary * 12,
           UPPER(email)
      FROM employee;
    

其中 UPPER 是 SQL 中将字符串转换为大写的函数，函数将在第 8 篇中进行介绍。该语句的结果如下（显示部分数据）：

![avatar](https://images.gitbook.cn/Fp3ONxBuI7kTXY3j_7K0d68-AMD5)

在上面的结果中，返回字段的名称不是很好理解；能不能给它指定一个更明确的标题呢？这就需要使用到 SQL 中的别名（Alias）功能了。

### 使用别名

为了提高查询结果的可读性，可以使用别名为表或者字段指定一个临时的名称。SQL 中使用关键字 AS 指定别名。我们为上面的示例指定一些更好理解的标题：

    
    
    SELECT e.emp_name AS "姓名",
           salary * 12 AS "工资",
           UPPER(email) "电子邮箱"
      FROM employee AS e; -- Oracle 需要去掉此处的 AS
    

> 别名中的关键字 AS 可以省略。对于 Oracle 而言，表别名不支持 AS 关键字，省略掉即可。

首先，我们为 employee 表指定了一个表别名 e；然后为查询的结果字段指定了 3
个更明确的列别名（使用双引号）。在查询中为表指定别名之后，引用表中的字段时可以加上别名限定，例如
e.emp_name，表示要查看哪个表中的字段。以下是使用别名之后的效果：

![avatar](https://images.gitbook.cn/FjLBB0P6pAvy17fvtJATbsqjPT7-)

在 SQL 语句中使用别名不会修改数据库中存储的表名或者列名，别名只在当前语句中生效。

在上面的示例中，我们还用到了 SQL 中的另一个功能：注释。

### SQL 注释

在 SQL 中可以像其他编程语言一样使用注释；注释可以方便我们理解代码的作用，但不会被执行。

SQL中的注释分为单行注释和多行注释。单行注释以两个连字符（--）开始，直到这一行结束；上一节中的示例就使用了单行注释。SQL 使用 C
语言风格的多行注释（/* … */），例如：

    
    
    SELECT e.emp_name AS "姓名",
           salary * 12 AS "工资",
           UPPER(email) "电子邮箱"
    /* 备注：SQL 别名使用示例
       作者：TonyDong
       日期：2019-11-01
    */
      FROM employee AS e;
    

> MySQL中的 # 也可以用于表示单行注释。

在 SQL 中，SELECT … FROM … 是最基本的查询形式；但是，有时候我们会看到一种更简单的查询：只有 SELECT 子句，没有 FROM
子句的查询。

### 无表查询

以下查询没有 FROM 子句，用于计算一个表达式的值：

    
    
    -- MySQL、SQL Server 以及 PostgreSQL 实现
    SELECT 1+1;
    
    1+1|
    ---|
      2|
    

这种形式的查询语句通常用于快速查找信息，或者当作计算器使用。但是需要注意的是这种语法并不属于 SQL 标准，而是数据库产品自己的扩展。MySQL、SQL
Server 以及 PostgreSQL 都支持无表查询；对于 Oracle 而言，可以使用以下等价的形式：

    
    
    -- Oracle 实现
    SELECT 1+1
      FROM dual;
    
    1+1|
    ---|
      2|
    

dual 是 Oracle 中的一个特殊的表；它只有一个字段且只包含一行数据，就是为了方便快速查询信息。另外，MySQL 也提供了 dual 表。

### 小结

本篇我们学习了如何使用 SELECT 和 FROM 查询表中的数据，通过投影操作获取指定的字段信息。SQL
不仅仅能够查询表中的数据，还可以返回算术运算、函数和表达式的结果。在许多数据库中，不包含 FROM
子句的无表查询可以用于快速获取信息。另外，别名和注释都可以让我们编写的 SQL 语句更易阅读和理解。

**练习题** ：查询部门表（department）和职位表（job）中的数据，熟悉它们的字段结构和内容。

### 参考文献

  * [SQL 编程风格指南](https://www.sqlstyle.guide/zh/)。

### 分享交流

我们为本专栏付费读者创建了微信交流群，以方便更有针对性地讨论专栏相关的问题。入群方式请添加 GitChat
小助手伽利略的微信号：GitChatty6（或扫描以下二维码），然后给小助手发「245」消息，即可拉你进群～

![](https://images.gitbook.cn/FsONnMw_1O_6pkv-U-ji0U1injRm)

