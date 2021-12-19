本节课是个赠送课时，因为 ES 接口更新太快，有的接口很快就过期了，另外一些读者也不想在熟悉 Java-ES
接口上投入太多时间。那么这节课时就针对如果接口过期或者不可用怎么办？想了解一个公共技术来与 ES 交互。

### ES 交互

因为 ES 是基于 HTTP 的，所以我们可以通过 HTTP 框架直接与 ES 交互，获取数据。这样就可以解决接口过期，或者用公共技术不用学习 Java-
ES 接口来操作 ES。这里推荐 OkHttpClient 框架，这个框架是开源的而且听说效率一流。

在 Maven 中引入包：

    
    
          <!-- https://mvnrepository.com/artifact/com.squareup.okhttp3/okhttp -->
            <dependency>
                <groupId>com.squareup.okhttp3</groupId>
                <artifactId>okhttp</artifactId>
                <version>4.2.2</version>
            </dependency>
    

这里我创建了一个 HttpHelper 工具类，用来封装对 ES 的 post 请求。

需要注意的有以下几点：

  1. OkHttpClient client 一定要放在方法外，这样可以节约资源，放置重复创建连接池。
  2. 一定要配置超时时间，这里我配置成了 1 分钟。

    
    
    public class HttpHelper {
        private static OkHttpClient client = new OkHttpClient.Builder()
                .connectTimeout(1L, TimeUnit.MINUTES)
                .readTimeout(1L, TimeUnit.MINUTES)
                .build();
        private static final MediaType jsonMediaType = MediaType.parse("application/json; charset=utf-8");
    
        public static String post(String url, String dsl) throws IOException {
            RequestBody body = RequestBody.create(dsl, jsonMediaType);
            Request request = new Request.Builder()
                    .url(url)
                    .post(body)
                    .build();
    
            Response response = client.newCall(request).execute();
            String result = response.body().string();
            return result;
        }
    }
    

接口使用方式如下。

配置好 URL，指定 ES 地址，指定查询哪个索引，加上 _search，表示将使用 search rest API。

    
    
        String url="http://localhost:9200/weather*/_search";
            String dsl="{\n" +
                    "  \"size\": 1,\n" +
                    "  \"query\": {\n" +
                    "    \"match_all\": {}\n" +
                    "  }\n" +
                    "  \n" +
                    "}";
            String post = HttpHelper.post(url, dsl);
            System.out.println(post);
    

查询结果如下：

    
    
    {
        "took":36,
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
                    "_id":"QacPAnEBhYrWUVXu-WcU",
                    "_score":1,
                    "_source":{
                        "weather":"fine",
                        "temperature":-8,
                        "city":"nanjing",
                        "timestamp":"2020-02-28T19:57:23.990666"
                    }
                }
            ]
        }
    }
    

### 小结

本节课中介绍了如何使用 HTTP 与 ES 进行交互，与上节课时使用 Java-ES 接口来说学习成本低，而且代码简单可读性强。只要配置好
URL，你的精力将会被引入到如何创建 DSL，不用去关注其他的问题。

但是缺点也明显，就是如果涉及到经常需要改变的 DSL，动态更改起来会比较麻烦，但是总比 [Java High Level REST
Client](https://www.elastic.co/guide/en/elasticsearch/client/java-
rest/7.1/java-rest-high.html) 为每个请求都封装一个类要简单，而且需要我们投入一定的时间去熟悉这个接口的使用。所以当你的代码对
ES DSL 改动不大的时候可以使用 HTTP 工具，这样你的代码会变得简单易读，可维护性也高。

