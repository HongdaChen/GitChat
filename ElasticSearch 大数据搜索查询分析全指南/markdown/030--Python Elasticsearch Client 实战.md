### 安装 Anaconda 环境

如果网络条件允许可以去 Anaconda 官网下载，如果一般推荐去清华镜像下载：

> <https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/>

Anaconda 是一个 Python 包管理器，它不仅仅可以很方便地下载各种包，而且提前装好了经常需要用的 Python 包，使用 Anaconda
还可以做到 Python 3 与 Python 2 共存等。

安装好 Anaconda 后，打开控制台，创建一个新的 Python 3 环境，并且给这个环境命名为 es，并指定 Python 版本。

    
    
    conda create --name es  python=3.7
    

切换到 ES Python 环境：

    
    
    conda activate es
    

查看 Python 版本是 3.7：

    
    
    (es) ➜  ~ python
    Python 3.7.6 (default, Jan  8 2020, 13:42:34)
    [Clang 4.0.1 (tags/RELEASE_401/final)] :: Anaconda, Inc. on darwin
    Type "help", "copyright", "credits" or "license" for more information.
    >>>
    

### Python 开发利器 Spyder

Spyder 是开发 Python 的一款工具，界面非常像 Matlab，非常适合做数据分析的工具。

首先安装 Spyder 然后再启动 Spyder，在 ES 的 Python 环境下输入 `spyder` 回车即可启动：

    
    
    (es) ➜  ~ conda install -c anaconda spyder
    (es) ➜  ~ spyder
    

界面图如下：

