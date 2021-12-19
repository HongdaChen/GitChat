### 前言

本节课我们来学习 Tomcat 集群的集体实现方式。

**什么是 Tomcat 集群？**

集群的意思是在多个服务器上部署同一个 Java
Application，并结合负载均衡来为客户端自动选择一个服务终端，这样就可以减轻服务端的压力，相当于之前是一个 Tomcat 来提供服务，现在同时搞
10 个 Tomcat，那平摊到每一个 Tomcat 的压力就小了很多。这里我们使用 Nginx 技术来实现对请求的负载均衡，如下图所示。

![](https://images.gitbook.cn/963fb740-9a1c-11e8-8334-9bfa28241acd)

**工具**

  * nginx-1.15.2
  * apache-tomcat-9.0.8

### 实现步骤

1\. homebrew 安装 Nginx。

2\. 解压两个 Tomcat，分别命名为 apache-tomcat-9.0.8-1 和 apache-tomcat-9.0.8-2。

![](https://images.gitbook.cn/a47cede0-9a1d-11e8-bd2f-43e393597943)

3\. 修改两个 Tomcat 的启动端口，分别为 8080 和 8181。

![](https://images.gitbook.cn/e7429670-9a1d-11e8-8334-9bfa28241acd)

![](https://images.gitbook.cn/fce0f350-9a1d-11e8-bd2f-43e393597943)

4\. 修改两个 Tomcat 默认的 index.jsp 页面，用以区分不同的 Tomcat。

![](https://images.gitbook.cn/3adc1b80-9a1e-11e8-bd2f-43e393597943)

![](https://images.gitbook.cn/3f0e5380-9a1e-11e8-992f-9dfb28d2b53f)

5\. 同时启动两个 Tomcat，访问测试。

![](https://images.gitbook.cn/52ee1070-9a1e-11e8-9724-fb3ff7de9e38)

![](https://images.gitbook.cn/573445a0-9a1e-11e8-9724-fb3ff7de9e38)

![](https://images.gitbook.cn/61605640-9a1e-11e8-8334-9bfa28241acd)

![](https://images.gitbook.cn/6679e100-9a1e-11e8-9724-fb3ff7de9e38)

6\. 配置 Nginx，打开 nginx/nginx.conf。

![](https://images.gitbook.cn/9b87ba20-9a1e-11e8-992f-9dfb28d2b53f)

进行如下配置：

    
    
        #Tomcat 集群
        upstream  myapp {   #Tomcat 集群名称 
            server    localhost:8080;   #tomcat1配置
            server    localhost:8181;   #tomcat2配置
        }
    
        server {
            listen       9090;
            server_name  localhost;
    
            #charset koi8-r;
    
            #access_log  logs/host.access.log  main;
    
            location / {
                #root   html;
                #index  index.html index.htm;
                proxy_pass http://myapp;
                proxy_redirect default;
            }
         }
    

核心配置：

![](https://images.gitbook.cn/f054dd30-9a1e-11e8-992f-9dfb28d2b53f)

6\. 终端命令启动 Nginx。

![](https://images.gitbook.cn/09125460-9a1f-11e8-992f-9dfb28d2b53f)

7\. 测试，访问 <http://localhost:9090>。

第一次看到，运行 tomcat1 中的程序。

![](https://images.gitbook.cn/128ec410-9a1f-11e8-992f-9dfb28d2b53f)

刷新，第二次看到运行 tomcat2 中的程序。

![](https://images.gitbook.cn/1d3e3580-9a1f-11e8-bd2f-43e393597943)

至此，我们利用 Nginx 实现了负载均衡的 Tomcat 集群。

### 总结

本节课我们讲解了 Nginx 的使用，使用 Nginx 的目的是搭建 Tomcat 服务集群，以应对高并发访问下 Tomcat
服务器压力过大可能出现宕机的情况，集群的核心思想就是一个 Tomcat 干活太累，那么就找十个 Tomcat 来一起分担任务和压力，Nginx
服务就是完成这项工作的。

### 分享交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《快速上手 Spring 全家桶》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「200」给小助手-伽利略获取入群资格。**
>
> 阅读文章过程中有任何疑问随时可以跟其他小伙伴讨论，或者直接向作者提问（作者看到后抽空回复）。你的分享不仅帮助他人，更会提升自己。

