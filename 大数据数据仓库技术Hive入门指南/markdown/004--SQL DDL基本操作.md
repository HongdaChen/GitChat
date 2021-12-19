## 概述&Client启动

### 基本概述

Hive DDL根据操作对象的不同可分为：数据库操作、表的基本操作、表的高级操作、函数操作。

对数据库的操作包含：对数据库的创建、删除、修改、查看元信息、切换、展示所有数据库。

表的基本操作：对表的创建、删除、修改、清空、查看元信息、查看所有表。

表的高级操作主要是分区、分桶。

函数的操作包含：创建、删除、查看元信息、查看所有函数。

### 客户端启动

执行Hive SQL时，可以使用beeline，也可以使用Hive Cli，但前提是启动HiveServer2和MetaStore。

先使用jps命令查看Node03中，Hive服务是否已经启动。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/703239b0e37d1ebe947346b51ba2ae81.png#pic_center)

如果没有启动，使用以下命令启动即可。

    
    
    # 启动HiveServer2
    hive --service hiveserver2 &
    # 启动Metastore
    hive --service metastore &
    

然后使用beeline或hive cli进入到hive命令行界面。

    
    
    # beeline方式（推荐）
    beeline -u jdbc:hive2://node03:10000 -n root
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/34512fc25931ea88de4fe0d96a7ffe58.png#pic_center)

    
    
    # hive cli方式
    hive
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/a443a1178cbeca2a127e7f778a7a696a.png#pic_center)

## 数据库基本操作

**创建数据库**

    
    
    CREATE DATABASE [IF NOT EXISTS] <database_name>
    [COMMENT '<database_comment>']
    [WITH DBPROPERTIES ('<property_name>'='<property_value>', ...)];
    

例如，创建数据库mydb：

    
    
    create database mydb;
    

**删除数据库**

可以删除空数据库，如果数据库不为空，则删除失败。

    
    
    DROP DATABASE [IF EXISTS] <database_name>;
    

例如，删除数据库mydb：

    
    
    drop database mydb;
    

可以使用cascade指令，强制删除非空数据库：

    
    
    DROP DATABASE [IF EXISTS] <database_name> cascade;
    

例如，强制删除数据库mydb：

    
    
    --先创建mydb数据库
    create database mydb;
    --强制删除数据库mydb
    drop database mydb cascade;
    

**修改数据库属性：**

可以使用ALTER DATABASE命令为某个数据库的DBPROPERTIES设置键-
值对属性值，来描述这个数据库的属性信息。数据库的其他元数据信息都是不可更改的，包括数据库名和数据库所在的目录位置。

    
    
    ALTER DATABASE <database_name> SET DBPROPERTIES ('<property_name>'='<property_value>', ...);
    

例如，为数据库mydb添加自定义属性edited-by，表示编辑者。

    
    
    --先创建mydb数据库
    create database mydb;
    --添加数据库属性
    alter database mydb set dbproperties('edited-by'='Joe');
    

**列出所有数据库：**

    
    
    SHOW DATABASES;
    

**切换数据库：**

    
    
    USE <database_name>;
    

例如，切换到mydb数据库中。

    
    
    USE mydb;
    

**查看数据库详情：**

    
    
    DESCRIBE database <database_name>;
    

例如，查看mydb数据库详情。

    
    
    DESCRIBE DATABASE mydb;
    

可以使用describe extended查看数据库详细信息：

    
    
    DESCRIBE DATABASE extended <database_name>;
    

## 表的基本操作

### 内表和外表

#### 什么是内表和外表？

Hive完整的建表语句如下：

    
    
    CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS]
    [<database_name>.]<table_name>
    [(<col_name> <data_type> [COMMENT '<col_comment>'] [, <col_name> <data_type> ...])]
    [COMMENT '<table_comment>']
    [PARTITIONED BY (<partition_key> <data_type> [COMMENT '<partition_comment>']
    [, <partition_key > <data_type>...])]
    [CLUSTERED BY (<col_name> [, <col_name>...])
    [SORTED BY (<col_name> [ASC|DESC] [, <col_name> [ASC|DESC]...])]
    INTO <num_buckets> BUCKETS]
    [
    [ROW FORMAT <row_format>]
    [STORED AS file_format]
    | STORED BY '<storage.handler.class.name>' [WITH SERDEPROPERTIES (<...>)]
    ]
    [LOCATION '<file_path>']
    [TBLPROPERTIES ('<property_name>'='<property_value>', ...)];
    

Hive的表根据对数据拥有的权限不同，又分为内表（Managed Table）和外表（External
Table）。其中内表是主要使用的表形式，而外表一般作为数据导入时的临时表使用。

如何区分内外表？Hive的表由两部分组成，一部分是存储在HDFS上的数据，一部分是存储在MetaStore中的元数据（表结构、权限、属性等信息）。内表和外表都在MetaStore中存储有元数据，但内表对HDFS上存储的数据有所有权，即它存储在Hive用于存放表的数据目录中，拥有读写的权限；而外表对HDFS上存储的数据没有所有权，仅有读权限，数据存放位置灵活，需要在建表时使用Location指定数据的存放位置。

