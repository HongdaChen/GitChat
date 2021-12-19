上一篇我们学习了 SQL 中常见的日期时间函数和类型转换函数，熟练使用各种函数可以让我们的数据处理和分析工作事半功倍。

本篇我们介绍一种为 SQL 语句增加逻辑处理功能的方法：CASE 表达式。

**SQL 中的 CASE 表达式可以根据不同条件产生不同的结果，实现类似于编程语言中的 IF-THEN-ELSE 逻辑功能** 。例如，根据员工的 KPI
计算相应的涨薪幅度，根据考试成绩评出优秀、良好、及格等。

CASE 表达式支持两种形式： **简单 CASE 表达式** 和 **搜索 CASE 表达式** 。

### 简单 CASE 表达式

简单 CASE 表达式的语法如下：

    
    
    CASE expression
      WHEN value1 THEN result1
      WHEN value2 THEN result2
      ...
      [ELSE default_result]
    END
    

表达式的计算过程如下图所示：

![simple case](https://img-blog.csdnimg.cn/20190724163844995.png)

首先计算 expression 的值；然后依次与 WHEN
列表中的值（value1，value2，…）进行比较，找到第一个相等的值并返回对应的结果（result1，result2，…）；如果没有找到相等的值，返回
ELSE 中的默认结果；如果此时没有指定 ELSE，返回 NULL 值。

以下示例使用简单 CASE 表达式将员工的部门编号显示为相应的名称：

    
    
    SELECT emp_name,
           CASE dept_id
             WHEN 1 THEN '行政管理部'
             WHEN 2 THEN '人力资源部'
             WHEN 3 THEN '财务部'
             WHEN 4 THEN '研发部'
             WHEN 5 THEN '销售部'
             WHEN 6 THEN '保卫部'
             ELSE '其他部门'
           END AS department
      FROM employee;
    

首先判断部门编号是否等于 1，等于就显示为“行政管理部”；否则，如果部门编号等于 2， 显示为“人力资源部”；依次类推；如果部门编号不等于 1 到 6
中的任何值，显示为“其他部门”。该查询的结果如下（显示部分内容）：

![simple case](https://img-blog.csdnimg.cn/20190724164917170.png)

**CASE 表达式的一个常见应用就是实现表的行列转换** 。假设存在以下学生成绩表：

    
    
    -- 创建成绩表 t_case，sname 为学生姓名，cname 为课程名称，score 为考试成绩
    CREATE TABLE t_case(sname varchar(10), cname varchar(10), score int);
    
    -- 插入测试数据
    INSERT INTO t_case(sname, cname, score) VALUES ('张三', '语文', 80);
    INSERT INTO t_case(sname, cname, score) VALUES ('李四', '语文', 77);
    INSERT INTO t_case(sname, cname, score) VALUES ('王五', '语文', 91);
    INSERT INTO t_case(sname, cname, score) VALUES ('张三', '数学', 85);
    INSERT INTO t_case(sname, cname, score) VALUES ('李四', '数学', 90);
    INSERT INTO t_case(sname, cname, score) VALUES ('王五', '数学', 60);
    INSERT INTO t_case(sname, cname, score) VALUES ('张三', '英语', 81);
    INSERT INTO t_case(sname, cname, score) VALUES ('李四', '英语', 69);
    INSERT INTO t_case(sname, cname, score) VALUES ('王五', '英语', 82);
    

执行以上语句创建 t_case 表并且插入数据。该表中的数据如下：

![student](https://img-blog.csdnimg.cn/20190724193652245.png)

每个学生的每科成绩都是一行数据。我们利用 CASE 表达式将其转换为按列显示的形式，最终的结果如下：

![student](https://img-blog.csdnimg.cn/20190724193935532.png)

首先，执行以下查询语句：

    
    
    SELECT sname,
           CASE cname WHEN '语文' THEN score ELSE 0 END AS "语文",
           CASE cname WHEN '数学' THEN score ELSE 0 END AS "数学",
           CASE cname WHEN '英语' THEN score ELSE 0 END AS "英语"
      FROM t_case;
    

第一个 CASE 表达式用于获取学生的语文成绩，cname 等于“语文”就返回考试成绩，不是“语文”就记为 0 分。第二个和第三个 CASE
表达式分别用于获取数学和英语成绩，原理和第一个 CASE 表达式相同。该语句执行的结果如下：

![student](https://img-blog.csdnimg.cn/20190724194827827.png)

目前的结果还是 9 条记录。然后将每个学生的成绩合并成一条记录，此时需要使用到分组汇总的操作：

    
    
    SELECT sname,
           SUM(CASE cname WHEN '语文' THEN score ELSE 0 END) AS "语文",
           SUM(CASE cname WHEN '数学' THEN score ELSE 0 END) AS "数学",
           SUM(CASE cname WHEN '英语' THEN score ELSE 0 END) AS "英语"
      FROM t_case
     GROUP BY sname;
    

在这里我们使用了 SUM 汇总函数和 GROUP BY 分组操作，这些内容是接下来两篇的主题。在此简单说明一下，GROUP BY
按照学生进行分组，这样每个学生最终只有一条记录；同时将学生每门课程的成绩使用 SUM 函数进行求和（每科成绩加上两个
0），还是得到每科的成绩。这样就实现了数据的行转列。

> 学习完接下来两篇关于聚合函数和分组操作的内容之后，就能更容易地理解上面的示例了。

简单 CASE
表达式在进行判断的时候，使用的是等值比较（=），只能处理简单的逻辑。如果想要进行复杂的逻辑处理，例如根据考试成绩评出优秀、良好、及格等，就需要使用更加强大的搜索
CASE 表达式。

### 搜索 CASE 表达式

搜索 CASE 表达式的语法如下：

    
    
    CASE
      WHEN condition1 THEN result1
      WHEN condition2 THEN result2
      ...
      [ELSE default_result]
    END
    

表达式的计算过程如下图所示：

![search case](https://img-blog.csdnimg.cn/20190724165617307.png)
按照顺序依次计算每个分支中的条件（condition1，condition2，…），找到第一个结果为真的分支并返回相应的结果（result1，result2，…）；如果没有任何条件为真，返回
ELSE 中的默认结果；如果此时没有指定 ELSE，返回 NULL 值。

**所有的简单 CASE 表达式都可以替换为等价的搜索 CASE 表达式** 。我们可以将上一节的示例改写如下：

    
    
    SELECT emp_name,
           CASE 
             WHEN dept_id = 1 THEN '行政管理部'
             WHEN dept_id = 2 THEN '人力资源部'
             WHEN dept_id = 3 THEN '财务部'
             WHEN dept_id = 4 THEN '研发部'
             WHEN dept_id = 5 THEN '销售部'
             WHEN dept_id = 6 THEN '保卫部'
             ELSE '其他部门'
           END AS department
      FROM employee;
    

首先判断部门编号等于 1 是否成立（为真），成立就显示为“行政管理部”；否则，判断部门编号等于 2 是否成立，
成立就显示为“人力资源部”；依次类推；如果部门编号不等于 1 到 6 中的任何值，显示为“其他部门”。查询结果参见上文中的简单 CASE 表达式示例。

搜索 CASE 表达式通常用于处理更加复杂的逻辑条件，以下示例基于员工的月薪将他们的收入分为“高”、“中”、“低”三个等级：

    
    
    SELECT emp_name, salary,
           CASE 
             WHEN salary < 10000  THEN '低收入'
             WHEN salary < 20000  THEN '中收入'
             ELSE '高收入'
           END AS grade
      FROM employee;
    

如果月薪小于 10000，显示为“低收入”；否则，如果月薪小于 20000（此时肯定大于等于 10000），显示为“中收入”；否则，月薪大于等于
20000，显示为“高收入”。该查询的结果如下（显示部分内容）：

![searc example](https://img-blog.csdnimg.cn/20190724173335520.png)

**CASE 表达式除了可以用于查询语句的 SELECT 列表，也可以出现在其他子句中，例如 WHERE、ORDER BY 等** 。在[第 6
篇](https://gitbook.cn/gitchat/column/5dae96ec669f843a1a4aed95/topic/5db7cd1af6a6211cb96197b1)中，为了解决空值在不同数据库中排序不同的问题，我们使用了
COALESCE 函数。以下示例使用 CASE 表达式实现了相同的效果：

    
    
    SELECT emp_name,
           CASE
             WHEN bonus IS NULL THEN 0
             ELSE bonus
             END AS bonus
      FROM employee
     WHERE dept_id = 2
     ORDER BY CASE
                WHEN bonus IS NULL THEN 0
                ELSE bonus
              END;
    

ORDER BY 中的 CASE 表达式将奖金为空的数据转换为 0，确保空值在升序时排在最前；其他数据保持不变。该查询的结果如下：

![order](https://img-blog.csdnimg.cn/20191028152325643.png)

CASE 表达式是标准的 SQL 功能，所有数据库都支持并且实现一致。除此之外，Oracle 还提供了一个专有函数：DECODE。

### DECODE 函数

Oracle 中的 DECODE 函数可以实现类似于简单 CASE 表达式的功能：

    
    
    DECODE(expression, value1, result1, value2, result2, ...[, default_result ])
    

该函数依次比较表达式 expression 与 valueN 的值，如果找到相等的值就返回对应的 resultN；如果没有匹配到任何相等的值，返回默认结果
default _result；如果此时没有提供 default_ result，返回 NULL 值。

以下语句利用 DECODE 函数将员工的部门编号显示为相应的名称：

    
    
    -- Oracle 实现
    SELECT emp_name,
           DECODE(dept_id, 1, '行政管理部',
                           2, '人力资源部',
                           3 ,'财务部',
                           4, '研发部',
                           5, '销售部',
                           6, '保卫部',
                              '其他部门') AS department
      FROM employee;
    

该查询的结果与前文中的简单 CASE 表达式示例相同。DECODE 是 Oracle 专有函数，推荐大家使用标准的 CASE 表达式。

> MySQL 中的 DECODE 函数是一个解密函数，与此无关。

### 小结

CASE 表达式为 SQL 语句提供了逻辑处理的能力，可以基于不同的条件返回不同的结果。CASE 表达式支持两种形式：简单 CASE 表达式和搜索 CASE
表达式。

**练习题** ：编写 SQL 语句查询员工信息，按照部门编号进行排序，并且确保同一个部门中的女性员工排在男性员工之前。

