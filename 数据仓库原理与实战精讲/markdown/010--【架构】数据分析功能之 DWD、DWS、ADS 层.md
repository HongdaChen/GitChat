### 数据明细层（DWD）

数据明细层对 ODS 层的数据除了进行清洗、标准化之外，还会进行维度退化。

维度是指对表中数据的一种组织方式，如时间、分类、地域；这些维度属性，在业务数据库中，会被拆分成多张表进行存储，这些表被称为维度表。

如下图所示，商品表和它的分类维度表（一、二、三级分类），按照范式标准一共拆分成了 4 张表。

![](https://images.gitbook.cn/5b614630-ef82-11ea-80b6-61caae27bd5a)

但在数据仓库产品中，一旦涉及到 join
关联操作，会消耗大量的资源，且降低运行的速度。所以会选择增加冗余，将这些维度表合并到主表中形成宽表。这种操作被称为维度退化。

![](https://images.gitbook.cn/6cd33770-ef82-11ea-b2cc-b183c5e37897)

而且一些大型的企业，在全国各地都开设了分公司，每个分公司的业务数据库只记录当前地区的数据，于是虽然数据内容相同，但地域不同，就形成了多张表。这些数据被汇总起来的时候，也需要对地域维度进行维度退化，增加一个地域字段，如
City，然后将所有地区的表汇总成一张表，从而提升之后的运算性能。

![](https://images.gitbook.cn/78e4d0f0-ef82-11ea-889f-9de3f88b5342)

### 数据汇总层（DWS）

数据汇总层的数据对数据明细层的数据，按照分析主题进行计算汇总，存放便于分析的宽表。

![](https://images.gitbook.cn/c14987f0-ef82-11ea-813e-7720407984bb)

在汇总时，会进行大量的运算，如 join、聚合操作。最后存储的宽表并非
3NF，而是注重数据聚合，复杂查询、处理性能更优的数仓模型，如在传统数仓中的维度模型，在大数据数仓中的宽表模型。

数据汇总层（DWS）是数据仓库设计的核心，数据仓库的面向主题特性、模型设计都在这个层中进行，目的就是为数据分析提供更优异的性能。

### 数据应用层（ADS）

数据应用层也被称为数据集市，对数据汇总层的数据进行分析后，结果数据会提供给外部系统使用（报表系统、业务系统）；而数据仓库的使用场景是分析计算，外部系统更注重的是查询、交互速度，且让外部系统直接对接数据仓库，数仓的安全性、稳定性会降低，频繁的查询也会给数仓带来压力。

所以需要专用的 ADS 层来存储结果数据，并为外部系统提供访问接口，提供更快的查询、交互速度。一般而言，ADS
层会使用单独的数据产品，它注重的功能不同，所以也被称为数据集市，与数据仓库做区分。

![](https://images.gitbook.cn/1730d820-ef84-11ea-813e-7720407984bb)

ADS 可能会由多个产品组成，来满足业务系统要求的不同功能。如果是报表决策类的业务，一般使用
Kylin，提供更快的交互查询速度；如果业务系统要求对结果进行并发查询，则使用 HBase；如果需要更智能的搜索检索，则使用 ElasticSearch。

当然如果数据量较小，或者为了节省成本，也可以将结果数据导出到传统数据库中，如 MySQL，以提升整体的查询性能。

