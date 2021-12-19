## 表存储格式的指定

### 内置存储格式

Hive创建表时默认使用的格式为TextFile，当然内置的存储格式除了TextFile，还有sequencefile、rcfile、ORC、Parquet、Avro。

可以使用stored as inputformat、outputformat为表指定不同的存储格式，首先TextFile存储格式为：

    
    
    STORED AS INPUTFORMAT
    'org.apache.hadoop.mapred.TextInputFormat'
    OUTPUTFORMAT
    'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
    

但对于内置的存储格式，可以简写为stored as ，如TextFile可以直接指定为：

    
    
    CREATE TABLE <table_name> (<col_name> <data_type> [, <col_name> <data_type> ...])
    STORED AS TEXTFILE;
    

当然TextFile是Hive默认的存储格式，不使用stored as进行指定，则默认为TextFile。

对于其它存储格式的指定如下：

**SequenceFile：**

    
    
    CREATE TABLE <table_name> (<col_name> <data_type> [, <col_name> <data_type> ...])
    STORED AS SequenceFile;
    

**RCFile：**

    
    
    CREATE TABLE <table_name> (<col_name> <data_type> [, <col_name> <data_type> ...])
    STORED AS RCFILE;
    

**ORCFile：**

    
    
    CREATE TABLE <table_name> (<col_name> <data_type> [, <col_name> <data_type> ...])
    STORED AS ORCFILE;
    

**Parquet：**

    
    
    CREATE TABLE <table_name> (<col_name> <data_type> [, <col_name> <data_type> ...])
    STORED AS Parquet;
    

**Avro：**

    
    
    CREATE TABLE <table_name> (<col_name> <data_type> [, <col_name> <data_type> ...])
    STORED AS AVRO;
    

至于在生产环境中，在不同场景中应该选择哪种存储格式，之前已经讲过，就不再赘述。

### 自定义存储格式

除了这些内置的存储格式，也可以通过实现InputFormat和OutputFormat来自定义存储格式，需要编程来进行实现。

其中InputFormat定义如何读取数据，OutputFormat定义如何写出数据，除此之外，一般情况下，自定义存储格式还需要使用 ROW FORMAT
SERDE 指定数据序列化&反序列化方式。

Hive默认的序列化方式为 org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe。

    
    
    # 添加实现自定义格式的Jar包
    add jar {path to jar};
    
    # 创建表时指定自定义格式
    CREATE TABLE <table_name> (<col_name> <data_type> [, <col_name> <data_type> ...])
    ROW FORMAT SERDE '<SerName>'
    STORED AS INPUTFORMAT
    ‘<InputFormatName>'
    OUTPUTFORMAT
    ‘<OutputFormatName>';
    

## Hive表压缩

### 压缩算法

Hive支持的压缩方法有bzip2、gzip、deflate、snappy、lzo等。Hive依赖Hadoop的压缩方法，所以Hadoop版本越高支持的压缩方法越多，可以在$HADOOP_HOME/conf/core-
site.xml中进行配置：

    
    
    <property>
    <name>io.compression.codecs</name>
    <value>org.apache.hadoop.io.compress.GzipCodec,org.apache.hadoop.io.compress.DefaultCodec,com.hadoop.compression.lzo.LzoCodec,com.hadoop.compression.lzo.LzopCodec,org.apache.hadoop.io.compress.BZip2Codec
    </value>
    </property>
    

常见的压缩格式有：

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/7fa43093f6e670c0ab3cf9d5d54b2d4f.png#pic_center)

其中压缩比bzip2 > zlib > gzip > deflate > snappy > lzo >
lz4，在不同的测试场景中，会有差异，这仅仅是一个大概的排名情况。bzip2、zlib、gzip、deflate可以保证最小的压缩，但在运算中过于消耗时间。

从压缩性能上来看：lz4 > lzo > snappy > deflate > gzip >
bzip2，其中lz4、lzo、snappy压缩和解压缩速度快，压缩比低。

所以一般在生产环境中，经常会采用lz4、lzo、snappy压缩，以保证运算效率。

### Native Libraries

Hadoop由Java语言开发，所以压缩算法大多由Java实现；但有些压缩算法并不适合Java进行实现，会提供本地库Native
Libraries补充支持。Native Libraries除了自带bzip2, lz4, snappy,
zlib压缩方法外，还可以自定义安装需要的功能库（snappy、lzo等）进行扩展。

而且使用本地库Native Libraries提供的压缩方式，性能上会有50%左右的提升。

使用命令可以查看native libraries的加载情况：

    
    
    hadoop checknative -a
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/95a4715a7ad9cc76d2b09d6f3e3dfb89.png#pic_center)

完成对Hive表的压缩，有两种方式：配置MapReduce压缩、开启Hive表压缩功能。因为Hive会将SQL作业转换为MapReduce任务，所以直接对MapReduce进行压缩配置，可以达到压缩目的；当然为了方便起见，Hive中的特定表支持压缩属性，自动完成压缩的功能。

