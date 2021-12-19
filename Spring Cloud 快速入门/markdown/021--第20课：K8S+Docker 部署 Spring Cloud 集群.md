在一个实际的大型系统中，微服务架构可能由成百上千个服务组成，我们发布一个系统如果都单纯的通过打包上传，再发布，工作量无疑是巨大的，也是不可取的，前面我们知道了可以通过
Jenkins 帮我们自动化完成发布任务。

但是，我们知道一个 Java
应用其实是比较占用资源的，每个服务都发布到物理宿主机上面，资源开销也是巨大的，而且每扩展一台服务器，都需要重复部署相同的软件，这种方式显然是不可取的。

容器技术的出现带给了我们新的思路，我们将服务打包成镜像，放到容器中，通过容器来运行我们的服务，这样我们可以很方便进行分布式的管理，同样的服务也可以很方便进行水平扩展。

Docker 是容器技术方便的佼佼者，它是一个开源容器。而 Kubernetes（以下简称 K8S），是一个分布式集群方案的平台，它天生就是和 Docker
一对，通过 K8S 和 Docker 的配合，我们很容易搭建分布式集群环境。

下面，我们就来看看 K8S 和 Docker 的吸引之处。

### 集群环境搭建

本文用一台虚拟机模拟集群环境。

> 操作系统：CentOS7 64位
>
> 配置：内存2GB，硬盘40GB。

