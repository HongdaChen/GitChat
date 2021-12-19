## Select操作类型

使用Select进行查询时，根据查询需求不同，可以分为过滤、排序、分桶与聚合、连接，这4类型查询操作。

过滤操作包含：Where、Case When Then、Having关键词；排序是Order By、Sort By；分桶与聚合为：Distribute
By、Cluster By、Group By；连接操作：Join。

Select语句的详细语法为：

    
    
    SELECT [ALL | DISTINCT] <select_expression>, <select_expression>, ...
    FROM [<database_name>.]<table_name>
    [WHERE <where_condition>]
    [GROUP BY <col_list>]
    [HAVING <having_condition>]
    [ORDER BY <col_name> [ASC|DESC] [, col_name [ASC|DESC], ...] ]
    [CLUSTER BY <col_list> |
    [DISTRIBUTE BY <col_list>] [SORT BY <col_name> [ASC|DESC] [, col_name [ASC|DESC], ...] ] ]
    [LIMIT (M,)N | [OFFSET M ROWS FETCH NEXT | FIRST] N ROWS ONLY];
    

## 过滤操作

### select数据列过滤

一般而言，在select后指定列名，便可以完成简单的数据列的过滤。

    
    
    --遍历所有数据，在生产中谨慎使用
    select * from <table_name>;
    --查询某几列数据
    select id,name,create_time from <table_name>;
    --查询数据时，为表指定别名
    select t.id from <table_name> t;
    

这种过滤方式，在表的存储类型为列式存储时，如ORCFile、ParquetFile、RCFile等，能够减少数据处理时间。

对数据列进行筛选时，可以使用函数，对某列或某几列数据进行处理：

    
    
    select id,upper(name) from <table_name>;
    

函数支持Hive内置函数和用户自定义函数（UDF），对于函数的使用，后面会单独进行讲解。

对于数字类型的列，支持使用算数运算符，进行算数运算：

    
    
    select id,company_name,user_name, (company_name + user_name) as sumint from <table_name>;
    

支持的算数运算符有：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/489e0da08f110b4be29e4bc19353108b.png#pic_center)

对于字符类型String，支持字符串拼接运算符：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/a6981be6fab6da8ecc3ccb3976a89afe.png#pic_center)

对于复杂数据类型，可以使用操作符进行数据访问：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/379cd0d63e965b067605fdad7ff79287.png#pic_center)

当然，支持将筛选后的数据，使用构造方法，构造为复杂数据类型。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/e281a2a45385d42f1d795a4800a346b8.png#pic_center)

### Where条件过滤

在查询过程中，一般会指定Where、Having关键词进行条件过滤。

其中Where对全表数据进行过滤，即在查询结果分组之前，将不符合条件的数据过滤掉，且条件中不能包含聚合函数。

    
    
    /* Where全表过滤 */
    SELECT * FROM <table_name> WHERE reg_date < 20120000 AND level = 'A' OR level = 'B';
    SELECT * FROM <table_name> WHERE reg_date BETWEEN 20100000 AND 20120000;
    SELECT name FROM <table_name> WHERE level IN ('A', 'B', 'C'', ‘D');
    SELECT name FROM <table_name> WHERE level NOT IN ('A', 'B', 'C'', ‘D');
    

Where后可以使用关系运算符、逻辑运算符进行条件判断。

Hive支持的 **关系运算符** 有：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/ffb25cfb4366948a26a9d5972dea6d72.png#pic_center)

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/555410f1c29aec6725bd7944e2a38db5.png#pic_center)

支持的 **逻辑运算符** 有：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/c584809272af2c700f8ca9e5451d9172.png#pic_center)|

### Having分组过滤

使用having，可以对Group By产生的分组进行过滤，即在查询结果分组之后，将不符合条件的组过滤掉，且条件中常包含聚合函数。

在SQL中的执行次序为：Where -> Group By -> Having。

    
    
    /* Having分组过滤 */
    SELECT name, avg(age)  FROM <table_name>
    WHERE reg_date < 20100000
    GROUP BY level  HAVING BY avg(age) < 30
    

在Having后，和Where一样，支持关系运算符和逻辑运算符的使用。

## 排序操作

### 全局排序Order By

全局排序，使用Order By来进行，具体语法为：

    
    
    SELECT <select_expression>, <select_expression>, ...
    FROM <table_name>
    ORDER BY <col_name> [ASC|DESC] [,col_name [ASC|DESC], ...]
    

但在Hive中使用全局排序时，需要注意，Hive会将所有数据交给一个Reduce任务计算，实现查询结果的全局排序。所以如果数据量很大，只有一个Reduce会耗费大量时间。

Hive的适用场景为离线批处理，在执行全量数据计算任务时，一般是不会用到全局排序的。但在数据查询中，全局排序会经常被用到，而Hive不擅长快速的数据查询，所以需要将Hive处理后的数据存放到支持快速查询的产品中，如Presto、Impala、ClickHouse等。术业有专攻，一个产品一定有自己的适用领域，如果用在不合适的场景，会造成资源浪费。

如果在数据处理过程中必须要用到全局排序，则最好使用UDF转换为局部排序。实现思路为：先预估数据范围，假设这里数据范围是0-100，然后在每个Map作业中，使用Partitioner对数据进行自定义分发，0-10的数据分发到一个Reduce中，10-20的到一个Reduce中，依次类推，然后在每个Reduce作业中进行局部排序即可。