但因为外表只有读权限，只能对表进行读操作，而不能进行写操作；所以在删除表的时候，外表只删除元数据，因为没有权限，所以无法删除数据，而内表删除后，数据和元数据都会被删除。

那为什么会使用外表呢？一般结构化数据通过ETL流程到达Hadoop平台时，因为可能会存在编码、换行符、脏数据问题，所以不会直接导入到Hive中存储为内表，而是先存储到为ETL创建的目录中。这个目录，因为不在Hive数据目录下，所以Hive并没有所有权。

于是Hive在将数据接入之前，首先创建外表，指定表的元数据信息（create table t1(col1 type, col2 type
..)），而数据则使用location指定到存放位置；因为有读权限，所以可以对外表数据进行查询，以便确认表的元数据信息是否正确（有没有缺少字段，字段类型是否合适），查看数据的编码、换行符是否存在问题。如果存在问题，则进行调整。

根据创建的外表，确定数据没有问题后，再创建内表，将外表的数据查询出来，导入到内表中。此时，数据就会存放到Hive目录下，并且拥有所有权限，可以进行增删改查操作。然后外表已经完成它的功能，可以进行删除，但删除操作仅仅会清空元数据，而不会删除数据（没有权限）。

#### 外表的创建

外表使用External标识，创建语法为：

    
    
    CREATE EXTERNAL TABLE <table_name>
    (<col_name> <data_type> [, <col_name> <data_type> ...])
    LOCATION '<file_path>';
    

比如HDFS中存放了采集到的人员信息数据psn.txt：

    
    
    1,tom,18,music-game-driver,std_addr
    2,jolin,21,music-movie,std_addr:beijing-work_addr:shanghai-addr:tokyo
    3,tony,33,book-game-food,std_addr:beijing-work_addr:xian
    4,lilei,12,scl_addr:xizhimen-home_addr:null
    5,hanmeimei,12,scl_addr:xizhimen
    6,baby,3,food,addr:tianjing
    

将数据上传到HDFS的/tmp/hive_data/psn目录下：

    
    
    hadoop fs -mkdir -p /tmp/hive_data/psn
    hadoop fs -put psn.txt /tmp/hive_data/psn/
    

