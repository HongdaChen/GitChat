ES 也提供了 Java 的接口。Java 是目前非常火爆流行的语言，本课时将主要介绍如何使用 ES 的 Java 接口，对 ES 进行操作，我们将对
Java Rest Client 的 Java High Level Rest Client 作为开发工具进行介绍。

我使用 IDEA 社区版作为开发工具，创建了一个 Spring Boot 项目。创建细节这里不过多介绍。

### Maven 依赖

Maven 是个非常好用的包管理器工具，在 pom.xml 添加下面的 Maven 依赖，就可以将 ES 客户端引入到现在的开发环境中：

    
    
    <dependency>
        <groupId>org.elasticsearch.client</groupId>
        <artifactId>elasticsearch-rest-high-level-client</artifactId>
        <version>7.1.1</version>
    </dependency>
    

### 配置连接客户端

配置好 IP 地址与端口号，连接成功后，需要手动地关闭连接：

    
    
    RestHighLevelClient client = new RestHighLevelClient(
            RestClient.builder(
                    new HttpHost("localhost", 9200, "http")));
    client.close();
    

### JSON 定义方式

直接在开发中写 JSON 字符串感觉比较 low，我个人喜欢两种方式定义 JSON 对象——Map 和 XContentBuilder。Map
的方式定义起来确实很方便，而且可读性也很高；但是如果需要嵌套的关系就显得非常麻烦，特别是定义 DSL 的时候，多层嵌套聚合用 Map 表达起来就很繁琐。

    
    
    Map<String, Object> jsonMap = new HashMap<>();
    jsonMap.put("user", "kimchy");
    jsonMap.put("postDate", new Date());
    jsonMap.put("message", "trying out Elasticsearch");
    

因此我更中意 XContentBuilder，一个 ES 定义的 JSON helper 类。用这个 helper 类，感觉就是在定义
JSON，嵌套表达起来也合理。后面将会使用 XContentBuilder 来作为数据对象或者 DSL 的 JSON 定义工具。

    
    
    XContentBuilder builder = jsonBuilder()
            .startObject()
            .field("user", "kimchy").endObject()
            .field("postDate", new Date())
            .field("message", "trying out Elasticsearch")
            .endObject().humanReadable(true);
    String s = Strings.toString(builder);
    

### Index 操作

Java 中封装了 IndexRequest 类，专门用来索引操作，先定义一个 JSON 对象，向 ES 中插入。

定义文档 ID，将 builder 或者 Map 数据放到 source 中，这里的 source 与 ES 中的 _source 字段含义差不多，指定
_source 里面会有哪些字段。定义好文档的 type 类型为 _doc，索引定义为 java_client。索引操作所需要的定义都已经包装到
IndexRequest 里面了，但是 IndexRequest 并不能发起请求，实际上它只是对请求索引操作的一个数据包装，真正 HTTP 请求 ES
的还是 client 对象。

    
    
    IndexRequest indexRequest = new IndexRequest()
            .id("1").source(builder);
    
    indexRequest.type("_doc");
    indexRequest.index("java_client");
    

发起 HTTP 请求，索引数据：

    
    
    IndexResponse indexResponse = client.index(indexRequest, RequestOptions.DEFAULT); System.out.println(indexResponse.toString());
    

打印结果为：

    
    
    IndexResponse[index=java_client,type=_doc,id=1,version=7,result=updated,seqNo=6,primaryTerm=1,shards={"total":2,"successful":2,"failed":0}]
    

在索引模式里面定义一个 java* 模式用来筛选我们刚插入的数据，然后取 discover 中查看，可以看到已经索引插入数据成功。

![image-20200504112349891](https://images.gitbook.cn/651e4ea0-9f70-11ea-a506-f32f5295a5a9)

### Get 操作

ES 为 Get 封装了一个 GetRequest 接口，将 Get 所能用到的参数都定义到里面去了。指定索引为 java_client，文档的 id 为
1。

    
    
    GetRequest getRequest = new GetRequest();
    getRequest.id("1");
    getRequest.index("java_client");
    

发送 GET 请求到 ES，GetResponse 用来封装返回的结果。

    
    
    GetResponse documentFields = client.get(getRequest, RequestOptions.DEFAULT);
    System.out.println(getRequest.toString());
    

GetResponse 返回结果如下：

    
    
    {
        "_index":"java_client",
        "_type":"_doc",
        "_id":"1",
        "_version":9,
        "_seq_no":8,
        "_primary_term":1,
        "found":true,
        "_source":{
            "bool":{
                "user":"kimchy"
            },
            "postDate":"2020-05-04T03:36:31.699Z",
            "message":"trying out Elasticsearch"
        }
    }
    

### Update 操作

ES 用 UpdateRequest 对 Update 操作进行了封装，之前也提到过 Index 文档跟 Update 文档的区别，Index
一个文档，如果这个文档已经存在那么就会删除这个文档，在重新索引文档，然后版本号加一。Update
的话这个文档必须存在，否则更新失败，如果更新成功后，版本号加一。

首先定义 UpdateRequest指定要更新的索引、id、type 与需要更新的内容，这里我把用户的名称修改为 xiaoming。

    
    
    UpdateRequest request = new UpdateRequest();
    request.index("java_client");
    request.id("1");
    XContentBuilder builder1 = jsonBuilder()
                        .startObject()
                        .startObject("bool")
                        .field("user", "xiaoming").endObject()
                        .endObject().humanReadable(true);
    
    request.doc(builder1);
    request.type("_doc");
    UpdateResponse update = client.update(request, RequestOptions.DEFAULT);
    System.out.println(update);
    

Response 输出结果如下：

    
    
    UpdateResponse[index=java_client,type=_doc,id=1,version=10,seqNo=9,primaryTerm=1,result=updated,shards=ShardInfo{total=2, successful=2, failures=[]}]
    

查看 ES 结果，可以看到姓名已经成功修改。

![image-20200504130848107](https://images.gitbook.cn/e0a0ce90-9f70-11ea-a321-115ce75343e0)

### Delete 操作

ES 使用 DeleteRequest 对 Delete 请求操作做了封装，需要定义文档的 id、索引、文档类型。

    
    
       DeleteRequest delete = new DeleteRequest();
                delete.id("1");
                delete.index("java_client");
                delete.type("_doc");
                DeleteResponse delete1 = client.delete(delete, RequestOptions.DEFAULT);
                System.out.println(delete1);
    

输出结果如下：

    
    
    DeleteResponse[index=java_client,type=_doc,id=1,version=11,result=deleted,shards=ShardInfo{total=2, successful=2, failures=[]}]
    

### 小结

Java 对 ES 的封装还是很全面的，但是如果初学者在没有了解 ES 的不同概念之前，直接使用这些 API 肯定会一头雾水，所以本课时完全可以把 21
课时的内容作为参考，就会明白为什么 ES 会这么封装，以及不同的 Request 需要定义哪些数据。