但一般而言，对全量数据进行全局排序的场景很少，一般只需要保证查询结果最终有序即可，这时可以先使用子查询得到一个小的结果集，然后再进行排序。

    
    
    select * from
    (select id,count(1) cnt from <table_name> where id!='0' group by user_id) t
    order by t.cnt;
    

如果是取TOP N的情况，则可以使用子查询，在每个Reduce中进行排序后，各自取得前N个数据，然后再对结果集进行全局排序，最终取得结果。

    
    
    --从表中获取name长度为TOP10的数据
    select t.id,t.name from
    (
    select id,name  from <table_name>
    distribute by length(name)  sort by length(name) desc limit 10
    ) t
    order by length(t.user_name) desc limit 10;
    

这里如果对distribute by不熟悉，在下面的聚合操作中，会有讲到。

### 局部排序Sort By

局部排序，适用Sort By来进行，具体语法为：

    
    
    SELECT <select_expression>, <select_expression>, ...
    FROM <table_name>
    SORT BY <col_name> [ASC|DESC] [,col_name [ASC|DESC], ...]
    

局部排序的操作，Hive会在每个Reduce任务中对数据进行排序。当启动多个Reduce任务时，Order By输出一个文件且全局有序，Sort
By输出多个文件且局部有序。

## 聚合操作

除了Group By之外，Hive还支持的聚合语法有Distribute By和Cluster By。

Group
By是将属于同一组的数据聚合在一起，在底层MapReduce执行过程中，同一组的数据会发送到一个Reduce中；那意味着每个Reduce会包含多组数据，同一组的数据会单独进行聚合运算。

与Group By不同，Distribute By和Cluster By聚合粒度没有Group
By那么细，它们仅仅是保证相同的数据分发到一个Reduce中，所以一般在分桶时使用。

### Distribute By

Distribute By通过哈希取模的方式，将列值相同的数据发送给同一个Reducer任务，实现数据的聚合。通常与Sort
By合并使用，实现先聚合后排序，可以指定升序ASC还是降序DESC。但Distribute By必须在Sort By之前。

    
    
    SELECT <select_expression>, <select_expression>, ...
    FROM <table_name>
    DISTRIBUTE BY <col_list>
    [SORT BY <col_name> [ASC|DESC] [, col_name [ASC|DESC], ...] ]
    

因为Distribute By可以配合Sort By使用，所以在每个Reduce任务中的数据可以实现更加灵活的局部排序。既可以DISTRIBUTE
BY字段进行排序，也可以使用其它字段进行排序。

### Cluster By

如果Distribute By列和Sort By列完全相同，且按升序排列，那么Cluster By = Distribute By … Sort
By。所以Cluster By相对没有Distribute By那么灵活，而且不能自定义排序。

    
    
    SELECT <select_expression>, <select_expression>, ...
    FROM <table_name>
    CLUSTER BY <col_list>
    

## JOIN连接

### Join方式

Hive支持的Join方式有Inner Join和Outer Join，这和标准SQL一致。除此之外，还支持一种特殊的Join：Left Semi-
Join。

#### Inner Join

其中，Inner Join是求两张表的交集数据，只有两张表同时拥有的数据，才会被筛选出来。

    
    
    SELECT a.* FROM a JOIN b ON (a.id = b.id)
    

#### Outer Join

Outer Join则包含Left Outer join（左外连接）、Right Outer join（右外连接）、Full Outer
join（全外连接）。

左外连接会保留左表的全部数据，如果右表中有对应数据，则进行补全，如果没有则填充为NULL。

    
    
    SELECT * FROM a Left Outer JOIN b ON (a.id = b.id)
    

右外连接会保留右表的全部数据，如果左表中有对应数据，则进行补全，如果没有则填充为NULL。

    
    
    SELECT * FROM a Right Outer JOIN b ON (a.id = b.id)
    

全外连接则会保留两张表的全部数据，两张表中有相同数据，则补全，如果没有则填充为NULL。

    
    
    SELECT * FROM a Full Outer JOIN b ON (a.id = b.id)
    

#### Left Semi-Join

Hive中，可以使用左半开连接实现 in / exists 语法，在0.13版本推出IN/NOT IN/EXISTS/NOT EXISTS
语法后，已经不经常使用。

    
    
    SELECT a.key, a.val
    FROM a LEFT SEMI JOIN b ON (a.key = b.key)
    

它的作用是，当a表的key值，存在于（IN 、Exists）b的key值中时，返回a表的数据。效果等同于以下SQL：

    
    
    SELECT a.key, a.value
    FROM a WHERE a.key in (SELECT b.key FROM B);
    

### Join优化

#### StreamTable

Hive在执行Join时，默认会将前面的表直接加载到缓存，后面一张表进行stream处理，即shuffle操作。这样可以减少shuffle过程，因为直接加载到缓存中的表，只需要等待后面stream过来的表数据，而不需要进行shuffle，相当于整体减少了一次shuffle过程。

所以在SQL语句中，大表放在join后面，会有很好的优化效果，或者可以直接标注为StreamTable，来指定进行stream的表。

    
    
    SELECT /*+ STREAMTABLE(a) */ a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key1)
    

#### MapJoin

Hive在执行Join时，可以使用MapJoin，将小表直接加载到Map作业中，以减少Shuffle开销。其实效果与stream
table一致。都是缓存小表数据的一种方式。

    
    
    SELECT /*+ MAPJOIN(b) */ a.key, a.value
    FROM a JOIN b ON a.key = b.key
    

