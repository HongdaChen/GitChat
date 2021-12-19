说起 Hive 大家首先自然会想到的就是 SQL ，所以现场手写 SQL 基本上也是面试的一个保留环节，这时候千万不要乱了阵脚，只要掌握 SQL
的一些常用的高阶语法，这些难题基本上都能迎刃而解，所以本章归纳总结了一些比较常用的 SQL 语法相关的面试题。另外，光会写 SQL
还是不够的，这还只能算是基本技能，要知道 Hive 的调优才能拿到高阶工程师的入场资格。

**本篇面试内容划重点：Hive 调优、窗口函数、列转行。**

### 性能问题和优化方案

造成 Hive 出现性能问题最常见的原因就是数据倾斜，数据倾斜是指计算过程中，数据分布不均匀，造成大量数据都聚集在少数的服务器上，造成了数据热点。

**可能造成数据倾斜的场景：**

  * 关联（`t1 join t2 on t1.f1=t2.f1`）：关联场景的关联字段 f1 的值分布不均匀（或者 NULL/0 这种值太多），会造成数据倾斜问题产生，大量数据被分发到少数几个 Reduce 处理。
  * 分组（`group by f1`）：同样 f1 值的分布不均匀会造成数据倾斜。
  * 去重（`count distinct f1`）：distinct 操作是全局去重，最后会在一个 reduce 中处理结果，同样会造成数据倾斜。可以替换为 `select sum(tt1.num) from (select 1 as num,f1 from t1 group by f1 ) tt`。

#### **加大并行度**

通过设置参数 hive.exec.parallel 值为 true，就可以开启并发执行。不过，在共享集群中，需要注意下，如果 job
中并行阶段增多，那么集群利用率就会增加。

打开任务并行执行：

    
    
    set hive.exec.parallel=true;
    

同一个 SQL 允许最大并行度，默认为 8：

    
    
    set hive.exec.parallel.thread.number=16;
    

如果数据倾斜严重可以考虑直接加大 Reduce 的并行度，缓解 Reduce 端的压力，但是要根据数据量来判断 reduce 的并行度，过多的启动和初始化
reduce 也会消耗时间和资源。

每个 Reduce 处理的数据默认是 256MB：

    
    
    SET hive.exec.reducers.bytes.per.reducer=256000000；
    

每个任务最大的 Reduce 数，默认为 1009：

    
    
    SET hive.exec.reducers.max=1009
    

Reduce 的数量默认 -1，Hive 自动确定 Reduce 数量：

    
    
    set mapreduce.job.reduces=-1
    

#### **MapJoin**

将小表放入内存中，在 Map 端进行 Join 操作，避免数据分发到 Reduce 后发生数据倾斜。

配置表在小表阈值范围内，是否自动启动 MapJoin：

    
    
    set hive.auto.convert.join = true;
    

配置小表的阈值：

    
    
    set hive.mapjoin.smalltable.filesize=25000000;
    

#### **预处理**

可以对倾斜的数据做个预处理，先合并一部分的数据，Hive 中有这样一个配置：

    
    
    hive.groupby.skewindata=true;
    

有数据倾斜的时候进行负载均衡，当选项设定为 true，生成的查询计划会有两个 MR Job。

  * 第一个 MR Job 会把 Map 端的数据随机分发到下游 Reduce 中，因为是随机，所以相同的 Key 不会完全分发到同一个 Reduce，同时也就不存在数据倾斜的问题了，这样每个 Reduce 做的只是部分聚合操作，所以我们还需要再做一次聚合。
  * 第二个 MR Job 再根据预处理的数据结果按照 Group By Key 分布到 Reduce 中（这个过程可以保证相同的 Group By Key 被分布到同一个 Reduce 中），最后完成最终的聚合操作。

#### **特殊处理**

如果我们已经知道造成我们的数据倾斜字段的值是哪个，我们可以把这个值先过滤掉，然后对这个值的数据再做特殊的处理，最后合并两部分的结果，具体如何操作得看实际的业务场景了。

#### **合并小文件**

之前 HDFS 的章节说过小文件会给 HDFS 带来压力，容易在文件存储端造成瓶颈，影响处理效率。Hive 中可以通过一些配置来合并 Map 和
Reduce 的结果文件，以此来消除小文件带来的影响。

合并 Map 输出文件，默认 True：

    
    
    hive.merge.mapfiles=true
    

是否合并 Reduce 端输出文件，默认 False：

    
    
    hive.merge.mapredfiles=false
    

合并文件的大小，默认 256M：

    
    
    hive.merge.size.per.task=256*1000*1000
    

