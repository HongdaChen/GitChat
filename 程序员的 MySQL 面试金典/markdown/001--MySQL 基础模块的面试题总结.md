### 说一下 MySQL 执行一条查询语句的内部执行过程？

  * 客户端先通过连接器连接到 MySQL 服务器。
  * 连接器权限验证通过之后，先查询是否有查询缓存，如果有缓存（之前执行过此语句）则直接返回缓存数据，如果没有缓存则进入分析器。
  * 分析器会对查询语句进行语法分析和词法分析，判断 SQL 语法是否正确，如果查询语法错误会直接返回给客户端错误信息，如果语法正确则进入优化器。
  * 优化器是对查询语句进行优化处理，例如一个表里面有多个索引，优化器会判别哪个索引性能更好。
  * 优化器执行完就进入执行器，执行器就开始执行语句进行查询比对了，直到查询到满足条件的所有数据，然后进行返回。

### MySQL 提示“不存在此列”是执行到哪个节点报出的？

此错误是执行到分析器阶段报出的，因为 MySQL 会在分析器阶段检查 SQL 语句的正确性。

### MySQL 查询缓存的功能有何优缺点？

MySQL 查询缓存功能是在连接器之后发生的，它的优点是效率高，如果已经有缓存则会直接返回结果。
查询缓存的缺点是失效太频繁导致缓存命中率比较低，任何更新表操作都会清空查询缓存，因此导致查询缓存非常容易失效。

### 如何关闭 MySQL 的查询缓存功能？

MySQL 查询缓存默认是开启的，配置 query _cache_ type 参数为 DEMAND（按需使用）关闭查询缓存，MySQL 8.0
之后直接删除了查询缓存的功能。

### MySQL 的常用引擎都有哪些？

MySQL 的常用引擎有 InnoDB、MyISAM、Memory 等，从 MySQL 5.5.5 版本开始 InnoDB 就成为了默认的存储引擎。

### MySQL 可以针对表级别设置数据库引擎吗？怎么设置？

可以针对不同的表设置不同的引擎。在 create table 语句中使用 engine=引擎名（比如Memory）来设置此表的存储引擎。完整代码如下：

    
    
    create table student(
       id int primary key auto_increment,
       username varchar(120),
       age int
    ) ENGINE=Memory
    

### 常用的存储引擎 InnoDB 和 MyISAM 有什么区别？

InnoDB 和 MyISAM 最大的区别是 InnoDB 支持事务，而 MyISAM 不支持事务，它们主要区别如下：

  * InnoDB 支持崩溃后安全恢复，MyISAM 不支持崩溃后安全恢复；
  * InnoDB 支持行级锁，MyISAM 不支持行级锁，只支持到表锁；
  * InnoDB 支持外键，MyISAM 不支持外键；
  * MyISAM 性能比 InnoDB 高；
  * MyISAM 支持 FULLTEXT 类型的全文索引，InnoDB 不支持 FULLTEXT 类型的全文索引，但是 InnoDB 可以使用 sphinx 插件支持全文索引，并且效果更好；
  * InnoDB 主键查询性能高于 MyISAM。

### InnoDB 有哪些特性？

**1）插入缓冲(insert buffer)**
：对于非聚集索引的插入和更新，不是每一次直接插入索引页中，而是首先判断插入的非聚集索引页是否在缓冲池中，如果在，则直接插入，否则，先放入一个插入缓冲区中。好似欺骗数据库这个非聚集的索引已经插入到叶子节点了，然后再以一定的频率执行插入缓冲和非聚集索引页子节点的合并操作，这时通常能将多个插入合并到一个操作中，这就大大提高了对非聚集索引执行插入和修改操作的性能。

**2）两次写(double write)** ：两次写给 InnoDB 带来的是可靠性，主要用来解决部分写失败(partial page
write)。doublewrite 有两部分组成，一部分是内存中的 doublewrite buffer ，大小为
2M，另外一部分就是物理磁盘上的共享表空间中连续的 128 个页，即两个区，大小同样为 2M。当缓冲池的作业刷新时，并不直接写硬盘，而是通过 memcpy
函数将脏页先拷贝到内存中的 doublewrite buffer，之后通过 doublewrite buffer 再分两次写，每次写入 1M
到共享表空间的物理磁盘上，然后马上调用 fsync 函数，同步磁盘。如下图所示

