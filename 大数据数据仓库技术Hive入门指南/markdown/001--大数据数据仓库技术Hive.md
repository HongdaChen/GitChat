## 基本概念

### 诞生背景

在已经存在分布式计算引擎MapReduce的情况下，为什么会诞生Hive这样的产品？其实主要还是因为易用性问题。虽然MapReduce提供了分布式开发的能力，但它毕竟是一个通用计算引擎，在特定且相对成熟的垂直场景中，易用性就比较差了。

而在传统数据分析中，最常见的还是结构化数据，这个场景有它成熟的分析工具——SQL。数据量达到某个量级之后，单机或MPP数据库无法承受其负载，势必要转向大数据平台；但数据迁移完成后，因为大数据有自己的计算引擎（如Mapreduce），所以之前所有使用SQL编写的分析任务，都需要重构为MapReduce任务，这个工作量极大，迁移成本极高。而且迁移之后，对结构化数据的分析，也不能再使用SQL这种方便的工具来进行了，需要学习MapReduce语法，学习成本也很大。

那可不可以将特定领域，已经成熟的语法和使用习惯，如结构化数据分析的SQL，也迁移到大数据平台上来？当然可以，而且在大数据产品中，都是致力于此，用于提升大数据在不同场景的易用性。在结构化数据分析，即数据仓库场景中，可以将SQL自动转化为MapReduce任务的，在Hadoop家族中，最常用的便是Hive了。

### 什么是Hive？

Hive早期由Facebook开发，后来由Apache软件基金会开发，并成为其顶级项目。它是基于Hadoop的一个数据仓库工具。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/47ba01b5e31adceb26327693e7e0b241.png#pic_center)

它的核心思想是将迁移到HDFS中的结构化的数据文件，映射为一张数据库表，并提供sql查询、分析功能，可以将sql语句转换为MapReduce任务进行运行，也支持转换为Spark任务，而且官方推荐底层转换为Spark进行运算。当然Hive对原生SQL的支持率并不高，大概在60%左右，它有自己特定的语法HQL（Hive
SQL)。但即便如此，在结构化数据分析中，已经带来极大的便捷了。

## 原理

### Hive架构

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/e57ba71b0f308b32f6bd6a3972a825ae.png#pic_center)

Hive架构主要由3部分构成，Client、Driver、MetaStore。其中Client客户端用于任务的提交，包含Shell客户端Beeline、Hive
Cli，还有JDBC客户端；Driver会对提交的任务进行解析，将HiveSQL转换为MapReduce作业，它包含解释器（Antlr）、编译器、优化器、执行器，共同完成HQL查询语句从语法分析、编译成逻辑查询计划、优化以及转换成物理查询计划的过程；在转换过程中，因为SQL处理的是结构化数据，而Hive的数据是以文件的形式保存在HDFS中的，所以要将文件映射为二维表结构，MetaStore中存储的元数据信息（表类型、属性、字段、权限等）便可以辅助完成数据映射的功能，一般使用Mysql或Derby作为MetaStore的数据存储。

最终通过Driver转换成的MapReduce任务，会提交到底层的Hadoop平台上进行运算，返回运算结果。

### 搭建模式

Hive提供三种搭建模式：本地模式、单用户模式、多用户/远程服务器模式。

#### 本地模式

本地模式最为简单，仅适用于测试环境。在启动时，直接使用Hive CLI客户端，它自带Driver，而且连接到一个内置的In-memory
的数据库Derby作为MetaStore。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/42925e4d0981a951a6ee628c41ad7e77.png#pic_center)

#### 单用户模式

单用户模式会部署MetaStore，一般使用MySQL，但Driver依然集成在Hive Cli中，因为没有权限管控功能，所以只适合单用户场景。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/b42cca778bc6e593f8f37166d120c795.png#pic_center)

#### 多用户模式/远程服务器模式

在生产环境中一般使用这种部署模式，在服务节点启动Thrift
Server——HiveServer2，它集成了Driver和权限管控功能，单独对外提供服务；因为具有权限管控，所以适合多用户场景。MetaStore也进行单独部署，一般使用Mysql。

在这种部署模式下，Hive Cli已经不推荐使用了，而使用Beeline/JDBC客户端与HiveServer2进行连接与交互。

![在这里插入图片描述](https://img-
blog.csdnimg.cn/img_convert/9bc1f7845ad98f09f343ca03b8db4f65.png#pic_center)

了解Hive的基本情况之后，我们正式开始进行环境搭建，开始学习Hive的基本使用，这里我们选择多用户模式进行搭建。