### 窗口函数

SQL 的聚合函数大家都知道，比如 sum()、avg()、max()
等等这些，这类函数可以将多行数据按照规则聚集为一行，如果我们要针对每一条数据都计算出对应的窗口范围的聚合结果，这时窗口函数就有用了，看下面一个例子。

    
    
    SELECT SUM(pv) 
             over ( 
               PARTITION BY userid 
               ORDER BY month 
               ROWS BETWEEN 3 preceding AND CURRENT ROW) 
    FROM tb 
    

sum(pv) 为分析函数（常用的还有 row_number()、max()、lag()）。over()
为开窗函数，意味着需要开启一个窗口，搭配着分析函数一起使用。

窗口的本质是为了指明分析函数处理数据的作用范围。来看看 over 内部的参数。

  * **partition** ：按照指定字段 userid 来分区，字段相同的值放在一个分区中。
  * **order by** ：分区内部根据 month 字段来排序。
  * **rows between … and …** ：限定 sum(pv) 函数在分区内部计算的物理取值范围，上面的例子没有指定 rows between，默认是 rows between unbounded preceding and current row 即从无边界（窗口分区内部当前行以上的所有数据）到当前行。
    * **n preceding** ：前 n 行
    * **n following** ：后 n 行
    * **current row** ：当前行

