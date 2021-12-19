## 数据导入

### 数据预处理

将文件导入到Hive中，需要文件编码格式为UTF-8，\n为换行符，否则就需要进行预处理。处理过程分为两部分：编码格式、换行符。

#### 编码格式处理

对于中文字符，如果是ASCii码，或者其它编码，则在Hive表中无法正确显示。

首先可以使用file命令提前查看文件编码类型和换行符情况。

    
    
    file $filename
    

如果编码不是UTF-8，则需要进行编码转换。转换方式可以在建表前，提前对文件进行转码处理；也可以不对文件进行处理，在表中指定文件编码格式。

对文件提前进行转码处理，可以使用iconv工具进行：

    
    
    # iconv是转码工具，-f源编码格式，-t目标编码格式
    iconv -f gbk -t utf-8 $sourceFile > $targetFile
    

如果不对文件进行提前的转码处理，可以在表中指定文件的编码格式：

    
    
    ALTER TABLE <tableName> SET SERDEPROPERTIES('serialization'='GBK');
    

#### 换行符处理

在不同操作系统中，文件默认的换行符会有所不同。Windows文件用\r\n换行，而Unix和Linux文件用\n换行。Windows文件直接导入到Hive表中时，最后一列数据因为多了'\r'，会存在问题；如果最后一列是数值类型，直接显示为NULL。所以必须处理Windows换行符，将\r\n转换为\n。

转换方式同样是两种，可以事前对文件进行换行符的转换，也可以不对文件进行转换，而在表中指定换行符类型。

事前对文件换行符转换，可以使用dos2unix工具：

    
    
    dos2unix $fileName
    

也可以不对文件进行处理，直接在建表时使用LINES TERMINATED BY指定换行符类型：

    
    
    ALTER TABLE <tableName> SET SERDEPROPERTIES('serialization'='GBK');
    
    CREATE EXTERNAL TABLE psn_win(
    id INT comment 'ID',
    name STRING comment '姓名',
    age INT comment '年龄',
    likes ARRAY<STRING> comment '爱好',
    address MAP<STRING,STRING> comment '地址'
    )comment '人员信息表'
    ROW FORMAT DELIMITED
    LINES TERMINATED BY '\r\n'
    LOCATION '/tmp/hive_data/psn/';
    

### 将文件导入表或分区（Load）

可以使用Load命令，直接将文件中的数据导入已存在的表或分区。要注意的是，Load仅仅是将数据文件移动到表或分区的目录中，不会对数据进行任何处理，如分桶、排序，也不支持动态分区。

但Load命令在较新的版本中，如3.x之后，对于有些不支持的使用方式，如自动分区、分桶，会在解析时自动转换为insert..select运行，而不会报错。为了规范起见，不建议随意使用Load命令，Load命令只作为将文件快速移动到Hive表的存储位置上时使用。

使用Load导入文件时，命令格式如下：

    
    
    LOAD DATA [LOCAL] INPATH '<path>’  [OVERWRITE] INTO
    TABLE <tablename>
    [PARTITION (<partition_key>=<partition_value>, ...)];
    

其中是文件存储路径，如果标注LOCAL，指向本地磁盘路径，不标则指向HDFS路径。既可指向文件也可指向目录，系统会将指定文件或目录下所有的文件移入表内。如果是HDFS路径时，执行操作的用户必须是的Owner，同时Hive用户必须有读写权限。

例如，创建一张内表，并将数据文件Load到表中。

    
    
    --创建人员信息表psn_load
    CREATE TABLE psn_load(
    id INT comment 'ID',
    name STRING comment '姓名',
    age INT comment '年龄',
    likes ARRAY<STRING> comment '爱好',
    address MAP<STRING,STRING> comment '地址'
    )comment '人员信息表'
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    COLLECTION ITEMS TERMINATED BY '-'
    MAP KEYS TERMINATED BY ':'
    LINES TERMINATED BY '\n';
    
    --将数据导入到表中
    load data inpath '/tmp/hive_data/psn/' into table psn_load;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/edf41ecdb1440eb79435ff4c298b383a.png#pic_center)

一般而言，在生产环境中不推荐使用Load方式进行数据导入，如果Hive集群设置了安全模式，在进行Load方式导入时权限设置步骤较多。推荐的方式是创建外表，并将外表Location设置为数据文件所在目录。

### 将查询结果导入表或分区（Insert..Select导入)

在进行数据导入时，推荐先对数据创建外表，然后检查数据格式、换行符的问题，如果不存在异常，则使用Insert..Select直接将数据导入到内表中。

    
    
    INSERT (OVERWRITE | INTO)  TABLE  <table_name>
    SELECT <select_statement>;
    

Insert..Select方式比较灵活，支持分区、分桶。

