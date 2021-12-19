### 启动步骤

一共有三个步骤：

  1. 在桌面或者任何地方新建一个文件夹 命名为 docker
  2. 新建一个文件命名为`docker-compose.yml`, 注意一定要命名成`docker-compose.yml`
  3. 把下面提供的代码粘贴复制进去，在当前目录下打开控制台输入 `docker-compose up`

值得注意的是你一定要保证你的 Docker 是处于打开状态，否则是无法启动的。我们利用 Docker 提供的 compose
工具帮我们定制好启动的服务内容。

### compose 脚本

服务有 cerebro 用来监控 ES 健康状态的，kibana 是 ES 的可视化图像界面，elasticsearch 则是就是 ES。

    
    
    version: '2.2'
    services:
      cerebro:
        image: lmenezes/cerebro:0.8.3
        container_name: cerebro
        ports:
          - "9000:9000"
        command:
          - -Dhosts.0.host=http://elasticsearch:9200
        networks:
          - es7net
      kibana:
        image: docker.elastic.co/kibana/kibana:7.4.2
        container_name: kibana7
        environment:
          - I18N_LOCALE=zh-CN
          - XPACK_GRAPH_ENABLED=true
          - TIMELION_ENABLED=true
          - XPACK_MONITORING_COLLECTION_ENABLED="true"
        ports:
          - "5601:5601"
        networks:
          - es7net
      elasticsearch:
        image: docker.elastic.co/elasticsearch/elasticsearch:7.4.2
        container_name: es7_01
        environment:
          - cluster.name=eslearn
          - node.name=es7_01
          - bootstrap.memory_lock=true
          - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
          - discovery.seed_hosts=es7_01,es7_02
          - cluster.initial_master_nodes=es7_01,es7_02
        ulimits:
          memlock:
            soft: -1
            hard: -1
        volumes:
          - es7data1:/usr/share/elasticsearch/data
        ports:
          - 9200:9200
        networks:
          - es7net
      elasticsearch2:
        image: docker.elastic.co/elasticsearch/elasticsearch:7.4.2
        container_name: es7_02
        environment:
          - cluster.name=eslearn
          - node.name=es7_02
          - bootstrap.memory_lock=true
          - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
          - discovery.seed_hosts=es7_01,es7_02
          - cluster.initial_master_nodes=es7_01,es7_02
        ulimits:
          memlock:
            soft: -1
            hard: -1
        volumes:
          - es7data2:/usr/share/elasticsearch/data
        networks:
          - es7net
    volumes:
      es7data1:
        driver: local
      es7data2:
        driver: local
    
    networks:
      es7net:
        driver: bridge
    

### 注意

要注意的是因为刚才我们部署单个节点的 ES 跟 Kibana，把端口号给占用了，所以在一键式部署分布式 ES
之前要把刚才的两个容器停掉。（如果你能够正常启动则略过）

使用这个命令停掉所有正在运行的容器：`docker stop $(docker ps)`

### 启动分布式 ES

下面让我们开始一键式部署分布式 ES 系统吧！

    
    
    (base) ➜  docker docker-compose up  
    Creating network "docker_es7net" with driver "bridge"
    Creating es7_02  ... done
    Creating es7_01  ... done
    Creating cerebro ... done
    Creating kibana7 ... done
    

在浏览器输入 http://localhost:5601/，进入 Kibana 中文界面

![image-20191114210324066](https://images.gitbook.cn/2020-04-07-063339.png)

**cerebro 查询**

另外脚本还安装了一个叫 cerebro 的工具，能够监视 ES
状态。在浏览器输入：http://localhost:9000/。从图片删不难看出，eslearn 这个 ES 实例，启动了 2 个节点，有 3 个索引，和
6 个分片。

![image-20191114214509614](https://images.gitbook.cn/2020-04-07-063343.png)

### 小结

分布式环境才是我们以后工作中会遇到的真实环境，使用 docker 一站式部署会节约你更多的时间。作为一个软件开发分析人员你可以了解 ES
分布式原理，但是业务分析与实现才是你的重中之重，部署与维护 ES
会有专门的运维人员来做，因此这里也是快速搭建好学习环境，没有将大量的篇幅放在如何在本机配置分布式 ES 。