![image-20200322160158998](https://images.gitbook.cn/2020-04-07-063309.png)

### 安装 elasticsearch-py

Python 是一门简单高效的语言，ES 提供了 Python 的接口，使用 Python 可以进行对 ES 的相关操作。

ES 提供了多个版本的 Python 接口：

    
    
    # Elasticsearch 7.x
    elasticsearch>=7.0.0,<8.0.0
    # Elasticsearch 6.x
    elasticsearch>=6.0.0,<7.0.0
    # Elasticsearch 5.x
    elasticsearch>=5.0.0,<6.0.0
    # Elasticsearch 2.x
    elasticsearch>=2.0.0,<3.0.0
    

使用 pip 安装 elasticsearch-py 最新版的，最新的一版是 Elasticsearch 7.5.1：

    
    
    pip install elasticsearch
    

如果需要安装 6.x 或者 5.x 需要找到对应的版本号：

    
    
    pip install elasticsearch==6.4.0
    

切换到 ES Python 环境，并安装 ES-py 客户端：

    
    
    Last login: Sun Mar 22 14:37:12 on ttys003
    (base) ➜  ~ conda activate es
    (es) ➜  ~ pip install elasticsearch
    Collecting elasticsearch
      Downloading elasticsearch-7.6.0-py2.py3-none-any.whl (88 kB)
         |████████████████████████████████| 88 kB 634 kB/s
    Collecting urllib3>=1.21.1
      Downloading urllib3-1.25.8-py2.py3-none-any.whl (125 kB)
         |████████████████████████████████| 125 kB 2.3 MB/s
    Installing collected packages: urllib3, elasticsearch
    Successfully installed elasticsearch-7.6.0 urllib3-1.25.8
    (es) ➜  ~
    

测试安装成功，如果在 Python 环境下，导入没出错基本上就是安装成功：

    
    
    from elasticsearch import Elasticsearch
    

![image-20200316212202361](https://images.gitbook.cn/2020-04-07-063310.png)

### Python ES 客户端

ES_API 是 ES 的地址，可以是多个，另外如果有登录验证模块，也可以使用 http_auth，timeout 设置连接超时，放置被占用时间过长。

    
    
    es = Elasticsearch([ES_API],http_auth=(USER,PASSWD),timeout=1000)
    

因为使用我们的 ES 是使用 Docker 进行启动的，开放的端口号是9200，没有用户名跟密码验证，因此如果想从外部访问 ES 容器内部，直接使用
localhost:9200，然后 Docker 会自动映射到 ES 的容器中。

    
    
    es = Elasticsearch(['localhost:9200'],timeout=1000)
    

查看 ES 信息，可以看到名字是 es7_01，集群名称是 eslearn，版本是 7.4.2，lucene 版本是 8.2.0。

    
    
    es.info()
    Out[43]: 
    {'name': 'es7_01',
     'cluster_name': 'eslearn',
     'cluster_uuid': 'gdO9SlQGSxqSH2QXuU8Tzg',
     'version': {'number': '7.4.2',
      'build_flavor': 'default',
      'build_type': 'docker',
      'build_hash': '2f90bbf7b93631e52bafb59b3b049cb44ec25e96',
      'build_date': '2019-10-28T20:40:44.881551Z',
      'build_snapshot': False,
      'lucene_version': '8.2.0',
      'minimum_wire_compatibility_version': '6.8.0',
      'minimum_index_compatibility_version': '6.0.0-beta1'},
     'tagline': 'You Know, for Search'}
    

检查一下 nodes 的节点信息，可以看到 IP 信息以及节点名称。

    
    
    es.cat.nodes()
    Out[26]: '172.18.0.4 42 97 3 0.23 0.48 0.54 dilm * es7_02\n172.18.0.5 33 97 3 0.23 0.48 0.54 dilm - es7_01\n'
    

![image-20200316214117792](https://images.gitbook.cn/2020-04-07-063312.png)

查看一下 master 信息，可以看到 master 是 172.18.0.4，es7_02。

    
    
    es.cat.master()
    Out[27]: 'lmCHH48ySCqFD-d5KdFQNg 172.18.0.4 172.18.0.4 es7_02\n'
    

查看 shards 信息：

    
    
    es.cat.shards()
    Out[34]: '.kibana_task_manager_1       0 p STARTED        2 23.2kb 172.18.0.5 es7_01\n.kibana_task_manager_1       0 r STARTED        2 23.2kb 172.18.0.4 es7_02\nkibana_sample_data_flights   0 p STARTED    13059  6.3mb 172.18.0.5 es7_01\nkibana_sample_data_flights   0 r STARTED    13059  6.4mb 172.18.0.4 es7_02\n.apm-agent-configuration     0 r STARTED        0   283b 172.18.0.5 es7_01\n.apm-agent-configuration     0 p STARTED        0   283b 172.18.0.4 es7_02\nkibina_sample_learning_index 0 p STARTED        3  4.4kb 172.18.0.5 es7_01\nkibina_sample_learning_index 0 r STARTED        3  9.5kb 172.18.0.4 es7_02\nkibana_sample_data_ecommerce 0 p STARTED     4675  4.9mb 172.18.0.5 es7_01\nkibana_sample_data_ecommerce 0 r STARTED     4675  4.8mb 172.18.0.4 es7_02\nclass1                       1 r STARTED        1  4.3kb 172.18.0.5 es7_01\nclass1                       1 p STARTED        1  4.3kb 172.18.0.4 es7_02\nclass1                       0 r STARTED        0   283b 172.18.0.5 es7_01\nclass1                       0 p STARTED        0   283b 172.18.0.4 es7_02\nclass                        0 r STARTED        8 14.9kb 172.18.0.5 es7_01\nclass                        0 p STARTED        8 14.9kb 172.18.0.4 es7_02\nkibana_sample_data_logs      0 r STARTED    14074 11.3mb 172.18.0.5 es7_01\nkibana_sample_data_logs      0 p STARTED    14074 11.4mb 172.18.0.4 es7_02\nclass2                       0 r STARTED        0   283b 172.18.0.5 es7_01\nclass2                       0 p STARTED        0   283b 172.18.0.4 es7_02\nclass2                       0 r UNASSIGNED                         \n.kibana_1                    0 p STARTED      175    1mb 172.18.0.5 es7_01\n.kibana_1                    0 r STARTED      175    1mb 172.18.0.4 es7_02\nstudent                      0 p STARTED        1  4.3kb 172.18.0.5 es7_01\nstudent                      0 r STARTED        1  4.3kb 172.18.0.4 es7_02\n'
    

判断索引是否存在：

    
    
    es.indices.exists("class")
    Out[37]: True
    

查看 class 的 mapping 信息：

    
    
    es.indices.get_mapping("class")
    Out[38]: 
    {'class': {'mappings': {'properties': {'age': {'type': 'long'},
        'name': {'type': 'text',
         'fields': {'keyword': {'type': 'keyword', 'ignore_above': 256}}},
        'sex': {'type': 'text',
         'fields': {'keyword': {'type': 'keyword', 'ignore_above': 256}}}}}}}
    

查看 class 索引的 settting 信息：

    
    
    es.indices.get_settings("class")
    Out[39]: 
    {'class': {'settings': {'index': {'creation_date': '1577714512597',
        'number_of_shards': '1',
        'number_of_replicas': '1',
        'uuid': 'b0QYQbFjSPWQ_Lfj1Gwuhw',
        'version': {'created': '7040299'},
        'provided_name': 'class'}}}}
    

### 小结

本篇介绍了如何安装 ES-py，以及如何如何使用 Python 连接上 Docker 中的 ES 容器，当使用 Docker 映射本地接口到 Docker
容器接口时候，就可以通过本地 IP 与接口访问到容器内部。当连接上 ES 之后，查看 ES 信息、ES 节点信息、ES 指定某个索引的信息等。

当连接上 ES 客户端后，就可以进行增删改查业务分析，把 ES 当做一个数据库来使用。这是基于 ES 分析业务需求的第一步，后面的课时将会介绍如何使用
Python 客户端建索引、添加数据、修改数据，以及如何在 ES-py 中如何使用 DSL。