现在需要对这些数据创建外表ext_psn。

    
    
    use default;
    CREATE EXTERNAL TABLE ext_psn(
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
    LINES TERMINATED BY '\n'
    LOCATION '/tmp/hive_data/psn/';
    

因为文件是外部数据，所以在创建表的时候，需要指定文件的分隔符。ROW FORMAT DELIMITED 是标志开始设置分隔符的语句。

首先每行数据的各个字段值，是由 " , " 进行分割，所以使用FIELDS TERMINATED BY ',' 进行指定。

其次数据的第4、5个字段，分别是数组和Map，属于复杂数据类型。各元素使用 “ - ” 进行分隔，例如第4个字段music-game-
driver，是3个元素music、game、driver组成的数组，而第5个字段shanghai-
addr，作为map类型，它的key和value分别是shanghai、addr。所以需要使用COLLECTION ITEMS TERMINATED BY
'-'指定元素的分隔符。

而第5个字段，例如std _addr:beijing-work_ addr:shanghai-
addr:tokyo，多个map元素之间使用:进行分隔，所以使用MAP KEYS TERMINATED BY ':' 进行指定。

行分隔符在Linux下默认是\n，使用LINES TERMINATED BY '\n’指定。

最后，对于外表，需要使用LOCATION指定数据在HDFS上的存放路径，之前数据已经存放到了/tmp/hive_data/psn/目录。

外表创建完成后，可以查看外表的表结构信息，并对数据进行查询。

    
    
    desc ext_psn;
    select * from ext_psn;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/ccca7c3d3e23f8e936774a5c61bd7884.png#pic_center)

查看表的详细元数据信息：

    
    
    desc formatted ext_psn;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/9e162ce2f6e86da3e6606df1ca74435d.png#pic_center)

#### 内表的创建

内表的创建和普通SQL相似，建表语法为：

    
    
    CREATE TABLE <table_name>
    [(<col_name> <data_type> [, <col_name> <data_type> ...])];
    

数据默认会存放到/user/hive/warehouse/$${db _name}.db/$${table_
name}目录下。如果是default数据库，数据存放位置为：/user/hive/warehouse/${table_name}。

首先，创建一张内表inner_psn，与案例中的外表字段保持一致。

    
    
    CREATE TABLE inner_psn(
    id INT comment 'ID',
    name STRING comment '姓名',
    age INT comment '年龄',
    likes ARRAY<STRING> comment '爱好',
    address MAP<STRING,STRING> comment '地址'
    ) comment '人员信息表';
    

查看表的元数据信息：

    
    
    desc formatted inner_psn;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/aa577f018b84795c987c7bf31ef7b570.png#pic_center)

当然在创建内表的时候，也可以使用Location自定义数据的存放位置，但与外表不同，内表必须对此目录拥有读写权限，以便对数据进行操作。

要注意区分内外表中Location用法的不同，在外表是Location是必须属性，要指定数据存放位置，用于数据的读取；在内表中，Location的作用则是更改表数据文件的默认存放位置。

### 临时表

除了内外表，在Hive中还有临时表。临时表是存放在内存中的一种表，一般在数据处理中用于临时保存数据，仅在当前Session可见，Session结束后即被删除。

而内表和外表数据永久表，如果临时表和永久表重名，在当前Session中该表名指向临时表，同名的永久表无法被访问。

临时表的创建语法为：

    
    
    CREATE TEMPORARY TABLE <table_name>
    (<col_name> <data_type> [, <col_name> <data_type> ...]);
    

创建临时表tmp_psn;

    
    
    CREATE TEMPORARY TABLE tmp_psn(
    id INT comment 'ID',
    name STRING comment '姓名',
    age INT comment '年龄',
    likes ARRAY<STRING> comment '爱好',
    address MAP<STRING,STRING> comment '地址'
    ) comment '人员信息表';
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/6828d5f75fbf20e68e682adc9dacd255.png#pic_center)

临时表本质上是特殊的一种内表，只是存储的位置不同。

### 表的创建方式

除了可以直接通过建表语句对表进行创建之外，对于内表的创建，可以使用简便方式直接复制其它表或视图的Schema（元数据）和数据。

比如外表创建之后，如果经过查询，发现表的Schema、数据均没有问题，创建内表时可以直接复制外表的内容。

首先可以通过已存在的表或视图来创建内表，只复制Schema，不复制数据，语法为：

    
    
    CREATE TABLE <table_name>
    LIKE <existing_table_or_view_name>;
    

还可以通过查询结果来创建内表，既复制Schema，又复制数据（会转换为MapReduce任务执行）：

    
    
    CREATE TABLE <table_name>
    AS SELECT <select_statement>;
    

接下来创建内表copy _psn_ 1，复制ext_psn的Schema信息：

    
    
    create table copy_psn_1 like ext_psn;
    

查看是否创建成功：

    
    
    desc copy_psn_1;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/d982d336a48f3b098f497a6e2b1264aa.png#pic_center)
然后创建内表copy _psn_ 2，既复制外表ext_psn的Schema，又拷贝其全部数据。

    
    
    create table copy_psn_2 as select * from ext_psn;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/7129394fb72a70abc76625ac40122d5b.png#pic_center)

查看是否创建成功：

    
    
    select * from copy_psn_2;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/f3f42b0e674ea8f5d55c7ad04ed64653.png#pic_center)

### 表的其它操作

**删除表：**

对已经创建的表，可以执行删除：

    
    
    DROP TABLE <table_name>;
    

对于内表，表的元数据和数据都会被删除；对于外表，只删除元数据。

**修改表：**

对于创建成功的表，可以进行修改操作，主要是表的重命名、属性的修改、列的增删改：

    
    
    /* 表重命名 */
    ALTER TABLE <table_name> RENAME TO <new_table_name>;
    /* 修改表属性 */
    ALTER TABLE <table_name> SET TBLPROPERTIES ('<property_name>' = '<property_value>' ... );
    ALTER TABLE <table_name> SET SERDEPROPERTIES ('<property_name>' = '<property_value>' ... );
    ALTER TABLE <table_name> SET LOCATION '<new_location>';
    /* 增加、删除、修改、替换列 */
    ALTER TABLE <table_name> ADD COLUMNS (<col_spec> [, <col_spec> ...])
    ALTER TABLE <table_name> DROP [COLUMN] <col_name>
    ALTER TABLE <table_name> CHANGE <col_name> <new_col_name> <new_col_type>
    ALTER TABLE <table_name> REPLACE COLUMNS (<col_spec> [, <col_spec> ...])
    

比如，可以将外表切换为内表，只需要修改表的属性即可。

    
    
    alert table ext_psn set tblproperties('EXTERNAL'='FALSE');
    

**清空表：**

清空表中的数据，但不删除元数据；该操作用于内表，不能用于外表。

    
    
    TRUNCATE TABLE <table_name>;
    

**查看表详情：**

可以使用describe命令查看表详情，当然也可以使用desc，desc是describe的简写形式。

    
    
    DESCRIBE <table_name>;
    

使用describe formatted可以查看更详细的表信息。

    
    
    DESCRIBE formatted <table_name>;
    

show create table命令可以查看表的schema信息：

    
    
    SHOW CREATE TABLE <table_name>;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/8c139c1da9a5ef83d820f989c8064a92.png#pic_center)

**列出当前数据库所有表：**

    
    
    show tables;
    

这些命令大多与标准SQL相同或类似，只要掌握Hive SQL特定的语法即可。

