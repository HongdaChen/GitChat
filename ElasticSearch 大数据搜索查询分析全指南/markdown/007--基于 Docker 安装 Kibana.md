Kibana 是 ES 的可视化分析工具，在 Kibana 上可以对 ES 中的业务数据各种聚合分析，监控等。可以把 Kibana
理解为ES的交互界面，为什么需要这个呢？

这个也好理解，传统数据库基本上是没有界面的，像 Mysql 就是一个服务实例，为了操作方便市场出现了很多工具，像我最喜欢用的 SqlYog，Navicat
等。

Kibana 与 ES 交互友好，使用 Kibana 可以更好的对 ES 数据进行分析统计可视化等。kibana 就相当于 ES 的可视化界面。使用 ES
做数据指标或者日志分析大多数都是在跟 Kibana 打交道，Kibana 是你完成 ES 相关业务需求必不可少的工具。

了解了基本的 ES 原理与可视化工具，现在就尝试安装一下，体验如何使用 ES 建索引，mapping ,以及查看数据。

### Kibana 安装

安装 Kibana ,同样去 Docker Hub 搜索镜像，要注意的是版本号一定要一致。

    
    
    (base) ➜  ~ docker pull kibana:7.4.2
    7.4.2: Pulling from library/kibana
    d8d02d457314: Already exists 
    bc64069ca967: Pull complete 
    c7aae8f7d300: Pull complete 
    8da0971e3b41: Pull complete 
    58ea4bb2901c: Pull complete 
    b1e21d4c2a7e: Pull complete 
    3953eac632cb: Pull complete 
    5f4406500758: Pull complete 
    340d85e0d1c7: Pull complete 
    1768564d16fb: Pull complete 
    Digest: sha256:355f9c979dc9cdac3ff9a75a817b8b7660575e492bf7dbe796e705168f167efc
    Status: Downloaded newer image for kibana:7.4.2
    docker.io/library/kibana:7.4.2
    

使用下面命令快速启动 Kibana 节点，--link 表示把 Kibana 放到与 ES
同一个网络中，在它们之间建立一个联系，使得它们能够畅通无阻的通讯沟通。同样需要对 Kibana
对外暴露一个5601端口，使得外部浏览器能够与内部Kibana 沟通通讯。

    
    
    docker run --link 78a05ed33ba5:elasticsearch -p 5601:5601 kibana:7.4.2
    

这里需要特别的关注，'78a05ed33ba5' 是我启动 ES 节点的容器ID，切不可盲目的粘贴复制我的命令代码，你只需要把你启动的 ES 容器的 ID
或者Name替换进去即可。使用`docker ps -a`即可查看 ID，首先要保证的是你的 ES节点正确启动起来。

    
    
    (base) ➜  ~ docker ps -a        
    CONTAINER ID        IMAGE                                                 COMMAND                  CREATED             STATUS              PORTS                                            NAMES
    78a05ed33ba5        docker.elastic.co/elasticsearch/elasticsearch:7.4.2   "/usr/local/bin/dock…"   9 minutes ago       Up 9 minutes        0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp   heuristic_edison
    

### 启动 Kibana

现在开始 Kibana 初体验吧，输入`http://localhost:5601`

![image-20191114203428104](https://images.gitbook.cn/2020-04-07-062351.png)

![image-20191114203502321](https://images.gitbook.cn/2020-04-07-062409.png)

打开界面后先愉快的体验一把，点击 Dev tool,在这里可以直接与 ES 进行交互，前面使用浏览器交互的 URL，在这里完全可以使用，体验并看看效果吧。

![image-20191114203758603](https://images.gitbook.cn/2020-04-07-062424.png)

### 小结

至此我们总结一下安装方式：

  1. 本机安装 docker desktop
  2. Docker hub 拉取 ElasticSearch ,kibana 镜像
  3. 使用提供的命令分别启动两个容器
  4. 打开 kibana 

一共就四个步骤就能轻松启动 ES 与 Kibana，告别各种繁琐安装方式以及坑，使用 docker 能够最快的让你进入到学习环境中，不用担心看不懂
docker 命令，它只是一个帮助我们快速搭建环境的工具，以后再慢慢了解就是了。

