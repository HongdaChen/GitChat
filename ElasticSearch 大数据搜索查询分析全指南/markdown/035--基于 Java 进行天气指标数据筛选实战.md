本课时我们将使用 ES Java 客户端实现相关的需求，以及如何使用 DSL 查询 ES。

### Search 查询

**给定需求：** 在天气索引中查询所有的数据，但是只返回一条。

SearchRequest 是 ES 对 Search 查询的封装，使用 SearchSourceBuilder 来配置 DSL
相关的参数，这里我们只返回一条结果，size 设置为 1，索引设置为 weather*。

    
    
    SearchRequest searchRequest = new SearchRequest();
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    searchSourceBuilder.query(QueryBuilders.matchAllQuery());
    searchSourceBuilder.size(1);
    searchRequest.indices("weather*");
    searchRequest.source(searchSourceBuilder);
    SearchResponse search = client.search(searchRequest, RequestOptions.DEFAULT);
    System.out.println(search);
    

这里运行时可能会报错：

    
    
     org.elasticsearch.action.search.SearchRequest.isCcsMinimizeRoundtrips()Z
    

使用下面的 Maven，将 elasticsearch-rest-high-level-client 包里面依赖的 elasticsearch 与
elasticsearch-rest-client 移除，重新引入 7.1 版本，即可解决这个问题。

    
    
    <dependency>
                <groupId>org.elasticsearch</groupId>
                <artifactId>elasticsearch</artifactId>
                <version>7.1.1</version>
            </dependency>
            <dependency>
                <groupId>org.elasticsearch.client</groupId>
                <artifactId>elasticsearch-rest-client</artifactId>
                <version>7.1.1</version>
            </dependency>
            <dependency>
                <groupId>org.elasticsearch.client</groupId>
                <artifactId>transport</artifactId>
                <version>7.1.1</version>
            </dependency>
            <dependency>
                <groupId>org.elasticsearch.client</groupId>
                <artifactId>elasticsearch-rest-high-level-client</artifactId>
                <version>7.1.1</version>
                <exclusions>
                    <exclusion>
                        <groupId>org.elasticsearch</groupId>
                        <artifactId>elasticsearch</artifactId>
                    </exclusion>
                    <exclusion>
                        <groupId>org.elasticsearch.client</groupId>
                        <artifactId>elasticsearch-rest-client</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
    

返回结果如下：

    
    
    {
        "took":72,
        "timed_out":false,
        "_shards":{
            "total":1,
            "successful":1,
            "skipped":0,
            "failed":0
        },
        "hits":{
            "total":{
                "value":10000,
                "relation":"gte"
            },
            "max_score":1,
            "hits":[
                {
                    "_index":"weather_monitor",
                    "_type":"_doc",
                    "_id":"XacPAnEBhYrWUVXu-nN9",
                    "_score":1,
                    "_source":{
                        "weather":"rainy",
                        "temperature":12,
                        "city":"shanghai",
                        "timestamp":"2020-03-17T21:27:42.554221"
                    }
                }
            ]
        }
    }
    

**给定需求：** 查询北京地区，3 月 7 号到 3 月 22 号天的天气状况。

这里我们用 term 查询筛选北京地区的，然后再用 range 查询一定范围内的数据。

这里使用 QueryBuilders 来构造查询条件，size 大小设置成了 10：

    
    
    LocalDate end = LocalDate.parse("2020-03-22");
    LocalDate start = LocalDate.parse("2020-03-07");
    SearchRequest searchRequest = new SearchRequest();
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    searchSourceBuilder.query(QueryBuilders.termQuery("city.keyword", "beijing"));
               searchSourceBuilder.query(QueryBuilders.rangeQuery("timestamp").lte(end).gte(start));
    searchSourceBuilder.size(10);
    searchRequest.indices("weather*");
    searchRequest.source(searchSourceBuilder);
    SearchResponse search = client.search(searchRequest, RequestOptions.DEFAULT);
    System.out.println(search);
    

上面的查询语句将会返回北京地区文档的所有字段，但是我们可能只是需要天气温度的字段，其他的字段希望能够过滤掉，作为网络传输优化的一种方式。

设置 fetchSource 为 false 可以过滤掉所有的字段：

    
    
    searchSourceBuilder.fetchSource(false);
    

可以通过 fetchSource 方法指定只返回温度字段，配置一个需要哪些字段，与哪些字段要过滤掉：

    
    
    String[] includeFields = new String[]{"temperature"};
    String[] excludeFields = new String[]{};
    searchSourceBuilder.fetchSource(includeFields, excludeFields);
    

返回结果如下：

    
    
    {
        "took":111,
        "timed_out":false,
        "_shards":{
            "total":1,
            "successful":1,
            "skipped":0,
            "failed":0
        },
        "hits":{
            "total":{
                "value":10000,
                "relation":"gte"
            },
            "max_score":1,
            "hits":[
                {
                    "_index":"weather_monitor",
                    "_type":"_doc",
                    "_id":"QqcPAnEBhYrWUVXu-WcU",
                    "_score":1,
                    "_source":{
                        "temperature":2
                    }
                },
                ...................
            ]
        }
    }
    