### MapReduce压缩配置

MapReduce支持中间压缩和最终结果压缩，可以在$$HADOOP _HOME/conf/mapred-site.xml或$$HADOOP_
HOME/conf/hive-site.xml中进行全局配置。当然也可以在beeline中直接使用set命令进行临时配置。

**中间压缩** 方式在Map阶段进行，可以减少shuffle中数据的传输量。

    
    
    -- 开启hive传输数据压缩功能
    set hive.exec.compress.intermediate=true;
    -- 开启mapreduce中map输出压缩功能
    set mapreduce.map.output.compress=true;
    -- 设置map输出时的压缩格式
    set mapreduce.map.output.compress.codec=org.apache.hadoop.io.compress.SnappyCodec;
    

这里指定了压缩方法为Snappy，压缩性能上会有很不错的表现。

指定中间压缩后，将ext _psn表中的数据拷贝到新表copress_ 1中：

    
    
    --为了在查看表文件内容时比较规整，规定了在存储时各字段间使用\t进行分割
    create table compress_1
    ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
    as select * from ext_psn;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/e563a943bd218041d4da0a706e9095ea.png#pic_center)

在执行过程中，会提高shuffle效率，但并不影响最终结果，表文件依然以非压缩方式存储：

    
    
    hadoop fs -ls /user/hive/warehouse/compress_1
    hadoop fs -cat /user/hive/warehouse/compress_1/000000_0
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/e76e602333397aa46422068031e5b439.png#pic_center)

**最终结果压缩** 在Reduce阶段进行，将结果输出到Hive表时压缩，以减少数据存储容量：

    
    
    -- 开启hive结果数据压缩功能
    set hive.exec.compress.output=true;
    -- 开启mapreduce中结果数据压缩功能
    set mapreduce.output.fileoutputformat.compress=true;
    -- 设置压缩格式
    set mapreduce.output.fileoutputformat.compress.codec =org.apache.hadoop.io.compress.SnappyCodec;
    -- 设置块级压缩，可选NONE\RECORD\BLOCK
    set mapreduce.output.fileoutputformat.compress.type=BLOCK;
    

压缩方式有NONE、RECORD、BLCOK；默认为RECORD，会为每一行进行压缩；设置为BLOCK时，会先将数据存储到1M的缓存中，然后再对缓存中的数据进行压缩，这种压缩方式会有更高的吞吐和性能。

指定最终结果压缩后，将ext _psn表中的数据拷贝到新表copress_ 2中：

    
    
    --为了在查看表文件内容时比较规整，规定了在存储时各字段间使用\t进行分割
    create table compress_2
    ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
    as select * from ext_psn;
    

可以看到，此时文件已经完成压缩了；当然，文件越大，压缩效果越明显。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/49b66c0ad3e0478e1b563e6ff9441c3c.png#pic_center)

查看最终表文件：

    
    
    hadoop fs -ls /user/hive/warehouse/compress_2
    hadoop fs -cat /user/hive/warehouse/compress_2/000000_0.snappy
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/ed4ae3d106381f92dc73aa8c70cf2633.png#pic_center)

可以发现，文件已经被压缩成以.snappy为后缀的压缩文件，数据也已经不再是明文保存。

在查询表文件时，Hive会自动对数据进行解压缩，以明文的方式显示结果：

    
    
    select * from compress_2;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/a3248de3d45588946b50db711da6660a.png#pic_center)

当然，除了Snappy压缩，还可以指定其它压缩方法，通过Hadoop支持的编码/解码器来完成。常见的压缩格式的编码/解码器有：

| **压缩格式** | **Hadoop压缩编码/解码器** | |:----|:----|:----|:----|
|DEFLATE|org.apache.hadoop.io.compress.DefaultCodec|
|Gzip|org.apache.hadoop.io.compress.GzipCodec|
|Bzip2|org.apache.hadoop.io.compress.BZip2Codec|
|LZO|com.hadoop.compression.lzo.LzopCodec|
|Snappy|org.apache.hadoop.io.compress.SnappyCodec|

在生产中，为了保证效率，使用LZO、Snappy压缩较多。

在数据导入时，Hive支持直接将gzip、bzip2格式的压缩文件导入到Textfile表中，会自动对压缩文件进行解析。

    
    
    --这里只是示例参考，可以手动创建gzip文件进行尝试导入
    CREATE TABLE raw (line STRING)
    ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' LINES TERMINATED BY '\n';
    
    LOAD DATA LOCAL INPATH '/tmp/weblogs/20090603-access.log.gz' INTO TABLE raw;
    

### Hive表压缩功能

除了直接配置MapReduce压缩功能外，Hive的ORC表和Parquet表直接支持表的压缩属性。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/7416500fd0d8bf5c70309ebba862d2b5.png#pic_center)

