### Docker 工具

为了快速学习理解 ES，我并不建议学员精通各种平台 ES 安装方法，有的人使用 Windows、MAC 以及 Ubuntu，目前据我所知，在 MAC
上安装是坑最少的，其他平台安装起来比较麻烦。我也不可能在不同的平台演示安装过程，为了能够统一安装方法，以及更快地进入 ES 学习环境，我这里使用
Docker 作为安装平台。只需要傻瓜式的一步步安装 Docker 这个软件，然后从网上下载别人打包好的 ES 容器，就能够一键式启动，这里仅仅把
Docker 当做工具，不需要深入研究命令以及原理。

### 安装 Docker

Docker 是个集装箱，里面可以装载各种环境的应用程序，它相当于一个虚拟机，为了后面更加简单的搭建分布式 ES 文本将采用 Docker
作为环境搭建的基础。我们的目的是简单的将 ES 实例部署在 Ubuntu
操作系统中，读者们不需要过分关注，它只是一个能够快速提供给我们任何操作系统环境的工具，无论是 CentOS、Ubuntu。玩过虚拟机的学员应该都听过或者用过
Virtual Box 与 VMWare 虚拟机，Docker 就相当于虚拟机，Docker 镜像就相当于虚拟机中的镜像，像 Windows、Ubuntu
镜像。Docker 容器就相当于虚拟机中启动的一个操作系统，这些简单的概念先大概有个了解。

首先百度搜索 Docker Desktop，去官网下载软件安装包，Docker 提供了 Mac 与 Windows 版本，如果你是个 Linux
爱好者，安装起来方便，但是我还是推荐你使用 Mac 或者 Windows 系统学习，学习曲线更平缓，不会让你陷入 Docker 学习中去。

![image-20191113094114812](https://images.gitbook.cn/2020-04-07-054407.png)

#### **Docker hub 下载 ES**

本机安装好 Docker Desktop 后启动 Docker，到登录到 Docker Hub 网站查找相应的 ES 镜像，Docker Hub
提供了很多各种打包好的应用程序，像 Redis、MySQL、Nginx 镜像，或者 Python、PyTroch、TensorFlow
镜像，应有尽有，只需要拉取镜像直接就可以使用，不需要进行各种繁琐的配置。

笔者经常到 Docker Hub 上拉取实验所需要的镜像，跳过繁琐的安装过程。

![image-20191113214127440](https://images.gitbook.cn/2020-04-07-054417.png)

打开控制台输入 `docker pull elasticsearch`，如果不加版本号，默认是下载最新版本，也可以下载指定版本。

    
    
    (base) ➜  ~ docker pull elasticsearch:7.4.2
    7.4.2: Pulling from library/elasticsearch
    d8d02d457314: Pull complete 
    f26fec8fc1eb: Pull complete 
    8177ad1fe56d: Pull complete 
    d8fdf75b73c1: Pull complete 
    47ac89c1da81: Pull complete 
    fc8e09b48887: Pull complete 
    367b97f47d5c: Pull complete 
    Digest: sha256:543bf7a3d61781bad337d31e6cc5895f16b55aed4da48f40c346352420927f74
    Status: Downloaded newer image for elasticsearch:7.4.2
    docker.io/library/elasticsearch:7.4.2
    

### 容器启动

启动容器，向外暴露 9200 端口与 9300
端口，这样就可以在本机访问到容器内部。如果容器不像外暴露端口的话，外部是无法找到这个容器的，此刻你的电脑就形成了一个路由器，Docker
中所有的容器处在同一个路由下，想要与外界通讯就必须提供端口，由本机映射该端口号。9200 是 HTTP 协议，可以通过这个端口，可以使用浏览器与 ES
交互，比如查看节点信息以及节点状态，这个端口主要用于外部通讯，9300 作为 TCP 协议，jar 之间就是通过 TCP 协议通讯，ES 集群之间是通过
9300 端口进行通讯。直接粘贴复制下面命令，启动一个单节点 ES 容器实例，这里就相当于我们启动了一个数据库实例服务。

    
    
    (base) ➜  ~ docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.4.2 
    

测试一下 ES 运行状态，在浏览器里输入 `http://localhost:9200/`，查看 ES 节点返回的结果信息。

可以观察到，这个 ES 实例名称为 67de0bb58971，集群名称为 docker-cluster，ES 版本号是 7.4.2，基于的 Lucene
版本是 8.2.0。

    
    
    {
      "name" : "67de0bb58971",
      "cluster_name" : "docker-cluster",
      "cluster_uuid" : "N0pOfApBReS5y-YMN7bR_Q",
      "version" : {
        "number" : "7.4.2",
        "build_flavor" : "default",
        "build_type" : "docker",
        "build_hash" : "2f90bbf7b93631e52bafb59b3b049cb44ec25e96",
        "build_date" : "2019-10-28T20:40:44.881551Z",
        "build_snapshot" : false,
        "lucene_version" : "8.2.0",
        "minimum_wire_compatibility_version" : "6.8.0",
        "minimum_index_compatibility_version" : "6.0.0-beta1"
      },
      "tagline" : "You Know, for Search"
    }
    

前面介绍过，9200 端口是 Http 协议接口，ES 提供了一个 Rest API `_cat` 作为监控 ES 状态信息的接口。

输入 `http://localhost:9200/_cat/` 查看提供的 API，下面的 API 都可以测试使用，使用 `/_cat/health`
查看节点状态。

    
    
    =^.^=
    /_cat/allocation
    /_cat/shards
    /_cat/shards/{index}
    /_cat/master
    /_cat/nodes
    /_cat/tasks
    /_cat/indices
    /_cat/indices/{index}
    /_cat/segments
    /_cat/segments/{index}
    /_cat/count
    /_cat/count/{index}
    /_cat/recovery
    /_cat/recovery/{index}
    /_cat/health
    /_cat/pending_tasks
    /_cat/aliases
    /_cat/aliases/{alias}
    /_cat/thread_pool
    /_cat/thread_pool/{thread_pools}
    /_cat/plugins
    /_cat/fielddata
    /_cat/fielddata/{fields}
    /_cat/nodeattrs
    /_cat/repositories
    /_cat/snapshots/{repository}
    /_cat/templates
    

例如：

输入 `http://localhost:9200/_cat/health` 查看节点信息，ES
为节点提供了三种颜色反应节点的健康状态：green、yellow、red，依次表示健康、一般、较差。从下面 ES 返回的节点信息可以看出节点运行状态良好。

    
    
    1573699390 02:43:10 docker-cluster green 1 1 0 0 0 0 0 0 - 100.0%
    

### 小结

越过繁琐的配置坑，我们只想快速地学习 ES 或者其他应用，在配置这块不同的平台会有不同的配置，往往是还没开始学习，配置可能得整个几天，使用 Docker
会让你直接进入到学习状态，并且不用管是什么平台。