### 聚合分析查询

**给定需求：** 查询全国地区 3 月 7 号到 3 月 22 号天气温度的平均值。

query 查询设置时间 range 条件，再对温度进行聚合取均值。AggregationBuilders 用来封装聚合操作，聚合别名设置为
city，聚合字段选用 city.keyword。source 的 size 设置为 0，不让它返回 source 数据。

    
    
    LocalDate end = LocalDate.parse("2020-03-22");
    LocalDate start = LocalDate.parse("2020-03-07");
    SearchRequest searchRequest = new SearchRequest();
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    TermsAggregationBuilder termsAggregationBuilder = AggregationBuilders.terms("city")
                 .field("city.keyword").subAggregation(AggregationBuilders.avg("avg").field("temperature"));
    searchSourceBuilder.query(QueryBuilders.rangeQuery("timestamp").lte(end).gte(start));
    searchSourceBuilder.aggregation(termsAggregationBuilder);
    searchSourceBuilder.size(0);
    searchRequest.indices("weather*");
    searchRequest.source(searchSourceBuilder);
    SearchResponse search = client.search(searchRequest, RequestOptions.DEFAULT);
    System.out.println(search);
    

返回数据如下：

    
    
    {
        "took":353,
        "timed_out":false,
        "_shards":{
            "total":1,
            "successful":1,
            "skipped":0,
            "failed":0
        },
        "hits":{
            "total":{
                "value":10000,
                "relation":"gte"
            },
            "max_score":null,
            "hits":[
    
            ]
        },
        "aggregations":{
            "sterms#city":{
                "doc_count_error_upper_bound":0,
                "sum_other_doc_count":17656,
                "buckets":[
                    {
                        "key":"nanjing",
                        "doc_count":1819,
                        "avg#avg":{
                            "value":15.221000549752612
                        }
                    }
                  .............
                ]
            }
        }
    }
    

### 多层嵌套聚合查询

多层嵌套聚合需求经常能够遇到，我们使用天气指标数据为例，设计一个多层嵌套的聚合需求。

**给定需求：** 查询不同城市 3 月 7 号到 3 月 22 号，不同天气出现的次数。

对于这个需求，我们需要先对城市进行聚合，然后再对天气进行聚合，就可以得到不同地区不同天气在 15 天内出现的次数。

先对 city 进行 terms 聚合，然后再对 weather 进行子聚合，配置 fetchSource 为 false 不返回任何 source
数据，query size 设置为 0，最后配置时间范围条件。

    
    
    LocalDate end = LocalDate.parse("2020-03-22");
    LocalDate start = LocalDate.parse("2020-03-07");
    SearchRequest searchRequest = new SearchRequest();
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    TermsAggregationBuilder termsAggregationBuilder = AggregationBuilders.terms("city")
                        .field("city.keyword");
                termsAggregationBuilder.subAggregation(AggregationBuilders.terms("weather").field("weather.keyword"));
    searchSourceBuilder.aggregation(termsAggregationBuilder);
    searchSourceBuilder.fetchSource(false);
    searchSourceBuilder.size(0);
    searchRequest.source(searchSourceBuilder);
    SearchResponse search = client.search(searchRequest, RequestOptions.DEFAULT);
                searchSourceBuilder.query(QueryBuilders.rangeQuery("timestamp").lte(end).gte(start));
                System.out.println(search);
    

返回结果如下：

    
    
    {
        "took":267,
        "timed_out":false,
        "_shards":{
            "total":15,
            "successful":15,
            "skipped":0,
            "failed":0
        },
        "hits":{
            "total":{
                "value":10000,
                "relation":"gte"
            },
            "max_score":null,
            "hits":[
    
            ]
        },
        "aggregations":{
            "sterms#city":{
                "doc_count_error_upper_bound":0,
                "sum_other_doc_count":8940,
                "buckets":[
                    {
                        "key":"nanjing",
                        "doc_count":9182,
                        "sterms#weather":{
                            "doc_count_error_upper_bound":0,
                            "sum_other_doc_count":0,
                            "buckets":[
                                {
                                    "key":"rainy",
                                    "doc_count":3088
                                },
                                {
                                    "key":"fine",
                                    "doc_count":3048
                                },
                                {
                                    "key":"cloudy",
                                    "doc_count":3046
                                }
                            ]
                        }
                    },
                    ...............................
                ]
            }
        }
    }
    

### 小结

本课时主要介绍如何使用 ES Java 接口进行 query 查询与聚合查询；其中踩了一个包版本不匹配的坑，我们通过新配置 Maven 包得到解决，总体来说
ES 提供的接口是完全面向对象的，优点与缺点并存。

优点是不用抒写大量的 DSL，因为都被封装好了，特别如果 DSL
需要很多变量来配置过滤条件，面向对象编程方法会更方便，因为只需要按照面向对象编程的思想去编写查询聚合逻辑。

缺点个人感觉有点封装过度了，有时候简单的 DSL 加上 HTTP 可能会更简单，将 DSL
用面向对象的思想表达出来无疑会多了很多业务逻辑代码，增加了复杂度。

