作为 YAML 工程师，经常需要使用 Kubernetes 集群来验证很多技术化场景，如何快速搭建一套完整标准化的集群至关重要。罗列当前能快速部署
Kubernetes 集群的工具有很多种，例如官方首当其冲有 kubeadm 工具，云原生社区有 sealos
作为一键部署的最佳方案，熟悉起来后部署都非常快。

但是你是否考虑过，并不是每一个 YAML 工程师都需要非常了解集群组件的搭配。这里，我给大家推荐的工具是基于单个文件的免配置的部署方式，对比 kubeadm
和 sealos 方案，去掉了对 Kubernetes 官方组件镜像的依赖，并且把 Kubernetes
相关的核心扩展推荐组件也都集成到这个二进制包中，通过软链接暴露，让环境依赖更少，这个安装工具就是 k8e（可以叫“kuber easy”或 K8
易）。k8e 是基于当前主流上游 Kubernetes 发行版 k3s 做的优化封装和裁剪。去掉对 IoT
的依赖，目标就是做最好的服务器版本的发行版本。并且和上游保持一致，可以自由扩展。

### 架构

k8e v1 架构图：

![k8e-arch](https://images.gitbook.cn/193f5ac0-8add-11eb-be8a-4b9e3ee42c47)

启动 k8e，你可以自己放一台机器做试验就可以，4Core/8G RAM
是最小标配。有很多朋友还想安装集群高可用模式，那么就需要三台起步。操作部署步骤如下。

### 下载一键安装工具 k8e

    
    
    mkdir -p /opt/k8e && cd /opt/k8e
    
    curl https://gitreleases.dev/gh/xiaods/k8e/latest/k8e -o k8e
    
    curl https://raw.githubusercontent.com/xiaods/k8e/master/contrib/start-bootstrap.sh -o start-bootstrap.sh
    
    curl https://raw.githubusercontent.com/xiaods/k8e/master/contrib/start-server.sh -o start-server.sh
    
    curl https://raw.githubusercontent.com/xiaods/k8e/master/contrib/start-agent.sh -o start-agent.sh
    
    curl https://raw.githubusercontent.com/xiaods/k8e/master/contrib/stop-k8e.sh -o stop-k8e.sh
    
    curl https://raw.githubusercontent.com/xiaods/k8e/master/contrib/setup-k8s-tools.sh -o setup-k8s-tools.sh
    
    curl https://raw.githubusercontent.com/xiaods/k8e/master/contrib/k8e-uninstall.sh -o k8e-uninstall.sh
    

### 启动集群过程

  * 第一台，属于引导服务（注意，第一台主机 IP 就是 api-server 的 IP）：bash start-bootstrap.sh。
  * 第 2 台到 N+1 台主控节点，必须是奇数，遵循 **CAP 原理** （注意：启动前修改 api-server 的 IP，指向第一台主机 IP）：bash start-server.sh。
  * 第 1 台到 N 台工作负载节点，遵循 **CAP 原理** （注意：启动前修改 api-server 的 IP，指向第一台主机 IP）：bash start-agent.sh。
  * 停掉 K8：bash stop-k8e.sh。
  * 加载 K8s 工具链（kubectl ctr crictl）：bash setup-k8s-tools.sh。

k8e 内置一个同步 api-server IP 的功能，同步后三台主机，宕机任何一台，集群还是 HA 高可用的。

默认 kubeconfig 放在 /etc/k8e/k8e/k8e.yaml 中。

你有三台 Server，就会有 3 个 api-server 的入口，一般我们期望加一个 haproxy 来汇聚 API 入口。这个可以通过 kube-
vip 来实现。下载 kubeconfig 文件，就可以远程管理。注意这里对于 VIP 的 IP，我们需要配置一个弹性 IP 来作为 api-server
的唯一入口 IP，需要启动 k8e 时告诉它生成正确的证书。

    
    
    --tls-san value   (listener) Add additional hostname or IP as a Subject Alternative Name in the TLS cert
    

bootstrap server 和其它 server 都需要配置样例的参数：

    
    
    k8e server --tls-san 192.168.1.1 
    

k8e 默认支持 flannel 网络，更换为 eBPF/cilium 网络，可如下配置启动加载：

注意主机系统必须满足： **Linux kernel >= 4.9.17**。

bootstrap server（172.25.1.55）：

    
    
    K8E_NODE_NAME=k8e-55  K8E_TOKEN=ilovek8e /opt/k8e/k8e server --flannel-backend=none --cluster-init --disable servicelb,traefik >> k8e.log 2>&1 &
    

server 2（172.25.1.56）：

    
    
    K8E_NODE_NAME=k8e-56 K8E_TOKEN=ilovek8e /opt/k8e/k8e server --server https://172.25.1.55:6443 --flannel-backend=none --disable servicelb,traefik >> k8e.log 2>&1 &
    

server 3（172.25.1.57）：

    
    
    K8E_NODE_NAME=k8e-57 K8E_TOKEN=ilovek8e /opt/k8e/k8e server --server https://172.25.1.57:6443 --flannel-backend=none --disable servicelb,traefik >> k8e.log 2>&1 &
    

安装 Cilium：

    
    
    helm install cilium cilium/cilium --version 1.9.5 --set global.etcd.enabled=true --set global.etcd.ssl=true  --set global.prometheus.enabled=true --set global.etcd.endpoints[0]=https://172.25.1.55:2379 --namespace kube-system
    

### 参考资料：

  * <https://github.com/xiaods/k8e/>

