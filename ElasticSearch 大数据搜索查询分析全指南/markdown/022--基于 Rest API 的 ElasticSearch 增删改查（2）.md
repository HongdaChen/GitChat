### _serach 操作

Mget 只能通过文档 id 来查询文档，如果需要其他复杂的条件查询明显不能够满足需求。ES 提供了 _search API
支持更复杂的查询。ElasticSearch 查询共分为两种方式，一种是基于 URI 查询，另一种是基于 POST 查询。

第一种是 URI 查询，这种查询方式很常见，将查询条件参数与 URI 放到一起。测试数据如下，向索引 class 插入 3 条文档。

    
    
    PUT class/_doc/1
    {
      "name":"xiaoming",
      "sex":"man",
      "age":16
    }
    PUT class/_doc/2
    {
      "name":"xiaohua",
      "sex":"man",
      "age":17
    }
    PUT class/_doc/3
    {
      "name":"xiaohong",
      "sex":"female",
      "age":17
    }
    PUT class/_doc/4
    {
      "name":"ziqi deng",
      "sex":"man",
      "age":16
    }
    PUT class/_doc/5
    {
      "name":"lanlan deng",
      "sex":"man",
      "age":16
    }
    

![image-20200107205024004](https://images.gitbook.cn/2020-04-07-063323.png)

#### **给定需求：查找姓名是 xiaohua 的文档**

使用 q 指定查询条件，`name=xiaohua`；使用 GET 关键字，指定查询的索引 class，使用 _search 来查询。

    
    
    GET class/_search?q=name:xiaohua
    

![image-20200107203344744](https://images.gitbook.cn/2020-04-07-063325.png)

如果想要查询 xiaohua 和 xiaohong 两个人的文档呢？

### 字段 OR 查询

在 name 后面加个空格就行，或者指定布尔关键字 OR，下面的两种的方法都能够查询两个人的文档。

    
    
    GET class/_search?q=name:xiaohua xiaohong
    GET class/_search?q=name:(xiaohua OR xiaohong)
    

![image-20200107203934129](https://images.gitbook.cn/2020-04-07-063327.png)

虽然上述的命令的查询结果是我们想要的。但是这两个命令的查询意义却不同。

这条查询命令，将会查询 name 字段中包含 xiaohua 的文档，同时也会查询文档中任意字段包含 xiaohong 的文档。

因此 xiaohong 这个条件没有限定给 name 字段。

    
    
    GET class/_search?q=name:xiaohua xiaohong
    

下面这个查询语句将能够更好的便于理解：查询所有文档中包含 deng 的文档，我并没有给 deng 这个条件限定给name字段或者其他字段。

    
    
    GET class/_search?q=deng
    

![image-20200107213100067](https://images.gitbook.cn/2020-04-07-63328.png)

小结：使用 URI 查询时候，要区分有括号与无括号查询的异同，因此要想查询名字是 xiao hong 或者 xiao hua
只需要使用括号将两者括起来，加不加 OR 都一样，因为默认是 OR。

### 字段 AND 查询

这种方式显然不能够查询名称是 ziqi deng，使用 OR 查询结果如下：将会返回两个人的结果，并且给出想要的匹配分数。ES
会对查询结果进行打分，并按照分数大小进行排序返回结果。

造成这个结果的原因是，ES 默认根据空格分开单词，并且按照全文检索的方式，检索所有文档中 name 字段包含 ziqi 或者
deng。因此查询到了下面两条结果。

![image-20200107204949820](https://images.gitbook.cn/2020-04-07-063328.png)

因此使用 AND 查询方式如下：下面两种写法都能够达到效果。

    
    
    GET class/_search?q=name:"ziqi  deng"
    GET class/_search?q=name:(ziqi AND deng)
    

![image-20200107205741514](https://images.gitbook.cn/2020-04-07-063329.png)

如果我再插入一个文档，再执行上面的命令，查询结果将不会相同：

    
    
    PUT class/_doc/6
    {
      "name":"ziqi 2 deng",
      "sex":"man",
      "age":16
    }
    

这个命令只能查到一条，那就是我们想要的结果：

    
    
    GET class/_search?q=name:"ziqi  deng"
    

但是这一条却把文档为 ziqi 2 deng 也查询到了。

    
    
    GET class/_search?q=name:(ziqi AND deng)
    

![image-20200107210610371](https://images.gitbook.cn/0dcc41b0-7d7f-11ea-9792-81939fbf7f0c)

### Term 与 Phrase 查询

ES 在对某个字段进行查询时，会分为两种查询方式，Term 与 Phrase 。简单来说就是把文本查询条件看做是一个“词”，还是“多个词”。

如果想要查询到 ziqi deng 这个人，那么就是把 ziqi deng 看做一个词，那么应该使用 Term 查询，使用双引号括起来即可。否则就是
Phrase 查询，看做两个词，你可以对这两个词选择使用 OR 或者 AND 查询。ES
提供了“或非”，那么必然也会提供“否查询（NOT）”。我们可以通过否查询，排除掉名称中姓 deng 的文档。

    
    
    GET class/_search?q=name:(NOT deng)
    

![image-20200107211521955](https://images.gitbook.cn/2020-04-07-063330.png)

对于 Term 查询比较简单，严格匹配上就行，对于 Phrase 查询更加灵活，可以对查询条件任意的组合 **OR AND NOT**
，需要注意的是，你需要给查询条件加上括号，并且一定要大写。简而言之，Term 与 Phrase
查询是你如何看待查询条件的，是一个整体？还是多个部分？这两个概念与传统数据库对比，就是 SQL 语句中的 like 模糊查询，与精确值查询的区别。

### 范围查询

**给定需求：** URI 中也支持范围查询，查询年龄小于 17 的文档，在 15 到 17 之间的，在大于等于 16 到 17 之间的。

    
    
    GET class/_search?q=age:<17
    GET class/_search?q=age:(>15  AND  <17)
    GET class/_search?q=age:(>=16  AND  <17)
    

![image-20200107214653087](https://images.gitbook.cn/2020-04-07-063331.png)

**小结：** 本节主要介绍了如何使用 URL 进行 Term 与 Phrase 查询，这对理解 ES 查询作用非常大，后面的课会介绍如何使用 DSL
进行高阶查询。