注：真正的分布式环境搭建方案类似，可以参考博文：[《Kubernetes学习2——集群部署与搭建》](https://blog.csdn.net/weixin_29115985/article/details/78950299)。

下面开始搭建集群环境。

**1.** 关闭防火墙：

    
    
    systemctl disable firewalld
    systemctl stop firewalld
    iptables -P FORWARD ACCEPT
    

**2.** 安装 etcd：

    
    
    yum install -y etcd
    

安装完成后启动 etcd：

    
    
    systemctl start etcd
    systemctl enable etcd
    

启动后，我们可以检查 etcd 健康状况：

    
    
    etcdctl -C http://localhost:2379 cluster-health
    

出现下面信息说明 etcd 目前是稳定的：

    
    
    member 8e9e05c52164694d is healthy: got healthy result from http://localhost:2379
    cluster is healthy
    

**3.** 安装 Docker：

    
    
    yum install docker -y
    

完成后启动 Docker：

    
    
    chkconfig docker on
    service docker start
    

**4.** 安装 Kubernetes：

    
    
    yum install kubernetes -y
    

安装完成后修改配置文件 `/etc/kubernetes/apiserver`：

    
    
    vi /etc/kubernetes/apiserver
    

把 `KUBE_ADMISSION_CONTROL` 后面的 ServiceAccount 删掉，如：

    
    
    KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
    

然后依次启动 `kubernetes-server`：

    
    
    systemctl enable kube-apiserver
    systemctl start kube-apiserver
    systemctl enable kube-controller-manager
    systemctl start kube-controller-manager
    systemctl enable kube-scheduler
    systemctl start kube-scheduler
    

再依次启动 `kubernetes-client`：

    
    
    systemctl enable kubelet
    systemctl start kubelet
    systemctl enable kube-proxy
    systemctl start kube-proxy
    

我们查看集群状态：

    
    
    kubectl get no
    

可以看到以下信息：

    
    
    NAME        STATUS    AGE
    127.0.0.1   Ready     1h
    

至此，我们基于 K8S 的集群环境就搭建完成了，但是我们发布到 Docker 去外部是无法访问的，还要安装 Flannel 以覆盖网络。

执行以下命令安装 Flannel：

    
    
    yum install flannel -y
    

安装完成后修改配置文件 `/etc/sysconfig/flanneld`：

    
    
    vim /etc/sysconfig/flanneld
    

内容如下：

    
    
    # Flanneld configuration options  
    
    # etcd url location.  Point this to the server where etcd runs
    FLANNEL_ETCD_ENDPOINTS="http://127.0.0.1:2379"
    
    # etcd config key.  This is the configuration key that flannel queries
    # For address range assignment
    FLANNEL_ETCD_PREFIX="/atomic.io/network"
    
    # Any additional options that you want to pass
    FLANNEL_OPTIONS="--logtostderr=false --log_dir=/var/log/k8s/flannel/ --etcd-prefix=/atomic.io/network  --etcd-endpoints=http://localhost:2379 --iface=enp0s3"
    

其中，enp0s3 为网卡名字，通过 ifconfig 可以查看。

然后配置 etcd 中关于 Flannel 的 key：

    
    
    etcdctl mk /atomic.io/network/config '{ "Network": "10.0.0.0/16" }'
    

其中 `/atomic.io/network` 要和配置文件中配置的一致。

最后启动 Flannel 并重启 Kubernetes：

    
    
    systemctl enable flanneld
    systemctl start flanneld
    systemctl enable flanneld
    service docker restart
    systemctl restart kube-apiserver
    systemctl restart kube-controller-manager
    systemctl restart kube-scheduler
    systemctl restart kubelet
    systemctl restart kube-proxy
    

这样，一个完整的基于 K8S+Docker 的集群环境搭建完成了，后面我们就可以在这上面部署分布式系统。

> 本文只是做演示，不会真正的发布一套系统，因此我们以注册中心 register 为例，演示如何发布一套分布式系统。

### 创建 Docker 镜像

我们首先将 register 本地打包成 Jar 上传到虚拟机上，然后通过 Dockerfile 来创建 register 的镜像，Dockerfile
内容如下：

    
    
    #下载java8的镜像
    FROM java:8
    #将本地文件挂到到/tmp目录
    VOLUME /tmp
    #复制文件到容器
    ADD register.jar /registar.jar
    #暴露8888端口
    EXPOSE 8888
    #配置启动容器后执行的命令
    ENTRYPOINT ["java","-jar","/register.jar"]
    

通过 docker build 构建镜像：

    
    
    docker build -t register.jar:0.0.1 .
    

执行该命令后，会打印以下信息：

    
    
    Sending build context to Docker daemon 48.72 MB
    Step 1/6 : FROM java:8
    Trying to pull repository docker.io/library/java ... 
    apiVersion: v1
    8: Pulling from docker.io/library/java
    5040bd298390: Pull complete 
    fce5728aad85: Pull complete 
    76610ec20bf5: Pull complete 
    60170fec2151: Pull complete 
    e98f73de8f0d: Pull complete 
    11f7af24ed9c: Pull complete 
    49e2d6393f32: Pull complete 
    bb9cdec9c7f3: Pull complete 
    Digest: sha256:c1ff613e8ba25833d2e1940da0940c3824f03f802c449f3d1815a66b7f8c0e9d
    Status: Downloaded newer image for docker.io/java:8
     ---> d23bdf5b1b1b
    Step 2/6 : VOLUME /tmp
     ---> Running in f6f284cf34f2
     ---> bf70efe7bea0
    Removing intermediate container f6f284cf34f2
    Step 3/6 : ADD register.jar registar.jar
     ---> 91d6f5aa9db3
    Removing intermediate container e4dd67f5acc2
    Step 4/6 : RUN bash -c 'touch /register.jar'
     ---> Running in 3b6d5f4ed216
    
     ---> 70381c5e0b5d
    Removing intermediate container 3b6d5f4ed216
    Step 5/6 : EXPOSE 8888
     ---> Running in b87b788ff362
     ---> 912e4f8e3004
    Removing intermediate container b87b788ff362
    Step 6/6 : ENTRYPOINT java -Djava.security.egd=file:/dev/./urandom jar /register.jar
     ---> Running in 1bc65e0bfbea
     ---> 1aec9d5e9c70
    Removing intermediate container 1bc65e0bfbea
    Successfully built 1aec9d5e9c70
    

这时通过 docker images 命令就可以看到我们刚构建的镜像：

    
    
    [root@localhost ~]# docker images
    REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
    register.jar         0.0.1               1aec9d5e9c70        2 minutes ago       692 MB
    docker.io/registry   2                   b2b03e9146e1        5 days ago          33.3 MB
    docker.io/java       8                   d23bdf5b1b1b        18 months ago       643 MB
    

### 系统发布

我们本地虚拟机有了镜像就可以通过 K8S 发布了。

**1.** 创建 `register-rc.yaml`：

    
    
    apiVersion: v1
    kind: ReplicationController
    metadata:
        name: register
    spec:
        replicas: 1
        selector:
            app: register
        template:
            metadata:
                labels:
                    app: register
            spec:
                containers:
                - name: register
                  #镜像名
                  image: register
                  #本地有镜像就不会去仓库拉取
                  imagePullPolicy: IfNotPresent
                  ports:
                  - containerPort: 8888
    

执行命令：

    
    
    [root@localhost ~]# kubectl create -f register-rc.yaml 
    replicationcontroller "register" created
    

提示创建成功后，我们可以查看 pod：

    
    
    [root@localhost ~]# kubectl get po
    NAME             READY     STATUS    RESTARTS   AGE
    register-4l088   1/1       Running   0          10s
    

如果 STATUS 显示为 Running，说明运行成功，否则可以通过以下命令来查看日志：

    
    
    kubectl describe po register-4l088
    

然后我们通过 docker ps 命令来查看当前运行的容器：

    
    
    [root@localhost ~]# docker ps
    CONTAINER ID        IMAGE                                                        COMMAND                CREATED             STATUS              PORTS               NAMES
    dd8c05ae4432        register                                                     "java -jar /app.jar"   8 minutes ago       Up 8 minutes                            k8s_register.892502b2_register-4l088_default_4580c447-8640-11e8-bba0-080027607861_5bf71ba9
    3b5ae8575079        registry.access.redhat.com/rhel7/pod-infrastructure:latest   "/usr/bin/pod"         8 minutes ago       Up 8 minutes                            k8s_POD.43570bb9_register-4l088_default_4580c447-8640-11e8-bba0-080027607861_1f38e064
    

可以看到容器已经在运行了，但是这样外部还是无法访问，因为 K8S 分配的是虚拟 IP，要通过宿主机访问，还需要创建 Service。

编写 `register-svc.yaml`：

    
    
    apiVersion: v1
    kind: Service
    metadata:
      name: register
    spec:
      type: NodePort
      ports:
      - port: 8888
        targetPort: 8888
        节点暴露给外部的端口（范围必须为30000-32767）
        nodePort: 30001
      selector:
        app: register
    

然后执行命令：

    
    
    [root@localhost ~]# kubectl create -f register-svc.yaml
    service "register" created
    

我们可以查看创建的 Service：

    
    
    [root@localhost ~]# kubectl get svc
    NAME         CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    kubernetes   10.254.0.1       <none>        443/TCP          20h
    register     10.254.228.248   <nodes>       8888:30001/TCP   37s
    

这时就可以通过 `IP:30001` 访问 register 了，如果访问不了需要先运行命令：

    
    
    iptables -P FORWARD ACCEPT
    

访问 `http://172.20.10.13:30001`，如图：

![enter image description
here](https://images.gitbook.cn/e2769340-8644-11e8-a808-ef5bdaa75e67)

至此，我们的注册中心就可以通过 K8S 部署了，现在看起来比较麻烦，但是在一个大的集群环境中是很爽的，我们可以在结合前面提到的
Jenkins，把刚才一系列手动的操作交给 Jenkins 做。