可以在导入时，指定数据要插入的分区：

    
    
    INSERT (OVERWRITE | INTO)  TABLE  <table_name>
    PARTITION (<partition_key>=<partition_value>[, <partition_key>=<partition_value>, ...])
    SELECT <select_statement>;
    

也可以进行动态分区导入，系统自动判断分区：

    
    
    /* 开启动态分区支持，并设置最大分区数*/
    set hive.exec.dynamic.partition=true;
    set hive.exec.max.dynamic.partitions=2000;
    /* <dpk>为动态分区键， <spk>为静态分区键 */
    INSERT (OVERWRITE | INTO) TABLE <table_name>
    PARTITION ([<spk>=<value>, ..., ] <dpk>, [..., <dpk>])
    SELECT <select_statement>;
    

当然也支持在数据导入时自动分桶：

    
    
    --开启配置
    set hive.enforce.bucketing=true;
    --插入数据
    INSERT (OVERWRITE | INTO)  TABLE  <table_name>
    SELECT <select_statement>;
    

## 数据导出

Hive支持将数据导出到本地目录，可能生成多个文件（与Map任务数对应）。

数据导出命令为：

    
    
    INSERT OVERWRITE [LOCAL] DIRECTORY <directory>
    [ROW FORMAT DELIMITED
    [FIELDS TERMINATED BY char [ESCAPED BY char] ]
    [COLLECTION ITEMS TERMINATED BY char]
    [MAP KEYS TERMINATED BY char]
    [LINES TERMINATED BY char]
    [NULL DEFINED AS char] ]
    [STORED AS TEXTFILE|ORC|CSVFILE]
    SELECT <select_statement>;
    

与导入数据不一样，不能用Insert Into导出数据，只能用Insert Overwrite。在导出时，可以使用ROW FORMAT
DELIMITED指定数据在最终导出文件中的分隔符，也可以使用STORED AS指定文件类型，默认为TEXTFILE。

命令中标注LOCAL代表导出到本地磁盘，不标代表导出到HDFS；DIRECTORY指定数据导出的文件目录。

表数据在导出文件中的分隔符，需要在ROW FORMAT DELIMITED声明后进行设置；FIELDS TERMINATED
BY设置每行数据中各字段的分隔符；LINES TERMINATED BY设置每行数据之间的分隔符，默认为\n；COLLECTION ITEMS
TERMINATED BY设置集合中各个成员间的分隔符，如Map集合有["name":"zhang",
"age":18]两个成员，则它们在存储时会按照指定的分隔符进行隔开；MAP KEYS TERMINATED
BY设置每个Map元素，即、之间的分隔符；NULL DEFINED AS设置空数据的替换字符。

例如，将表psn_load导出到本地/root/psn目录下。

    
    
    insert overwrite local directory '/root/psn'
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY ','
    COLLECTION ITEMS TERMINATED BY '-'
    MAP KEYS TERMINATED BY ':'
    LINES TERMINATED BY '\n'
    STORED AS TEXTFILE
    SELECT * FROM psn_load;
    

最终导出结果为：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/f37343ac135aaf4af313b891dd1c2247.png#pic_center)

当然，也可以将数据导出到多个路径中：

    
    
    FROM <from_statement>
    INSERT OVERWRITE [LOCAL] DIRECTORY <directory1> SELECT <select_statement1>
    [INSERT OVERWRITE [LOCAL] DIRECTORY <directory1> SELECT <select_statement1>] ...
    

例如，将表psn _load导出到本地/tmp/psn_ n1和/tmp/psn_n2目录中：

    
    
    from psn_load
    insert overwrite local directory '/tmp/psn_n1' select *
    insert overwrite local directory '/tmp/psn_n2' select *;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/3da7b97da797064dcbf9ed60ce261ff8.png#pic_center)

默认存储格式为：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/40aed89bcbeec17728f84c9a8e5298c6.png#pic_center)

## 表的备份&还原

为了对Hive表数据进行备份，可以将表的元数据和数据都导出到HDFS中：

    
    
    export table <table_name> PARTITION (<spk>=<value>) to '<PATH>';
    

对备份的数据，可以使用import语法进行还原：

    
    
    import from '<PATH>';
    

例如，对数据表psn_load进行备份，然后将表删除后再进行恢复：

    
    
    --备份数据表
    export table psn_load to '/tmp/psn_load';
    --删除表
    drop table psn_load;
    --恢复数据表
    import from '/tmp/psn_load';
    

当然， 如果在还原时指定了表名，则会对备份表进行改名操作：

    
    
    import table <table_name> from '<PATH>';
    

支持将表数据还原到分区中：

    
    
    export table <table_name> PARTITION (<spk>=<value>) to '<PATH>';
    import table <table_name> partition (<spk>=<value>) from '<PATH>';
    

也支持将表数据导入成外表：

    
    
    export table <table_name> to '<PATH>';
    import external table <table_name> from '<PATH>';
    