但支持的压缩格式有限，ORC表支持None、Zlib、Snappy压缩，默认为ZLIB压缩。但这3种压缩格式不支持切分，所以适合单个文件不是特别大的场景。使用Zlib压缩率高，但效率差一些；使用Snappy效率高，但压缩率低。

Parquet表支持Uncompress、Snappy、Gzip、Lzo压缩，默认不压缩Uncompressed。其中Lzo压缩是支持切分的，所以在表的单个文件较大的场景会选择Lzo格式。Gzip方式压缩率高，效率低；而Snappy、Lzo效率高，压缩率低。

#### ORC表压缩

ORC表的压缩，需要通过表属性orc.compress来指定。orc.compress的值可以为NONE、ZLIB、SNAPPY，默认为ZLIB。

首先创建一个非压缩的ORC表：

    
    
    create table compress_orc_none
    ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
    STORED AS orc
    tblproperties ("orc.compress"="NONE")
    as select * from ext_psn;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/885c57d81fc2044b6002c7e4de5f7634.png#pic_center)

在HDFS中，查看表文件大小。

    
    
    hadoop fs -ls -h /user/hive/warehouse/compress_orc_none
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/1d56acf5f218079ba13dd897f6b2b3bf.png#pic_center)

然后再创建一个使用SNAPPY压缩的ORC表：

    
    
    create table compress_orc_snappy
    ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
    STORED AS orc
    tblproperties ("orc.compress"="SNAPPY")
    as select * from ext_psn;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/08e1935b02bea971826a06a361c5b512.png#pic_center)

在HDFS中，查看压缩后的表文件大小。

    
    
    hadoop fs -ls -h /user/hive/warehouse/compress_orc_snappy
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/0d4c878995145560e70482a615df622b.png#pic_center)

这里文件大小没有变化，这是因为数据量较少，进行Snappy压缩时，反而增加了压缩头和尾的额外数据。数据量较大时，就会有明显的效果。

最后，使用默认压缩格式ZLIB的ORC表，进行对比：

    
    
    create table compress_orc_zlib
    ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
    STORED AS orc
    as select * from ext_psn;
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/670c58e05ec483be9e605314e6bd28fc.png#pic_center)

在HDFS中，查看压缩后的表文件大小。

    
    
    hadoop fs -ls -h /user/hive/warehouse/compress_orc_zlib
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/b69a984a72bc41c98443aa6e43f3320d.png#pic_center)

Zlib格式虽然压缩效率低，但压缩率很高，可以看到相对于None、Snappy格式，文件大小都有很明显的减少。

#### Parquet表压缩

Parquet表的压缩，通过表属性parquet.compression指定。值可以是Uncompressed、Snappy、Gzip、Lzo。默认是不进行压缩的Uncompressed。

首先创建一张普通Parquet表，默认为Uncompressed压缩格式：

    
    
    create table compress_parquet_none
    ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
    STORED AS parquet
    as select * from ext_psn;
    

在HDFS中，查看表文件大小。

    
    
    hadoop fs -ls -h /user/hive/warehouse/compress_parquet_none
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/2147f6b5a7e9a6262827b4ac0f87838f.png#pic_center)

然后创建压缩率较低，但效率较高的Snappy格式的Parquet表：

    
    
    create table compress_parquet_snappy
    ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
    STORED AS parquet
    tblproperties ("parquet.compress"="SNAPPY")
    as select * from ext_psn;
    

在HDFS中，查看压缩后的表文件大小。

    
    
    hadoop fs -ls -h /user/hive/warehouse/compress_parquet_snappy
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/ea0cd4538171b2311de55cc69395a512.png#pic_center)

数据量较小，大小没有发生变化。最后创建压缩率较高，但效率较低的Gzip格式的Parquet表：

    
    
    create table compress_parquet_gzip
    ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
    STORED AS parquet
    tblproperties ("parquet.compress"="GZIP")
    as select * from ext_psn;
    

在HDFS中，查看压缩后的表文件大小。

    
    
    hadoop fs -ls -h /user/hive/warehouse/compress_parquet_gzip
    

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/ba076e3965bf9db12abcffcf3ed789e4.png#pic_center)

在小数据量情况下，大小依然没有发生变化。虽然小数据量的参考意义不大，但基本能看出来，Parquet各压缩方式之间还是比较稳定的，而且整体要比ORC的压缩率要低。

#### 全局压缩配置

除了在建表时手动指定ORC、Parquet表的压缩格式的属性之外，也可以在执行建表语句前，使用set命令进行指定。

    
    
    --设置parquet表的压缩格式为SNAPPY
    set parquet.compression=SNAPPY;
    --设置orc表的压缩格式为SNAPPY
    set orc.compress=SNAPPY
    

当然，这意味着，在生产环境中，可以将参数直接全局配置到hive-site.xml文件中，来规定表的压缩格式。

    
    
    <property>
    <name>parquet.compression</name>
    <value>SNAPPY</value>
    </property>
    