![avatar](https://images.gitbook.cn/Fh1qmU3TsCwz4YDcT9C92j0GWfDo)

**3）自适应哈希索引(adaptive hash index)** ：由于 InnoDB 不支持 hash 索引，但在某些情况下 hash
索引的效率很高，于是出现了 adaptive hash index 功能， InnoDB 存储引擎会监控对表上索引的查找，如果观察到建立 hash
索引可以提高性能的时候，则自动建立 hash 索引。

### 一张自增表中有三条数据，删除了两条数据之后重启数据库，再新增一条数据，此时这条数据的 ID 是几？

如果这张表的引擎是 MyISAM，那么 ID=4，如果是 InnoDB 那么 ID=2（MySQL 8 之前的版本）。

### MySQL 中什么情况会导致自增主键不能连续？

以下情况会导致 MySQL 自增主键不能连续：

  * 唯一主键冲突会导致自增主键不连续；
  * 事务回滚也会导致自增主键不连续。

### InnoDB 中自增主键能不能被持久化？

自增主键能不能被持久化，说的是 MySQL 重启之后 InnoDB 能不能恢复重启之前的自增列，InnoDB 在 8.0 之前是没有持久化能力的，但
MySQL 8.0 之后就把自增主键保存到 redo log（一种日志类型，下文会详细讲）中，当 MySQL 重启之后就会从 redo log 日志中恢复。

### 什么是独立表空间和共享表空间？它们的区别是什么？

**共享表空间** ：指的是数据库的所有的表数据，索引文件全部放在一个文件中，默认这个共享表空间的文件路径在 data 目录下。 **独立表空间**
：每一个表都将会生成以独立的文件方式来进行存储。
共享表空间和独立表空间最大的区别是如果把表放再共享表空间，即使表删除了空间也不会删除，所以表依然很大，而独立表空间如果删除表就会清除空间。

### 如何设置独立表空间？

独立表空间是由参数 innodb _file_ per_table 控制的，把它设置成 ON 就是独立表空间了，从 MySQL 5.6.6
版本之后，这个值就默认是 ON 了。

### 如何进行表空间收缩？

使用重建表的方式可以收缩表空间，重建表有以下三种方式：

  * alter table t engine=InnoDB
  * optmize table t
  * truncate table t

### 说一下重建表的执行流程？

  * 建立一个临时文件，扫描表 t 主键的所有数据页；
  * 用数据页中表 t 的记录生成 B+ 树，存储到临时文件中；
  * 生成临时文件的过程中，将所有对 t 的操作记录在一个日志文件（row log）中；
  * 临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表 t相同的数据文件；
  * 用临时文件替换表 t 的数据文件。

### 表的结构信息存在哪里？

表结构定义占有的存储空间比较小，在 MySQL 8 之前，表结构的定义信息存在以 .frm 为后缀的文件里，在 MySQL 8
之后，则允许把表结构的定义信息存在系统数据表之中。

### 什么是覆盖索引？

覆盖索引是指，索引上的信息足够满足查询请求，不需要再回到主键上去取数据。

### 如果把一个 InnoDB 表的主键删掉，是不是就没有主键，就没办法进行回表查询了？

可以回表查询，如果把主键删掉了，那么 InnoDB 会自己生成一个长度为 6 字节的 rowid 作为主键。

### 执行一个 update 语句以后，我再去执行 hexdump 命令直接查看 ibd 文件内容，为什么没有看到数据有改变呢？

可能是因为 update 语句执行完成后，InnoDB 只保证写完了 redo log、内存，可能还没来得及将数据写到磁盘。

### 内存表和临时表有什么区别？

  * 内存表，指的是使用 Memory 引擎的表，建表语法是 create table … engine=memory。这种表的数据都保存在内存里，系统重启的时候会被清空，但是表结构还在。除了这两个特性看上去比较“奇怪”外，从其他的特征上看，它就是一个正常的表。
  * 而临时表，可以使用各种引擎类型 。如果是使用 InnoDB 引擎或者 MyISAM 引擎的临时表，写数据的时候是写到磁盘上的。

### 并发事务会带来哪些问题？

  * 脏读
  * 修改丢失
  * 不可重复读
  * 幻读

### 什么是脏读和幻读？

脏读是一个事务在处理过程中读取了另外一个事务未提交的数据；幻读是指同一个事务内多次查询返回的结果集不一样（比如增加了或者减少了行记录）。

### 为什么会出现幻读？幻读会带来什么问题？

因为行锁只能锁定存在的行，针对新插入的操作没有限定，所以就有可能产生幻读。 幻读带来的问题如下：

  * 对行锁语义的破坏；
  * 破坏了数据一致性。

### 如何避免幻读？

使用间隙锁的方式来避免出现幻读。间隙锁，是专门用于解决幻读这种问题的锁，它锁的了行与行之间的间隙，能够阻塞新插入的操作
间隙锁的引入也带来了一些新的问题，比如：降低并发度，可能导致死锁。

### 如何查看 MySQL 的空闲连接？

在 MySQL 的命令行中使用 `show processlist;` 查看所有连接，其中 Command 列显示为 Sleep
的表示空闲连接，如下图所示：

![avatar](https://images.gitbook.cn/FhQbOAma7Hrue1H5pB3NVy9Y9VVe)

### MySQL 中的字符串类型都有哪些？

MySQL 的字符串类型和取值如下：

**类型** | **取值范围**  
---|---  
CHAR(N) | 0~255  
VARCHAR(N) | 0~65536  
TINYBLOB | 0~255  
BLOB | 0~65535  
MEDUIMBLOB | 0~167772150  
LONGBLOB | 0~4294967295  
TINYTEXT | 0~255  
TEXT | 0~65535  
MEDIUMTEXT | 0~167772150  
LONGTEXT | 0~4294967295  
VARBINARY(N) | 0~N个字节的变长字节字符集  
BINARY(N) | 0~N个字节的定长字节字符集  
  
### VARCHAR 和 CHAR 的区别是什么？分别适用的场景有哪些？

VARCHAR 和 CHAR 最大区别就是，VARCHAR 的长度是可变的，而 CHAR 是固定长度，CHAR 的取值范围为1-255，因此 VARCHAR
可能会造成存储碎片。由于它们的特性决定了 CHAR 比较适合长度较短的字段和固定长度的字段，如身份证号、手机号等，反之则适合使用 VARCHAR。

### MySQL 存储金额应该使用哪种数据类型？为什么？

MySQL 存储金额应该使用 `decimal` ，因为如果存储其他数据类型，比如 `float` 有导致小数点后数据丢失的风险。

### limit 3,2 的含义是什么？

去除前三条数据之后查询两条信息。

### now() 和 current_date() 有什么区别？

now() 返回当前时间包含日期和时分秒，current_date() 只返回当前时间，如下图所示：

![avatar](https://images.gitbook.cn/FgQmCmOxXBEjTyldp7YJFSJZGdid)

### 如何去重计算总条数？

使用 distinct 去重，使用 count 统计总条数，具体实现脚本如下：

> select count(distinct f) from t

### last _insert_ id() 函数功能是什么？有什么特点？

last _insert_ id() 用于查询最后一次自增表的编号，它的特点是查询时不需要不需要指定表名，使用 `select
last_insert_id()` 即可查询，因为不需要指定表名所以它始终以最后一条自增编号为主，可以被其它表的自增编号覆盖。比如 A 表的最大编号是
10，last _insert_ id() 查询出来的值为 10，这时 B 表插入了一条数据，它的最大编号为 3，这个时候使用 last _insert_
id() 查询的值就是 3。

### 删除表的数据有几种方式？它们有什么区别？

删除数据有两种方式：delete 和 truncate，它们的区别如下：

  * delete 可以添加 where 条件删除部分数据，truncate 不能添加 where 条件只能删除整张表；
  * delete 的删除信息会在 MySQL 的日志中记录，而 truncate 的删除信息不被记录在 MySQL 的日志中，因此 detele 的信息可以被找回而 truncate 的信息无法被找回；
  * truncate 因为不记录日志所以执行效率比 delete 快。

delete 和 truncate 的使用脚本如下：

> delete from t where username='redis'; truncate table t;

### MySQL 中支持几种模糊查询？它们有什么区别？

MySQL 中支持两种模糊查询：regexp 和 like，like 是对任意多字符匹配或任意单字符进行模糊匹配，而 regexp
则支持正则表达式的匹配方式，提供比 like 更多的匹配方式。 regexp 和 like 的使用示例如下： select * from person
where uname like '%SQL%';> select from person where uname regexp '.SQL*.';

### MySQL 支持枚举吗？如何实现？它的用途是什么？

MySQL 支持枚举，它的实现方式如下：

    
    
    create table t(
        sex enum('boy','grid') default 'unknown'
    );
    

枚举的作用是预定义结果值，当插入数据不在枚举值范围内，则插入失败，提示错误 `Data truncated for column 'xxx' at row
n` 。

### count(column) 和 count(*) 有什么区别？

count(column) 和 count() 最大区别是统计结果可能不一致，count(column) 统计不会统计列值为 null 的数据，而
count() 则会统计所有信息，所以最终的统计结果可能会不同。

### 以下关于 count 说法正确的是？

A. count 的查询性能在各种存储引擎下的性能都是一样的。 B. count 在 MyISAM 比 InnoDB 的性能要低。 C. count 在
InnoDB 中是一行一行读取，然后累计计数的。 D. count 在 InnoDB 中存储了总条数，查询的时候直接取出。

答：C

### 为什么 InnoDB 不把总条数记录下来，查询的时候直接返回呢？

因为 InnoDB 使用了事务实现，而事务的设计使用了多版本并发控制，即使是在同一时间进行查询，得到的结果也可能不相同，所以 InnoDB
不能把结果直接保存下来，因为这样是不准确的。

### 能否使用 show table status 中的表行数作为表的总行数直接使用？为什么？

不能，因为 show table status 是通过采样统计估算出来的，官方文档说误差可能在 40% 左右，所以 show table status
中的表行数不能直接使用。

### 以下哪个 SQL 的查询性能最高？

A. select count(*) from t where time>1000 and time<4500 B. show table status
where name='t' C. select count(id) from t where time>1000 and time<4500 D.
select count(name) from t where time>1000 and time<4500

答：B 题目解析：因为 show table status 的表行数是估算出来，而其他的查询因为添加了 where 条件，即使是 MyISAM
引擎也不能直接使用已经存储的总条数，所以 show table status 的查询性能最高。

### InnoDB 和 MyISAM 执行 select count(*) from t，哪个效率更高？为什么？

MyISAM 效率最高，因为 MyISAM 内部维护了一个计数器，直接返回总条数，而 InnoDB 要逐行统计。

### 在 MySQL 中有对 count(*) 做优化吗？做了哪些优化？

count(*) 在不同的 MySQL 引擎中的实现方式是不相同的，在没有 where 条件的情况下：

  * MyISAM 引擎会把表的总行数存储在磁盘上，因此在执行 count(*) 的时候会直接返回这个这个行数，执行效率很高；
  * InnoDB 引擎中 count(*) 就比较麻烦了，需要把数据一行一行的从引擎中读出来，然后累计基数。

但即使这样，在 InnoDB 中，MySQL 还是做了优化的，我们知道对于 count( _)
这样的操作，遍历任意索引树得到的结果，在逻辑上都是一样的，因此，MySQL
优化器会找到最小的那颗索引树来遍历，这样就能在保证逻辑正确的前提下，尽量少扫描数据量，从而优化了 count(_ ) 的执行效率。

### 在 InnoDB 引擎中 count(*)、count(1)、count(主键)、count(字段) 哪个性能最高？

count(字段)<count(主键 id)<count(1)≈count(*) 题目解析：

  * 对于 count(主键 id) 来说，InnoDB 引擎会遍历整张表，把每一行的 id 值都取出来，返回给 server 层。server 层拿到 id 后，判断是不可能为空的，就按行累加。
  * 对于 count(1) 来说，InnoDB 引擎遍历整张表，但不取值。server 层对于返回的每一行，放一个数字“1”进去，判断是不可能为空的，按行累加。
  * 对于 count(字段) 来说，如果这个“字段”是定义为 not null 的话，一行行地从记录里面读出这个字段，判断不能为 null，按行累加；如果这个“字段”定义允许为 null，那么执行的时候，判断到有可能是 null，还要把值取出来再判断一下，不是 null 才累加。
  * 对于 count(*) 来说，并不会把全部字段取出来，而是专门做了优化，不取值，直接按行累加。

所以最后得出的结果是：count(字段)<count(主键 id)<count(1)≈count(*)。

### MySQL 中内连接、左连接、右连接有什么区别？

  * 内连（inner join）— 把匹配的关联数据显示出来；
  * 左连接（left join）— 把左边的表全部显示出来，右边的表显示出符合条件的数据；
  * 右连接（right join）— 把右边的表全部显示出来，左边的表显示出符合条件的数据；

### 什么是视图？如何创建视图？

视图是一种虚拟的表，具有和物理表相同的功能，可以对视图进行增、改、查操作。视图通常是一个表或者多个表的行或列的子集。 视图创建脚本如下：

    
    
    create view vname as
    select column_names
    from table_name
    where condition
    

### 视图有哪些优点？

  * 获取数据更容易，相对于多表查询来说；
  * 视图能够对机密数据提供安全保护；
  * 视图的修改不会影响基本表，提供了独立的操作单元，比较轻量。

### MySQL 中“视图”的概念有几个？分别代表什么含义？

MySQL 中的“视图”概念有两个，它们分别是：

  * MySQL 中的普通视图也是我们最常用的 view，创建语法是 create view ...,它的查询和普通表一样；
  * InnoDB 实现 MVCC（Multi-Version Concurrency Control）多版本并发控制时用到的一致性读视图，它没有物理结构，作用是事务执行期间定于可以看到的数据。

### 使用 delete 误删数据怎么找回？

可以用 Flashback 工具通过闪回把数据恢复回来。

### Flashback 恢复数据的原理是什么？

Flashback 恢复数据的原理是是修改 binlog 的内容，拿回原库重放，从而实现数据找回。

* * *

## 限时福利

**加入作者群，与大佬面对面交流，同时还能获得最新面试经验。 赶紧添加小助手香香姐，微信ID「xiangcode」，发送暗号「6000」即可**

