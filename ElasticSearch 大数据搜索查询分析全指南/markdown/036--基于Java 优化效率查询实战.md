在之前的课时中，我们优化查询效率一共有两个方面，一是对于 query 查询，如果想要一次性取大量数据，我们选择使用 Scroll
查询。当碰到聚合数据过大时候，我们介绍了两种：一种是 Partition 查询，一种是 Composite 查询。本节课中将会对以上的优化方式做一个
Java 的实现。

### Scroll 查询

**给定需求：** 查询天气索引中北京地区的所有数据。

Java 中一切皆是对象，所以对于返回的天气数据创建一个对应的 bean 类。

    
    
    public class weather {
        private String city;
        private int temperature;
        private Date timestamp;
    
        public String getCity() {
            return city;
        }
    
        public void setCity(String city) {
            this.city = city;
        }
    
        public int getTemperature() {
            return temperature;
        }
    
        public void setTemperature(int temperature) {
            this.temperature = temperature;
        }
    
        public Date getTimestamp() {
            return timestamp;
        }
    
        public void setTimestamp(Date timestamp) {
            this.timestamp = timestamp;
        }
    }
    

定义 weather_monitor 索引，设置 term 查询，字段设置成 city，条件设置成 beijing。配置一个合理的 size，这里我设置成
100，真正应用到需求中可以设置大一点。

  * `searchRequest.scroll` 配置 scroll 查询的接口
  * `TimeValue.timeValueMinutes(1L)` 设置快照过期时间为 1 分钟
  * `String scrollId = searchResponse.getScrollId();` 获取 scrollId 用来配置下次查询
  * `SearchHit[] resutls = hits.getHits();` 得到查询的结果

    
    
    SearchRequest searchRequest = new SearchRequest("weather_monitor");
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    searchSourceBuilder.query(termQuery("city.keyword", "beijing"));
    searchSourceBuilder.size(100);
    searchRequest.source(searchSourceBuilder);
    searchRequest.scroll(TimeValue.timeValueMinutes(1L));
    SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);
    String scrollId = searchResponse.getScrollId();
    SearchHit[] resutls = hits.getHits();
    

解析返回结果，这里使用 Java 8 的 Stream 接口来处理。返回 List 数据。

    
    
     List<Weather> collect = Stream.of(resutls).map(result -> {
                    Map<String, Object> source = result.getSourceAsMap();
                    Weather weather = new Weather();
                    weather.setCity(String.valueOf(source.get("city")));
                    weather.setTemperature((int) source.get("temperature"));
                    weather.setTimestamp(LocalDateTime.parse(String.valueOf(source.get("timestamp"))).toDate());
                    return weather;
                }).collect(Collectors.toList());
    

下一次 Scroll 查询，配置上一次获取的 scrollId 进行再一次查询：

    
    
    SearchScrollRequest scrollRequest = new SearchScrollRequest(scrollId); 
    scrollRequest.scroll(TimeValue.timeValueSeconds(30));
    SearchResponse searchScrollResponse = client.scroll(scrollRequest, RequestOptions.DEFAULT);
    scrollId = searchScrollResponse.getScrollId();  
    hits = searchScrollResponse.getHits(); 
    

### Partition 查询

**给定需求：** 不同城市温度的平均值。

首先要确定基数，使用 cardinality 接口，确定 city 个数只有 11 个：

    
    
    SearchRequest searchRequest = new SearchRequest();
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
                searchSourceBuilder.aggregation(AggregationBuilders.cardinality("city_count").field("city.keyword"));
    searchSourceBuilder.size(0);
    searchRequest.indices("weather*");
    searchRequest.source(searchSourceBuilder);
    SearchResponse search = client.search(searchRequest, RequestOptions.DEFAULT);
    System.out.println(search);
    

然后再设置 Partition，这里为了简单 num_partition 设置成 2，也就是把数据分成 2 份。

只需要加上 includeExclude，配置 Partition 与 num_partition 即可。

    
    
     TermsAggregationBuilder termsAggregationBuilder = AggregationBuilders.terms("city").includeExclude(new IncludeExclude(0,2))
                        .field("city.keyword").subAggregation(AggregationBuilders.avg("avg").field("temperature"));
    

完成代码如下，是不是简单方便，比直接写 DSL 方便多了。

    
    
     SearchRequest searchRequest = new SearchRequest();
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    TermsAggregationBuilder termsAggregationBuilder = AggregationBuilders.terms("city").includeExclude(new IncludeExclude(0,2))
                        .field("city.keyword").subAggregation(AggregationBuilders.avg("avg").field("temperature"));
    searchSourceBuilder.aggregation(termsAggregationBuilder);
    searchSourceBuilder.size(0);
    searchRequest.indices("weather*");
    searchRequest.source(searchSourceBuilder);
    SearchResponse search = client.search(searchRequest, RequestOptions.DEFAULT);
    System.out.println(search);
    

返回结果如下：

    
    
    {
        "aggregations":{
            "sterms#city":{
                "doc_count_error_upper_bound":0,
                "sum_other_doc_count":0,
                "buckets":[
                    {
                        "key":"nanjing",
                        "doc_count":9182,
                        "avg#avg":{
                            "value":14.901655412764104
                        }
                    },
          ......................
                ]
            }
        }
    }
    

### Composite 查询

我们在 29 课详细地介绍了 Composite 与 Partition 的区别，以及在什么样的条件下对两者进行选择。这里就不再赘述，直接介绍如何在
Java 中使用 Composite 查询接口。

配置一个 TermsValuesSourceBuilder 作为组合 Composite
的条件工具，AggregationBuilders.composite 接口传入参数后即可。

    
    
    SearchRequest searchRequest = new SearchRequest();
    SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
    TermsValuesSourceBuilder city = new TermsValuesSourceBuilder("city").field("city.keyword");
    CompositeAggregationBuilder composite = AggregationBuilders.composite("city", Arrays.asList(city)).subAggregation(AggregationBuilders.avg("avg").field("temperature"));;
    searchSourceBuilder.aggregation(composite);
    searchSourceBuilder.size(0);
    searchRequest.indices("weather*");
    searchRequest.source(searchSourceBuilder);
    SearchResponse search = client.search(searchRequest, RequestOptions.DEFAULT);
    System.out.println(search);
    

查询结果如下，如果需要下一次查询使用 composite.aggregateAfter() 配置 after_key 即可。

    
    
    {
      ............
      ,
        "aggregations":{
            "composite#city":{
                "after_key":{
                    "city":"suzhou"
                },
                "buckets":[
                    {
                        "key":{
                            "city":"beijing"
                        },
                        "doc_count":9131,
                        "avg#avg":{
                            "value":14.998904829701019
                        }
                    }
                  ...............
                ]
            }
        }
    }
    

### 小结

本课时主要介绍了如何使用 Java 的 ES 接口进行优化效率查询，涉及的查询方式有 Scroll 查询、Partition 查询、Composite
查询，详细介绍了接口的使用方式。

本课时一定要上手写代码，只有写出来运行出来后才能深入地理解接口的妙处，另外当你非常熟悉 DSL 语法的时候，会发现本课时会非常轻松。

总的来说 ES 的 Java 封装还是很优秀的，理解不同接口的封装方式、命名方式，有助你对面向对象编程理解的提升。

