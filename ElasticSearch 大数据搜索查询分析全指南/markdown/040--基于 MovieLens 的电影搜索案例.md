Elastic 提供了一款 App Search 工具，这块工具能够无缝与 ElasticSearch 结合，App Search
这款工具直接通过操作而不用开发任何页面就可以实现一个检索系统。下载链接：[App
Search](https://www.elastic.co/downloads/past-releases/app-search-7-4-0)。

下载解压好之后，有以下两个地方需要做修改。这里注意，我将 App Search 安装到本地，如果你安装到其他机器，你需要指定好
elasticsearch.host 的参数。

    
    
    #
    allow_es_settings_modification: true
    #
    # Elasticsearch full cluster URL:
    #
    elasticsearch.host: http://127.0.0.1:9200
    

![image-20200607205103861](https://images.gitbook.cn/c98e51c0-ac75-11ea-8951-fd44b41d4a8a)

如何出现以下界面说明启动成功了：

![image-20200607205314956](https://images.gitbook.cn/d2864f30-ac75-11ea-a004-316ca14d0de8)

这里如果启动失败，你首先要检查一下，ES 是否能够连通，其次是 App Search 版本与 ES 的版本是否一致，能够对应上。

访问 http://localhost:3002/ 将会出现如下界面：

![image-20200607205453965](https://images.gitbook.cn/ddbe5190-ac75-11ea-9793-c52d403756b9)

新建一个搜索引擎：

![image-20200608205107138](https://images.gitbook.cn/e76b4720-ac75-11ea-
aa64-5561411ce61e)

本次实验以 MovieLens 数据集为基础，作为 App Search
功能一个探索。数据集地址：[MovieLens](https://grouplens.org/datasets/movielens/)。

我下载了最小的数据集。

![image-20200607214151159](https://images.gitbook.cn/0f8f9d50-ac76-11ea-
bbfe-31ec274b5742)

切换到 ES 环境，我们将用 Python 对数据集进行简单的处理：

![image-20200607213106242](https://images.gitbook.cn/1abcf150-ac76-11ea-
acb2-8335c26cfba1)

启动 Spyder 后，我们用 Python 处理 CSV。解压压缩包后，共有
links.csv、movies.csv、ratings.csv、tags.csv 几个文件，这里为了方便，我们把 movies.csv、tags.csv
文件 merge 到一起，作为实验数据。这里要注意，CSV 文件的字段名称不能出现大写字母，否则会导入失败。

    
    
    import pandas as pd
    import numpy as np
    import json
    
    
    user_tags=['user_id','movie_id','tag','timestamp']
    users = pd.read_csv('tags.csv',  names=user_tags,encoding='latin-1')
    
    movies_tag=['movie_id','title','genres']
    movies = pd.read_csv('movies.csv',  names=movies_tag,encoding='latin-1')
    
    results=users.merge(movies)
    
    
    a=results.to_json(orient="records",force_ascii=False)
    with open("results.json","w") as f:
        f.write(a)
    

在 App Search 中上传我们刚处理好的 JSON 文件。上传完成之后，就会跳出已经导入成功的窗口。

![image-20200608200421078](https://images.gitbook.cn/4a6bd240-ac76-11ea-89ce-91247f7c923b)

![image-20200608220633168](https://images.gitbook.cn/5293ae70-ac76-11ea-a004-316ca14d0de8)

在 schema 这里我们还可以更新字段类型：

![image-20200608221017938](https://images.gitbook.cn/5d431fe0-ac76-11ea-
ae24-b5f7c6b64137)

然后我们在 Reference UI 这里创建一个 Search 应用：

![image-20200608221210550](https://images.gitbook.cn/66e73bd0-ac76-11ea-
acb2-8335c26cfba1)

我们这里关键字输入 some，就可以看到以下搜索的案例。如果你觉得满意就可以将这个项目的 zip 下载下来，然后根据自己的需求，再进一步做优化。

![image-20200608221345712](https://images.gitbook.cn/720dd910-ac76-11ea-8951-fd44b41d4a8a)

App-seach 不仅仅提供了前端的可视化工作，也提供了一个后台管理工作，通过后台管理可以看到具体的查询信息、点击信息等。

![image-20200608223201250](https://images.gitbook.cn/7d63eac0-ac76-11ea-b8fa-67c56133b3a3)

### 小结

本课时主要讲解了 App Search 的工作原理以及使用方法，目前已经推出企业版了。App Search
强大与方便之处还是在于它的后台统计功能。可以用来监控哪个查询频率最高，哪个链接转化率最好等。对于很多企业而言还是比较方便的，不用分配人力物力去重新开发一个
ES 查询监控系统。