![image.png](https://images.gitbook.cn/adef11c0-e2b7-11ea-8b75-e94d40ced3ac)

  * **range between … and …** ：rows between 是物理窗口，限定的是行所在位置的范围，而 range 是逻辑窗口，是指定当前行对应数值的范围取值。

#### **级联求和**

需求：根据下表统计各个用户在各个月份的浏览次数和截止当前月份的累计浏览次数。

源表：tb_daily_pv

uid（用户 ID） | date（天） | pv（浏览次数）  
---|---|---  
aaa | 2020-07-01 | 150  
aaa | 2020-07-05 | 50  
bbb | 2020-07-20 | 350  
bbb | 2020-08-05 | 100  
aaa | 2020-08-09 | 300  
bbb | 2020-07-09 | 50  
ccc | 2020-07-01 | 50  
bbb | 2020-07-01 | 51  
  
分析：

  * 各个月份的浏览次数，只需解析日期为月份，然后 `group by uid, month` 做聚合即可；
  * 截止当前月份的累计浏览次数，需要按照用户 ID 来分区，然后按照月份排序，聚合从当前分区第一行到当前行的所有结果，即 `rows between unbounded preceding and current row`（可省略）。

**具体 SQL：**

    
    
    SELECT t2.uid, 
           t2.month, 
           t2.pv_month, 
           sum(t2.pv_month) OVER ( partition BY t2.uid ORDER BY t2.month) AS pv_total 
    FROM   (SELECT uid, 
                   month, 
                   sum(pv) AS pv_month 
            FROM   (SELECT uid, 
                           Date_format(date, 'yyyy-MM') AS month, 
                           pv 
                    FROM   tb_daily_pv) t1 
            GROUP  BY uid, 
                      month)t2 
    ORDER  BY t2.uid, 
              t2.month 
    

**输出结果：**

uid（用户 ID） | month（月份） | pv（浏览次数） | pv_total（月累计浏览次数）  
---|---|---|---  
aaa | 2020-07 | 200 | 200  
aaa | 2020-08 | 300 | 500  
bbb | 2020-07 | 451 | 451  
bbb | 2020-08 | 100 | 551  
ccc | 2020-07 | 50 | 50  
  
#### **分组 TopN**

这个问题仍然是窗口函数的应用。

**需求** ：是求出每天浏览次数前 2 的用户。

源表：tb_daily_pv

    
    
    SELECT uid,date,rank FROM (
    SELECT uid,date,
         row_number() over(partition BY date ORDER BY pv DESC) rank
       FROM
         tb_daily_pv) as tb1
    WHERE rank <= 2
    

**输出结果：**

uid（用户 ID） | date（天） | pv（浏览次数）  
---|---|---  
aaa | 2020-07-01 | 150  
bbb | 2020-07-01 | 51  
aaa | 2020-07-05 | 50  
bbb | 2020-07-09 | 50  
bbb | 2020-07-20 | 350  
bbb | 2020-08-05 | 100  
aaa | 2020-08-09 | 300  
  
#### **去重**

**需求** ：用户去重，取最小的行为时间。

源表：tb_daily_pv 和 TopN 的逻辑类似，按照用户来分区，同一个分区内取出一条数据即可。

    
    
    SELECT * FROM  ( 
     SELECT *, 
       row_number() 
        OVER (partitionby uid order BY month ASC) num 
     FROM   tb_daily_pv ) t 
    WHERE  t.num=1;
    

### 条件判断

Hive 的条件判断可以用 case when 来实现。

**需求** ：统计当月不同日期时间段的浏览次数。

源表：tb_daily_pv

**分析：** 通过 date_format 解析 date，更加日期分别划分日期段（date_phase），再根据 month 分组，将 pv
聚合相加即可。

    
    
    SELECT 
        month,date_phase,sum(pv) as pv
    FROM 
    (SELECT date_format(date, 'yyyyMM') as month,pv,
      CASE WHEN date_format(date, 'dd') between 1 and 5 THEN '1-5' 
      WHEN date_format(date, 'dd') between 6 and 10  THEN '6-10'
      WHEN date_format(date, 'dd') between 11 and 15  THEN '11-15'
      WHEN date_format(date, 'dd') between 16 and 20  THEN '16-20'
      WHEN date_format(date, 'dd') between 21 and 25  THEN '21-25'
      ELSE '26-end' END as date_phase
    FROM tb_daily_pv) as tb1  group by month,date_phase;
    

注意这里有个小细节，这里嵌套了一个 select 子句，是因为 group by 后面是无法跟 select 中的别名的，即无法在子句中直接 group
by date_phase，而只能 group by 整个很长的 case when 语句，因为 Hive-SQL 的执行顺序为：

> where -> group -> having -> select -> order

所以 select 中定义的别名只能在 order by 中生效。

**输出结果：**

month（月） | date_phase(日期段) | pv（浏览次数）  
---|---|---  
2020-07 | 1-5 | 301  
2020-07 | 6-10 | 50  
2020-07 | 16-20 | 350  
2020-08 | 1-5 | 100  
2020-08 | 6-10 | 300  
  
### 行转列 & 列转行

为了 Hive 的计算和查询更加方便，行转列和列转行在实际工作中也会经常用到，如下两张表所示。

  * 行转列是指将 **行表** 某一列（key）的值作为 **列表** 的字段名输出。
  * 列转行则是相反的操作，将 **列表** 的字段名作为 **行表** 某个字段的值输出。

**行表：tb_row**

uid | key | value  
---|---|---  
101 | c1 | 12  
102 | c2 | 13  
103 | c3 | 21  
101 | c2 | 21  
103 | c1 | 65  
102 | c3 | 0  
  
**列表：tb_col**

uid | c1 | c2 | c3  
---|---|---|---  
101 | 12 | 21 | null  
102 | null | 13 | 0  
103 | 65 | null | 21  
  
#### **行转列**

先将 key 和 value 组成 map 结构，再将 value 值一个个取出来，加上对应的列名即可完成行转列，可见，此方式仅适用于 key
字段的值不多且固定的场景。

    
    
    SELECT UID, 
           kv['c1'] AS c1, 
           kv['c2'] AS c2, 
           kv['c3'] AS c3 
    FROM   (SELECT uid, to_map(key, value) kv 
            FROM   tb_row 
            GROUP  BY uid) t 
    

#### **列转行**

**1\. EXPLODE 函数**

列转行首先想到的应该是 EXPLODE 函数，它可以将一个集合展开拆成行，所以我们需要先将 tb_col 的字段构建出一个 map 结构，然后用
EXPLODE 展开，即：

    
    
    SELECT
      explode (map( 'c1', c1, 'c2', c2, 'c3', c3 ))
    FROM
      tb_col;
    

**获得结果：**

key | value  
---|---  
c1 | 12  
c2 | 13  
c3 | 21  
c2 | 21  
c1 | 65  
c3 | 0  
  
但是 EXPLODE 有一个限制，它出现在 SELECT 子句中的时候，不能与其它列共同出现。也就是我们无法再 select 出 uid，所以接下来只需要把
uid 加上去就完成了，此时需要用到 lateral view。

**2\. lateral view**

lateral view 是用来和类似 explode 这种 UDTF 函数联合用的，lateral view 首先为原始表 tb_col 的每行调用
explode 函数，将生成的结果放到一个虚拟表中，然后这个虚拟表会和输入行进行 join 来达到连接 UDTF 外的 select 字段的目的。具体
SQL 如下所示。

    
    
    SELECT t1.uid, 
           t2.key, 
           t2.value 
    FROM   tb_col t1 lateral view explode 
             (map( 'c1', c1, 'c2', c2, 'c3', c3 )) t2 AS key,value
    

