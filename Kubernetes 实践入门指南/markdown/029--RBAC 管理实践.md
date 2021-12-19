Kubernetes 引入基于角色的授权管理机制（RBAC）由来已久，在最近的 1.18
版本中更是推荐所有场景下都应该使用此特性管理集群。所以，我们有必要尽快了解并熟悉这套管理授权的机制和方法。

RBAC 定义了如下术语：

  * 实体：可以是用户，比如你或者同事。也可以是一个 Pod 或者外面的第三方服务。总之代表需要访问 Kubernetes 内部对象资源的使用者。
  * 资源：可以是 Pod、ConfigMap、Secret 等对象
  * 角色：因为我们如果想把一个用户的权限规则复制到其它用户身上会有很多工作量，所以我们可以把权限规则定义成一个角色，然后可以按照需要加入或者从这个角色中移出用户。
  * 角色绑定：这个就是角色和用户直接的实际关联关系

Kubernetes 默认提供了一些角色如下：

ClusterRole | ClusterRoleBinding | Description  
---|---|---  
cluster-admin | system:masters | 这个角色可以被看作是 Linux 机器上的 root 用户。请注意，当与
RoleBinding 一起使用时，它赋予用户对该 RoleBinding
命名空间下的资源的完全访问权，包括命名空间本身（但不包括其他命名空间的资源）。但是，当与 ClusterRoleBinding
一起使用时，它赋予用户对整个集群中的每个资源（所有命名空间）的超级控制权。  
admin | None | 与 RoleBinding
一起使用，它使用户可以完全访问该命名空间内的资源，包括其他角色和角色绑定。但是，它不允许写访问资源配额或命名空间本身。  
edit | None | 与 RoleBinding 一起使用，除了查看或修改角色和角色绑定外，它授予用户与管理员在给定命名空间中的访问级别相同。  
view | None | 与 RoleBinding 一起使用，它授予用户对大多数命名空间资源的只读访问权限，但 Roles、RoleBindings 和
Secrets 除外。  
  
### 身份认证策略

#### **X509 客户证书方式**

为了更好地理解 RBAC，让我们从案例中学习体会。

Case1：添加一个管理员用户 Admin。

1\. 您需要从这里下载并安装 CFSSL 工具

> <https://pkg.cfssl.org/>

2\. 创建一个证书签署请求的 JSON 文件 **user.json** 如下：

    
    
    {
        "CN": "alice",
        "key": {
            "algo": "rsa",
            "size": 4096
        },
        "names": [{
            "O": "alice",
            "email": "alice@mycompany.com"
        }]
    }
    

运行文件中生成 CSR（证书签名请求）命令并输出结果如下所示：

    
    
    $ cfssl genkey user.json | cfssljson -bare client
    2019/11/09 18:14:33 [INFO] generate received request
    2019/11/09 18:14:33 [INFO] received CSR
    2019/11/09 18:14:33 [INFO] generating key: rsa-4096
    2019/11/09 18:14:34 [INFO] encoded CSR
    

3\. 执行第二步命令创建一个 client.csr 文件，包含了 csr 数据。还有一个密钥文件 client-
key.pem，其中包含用于签署请求的私钥。

4\. 使用以下命令将 csr 请求转换为 Base64 编码：`cat client.csr | base64 | tr -d
'\n'`。保留一个文本的副本，因为我们将在下一步中使用它。

5\. 创建一个 CertificateSigningRequest 资源，方法是创建一个 csr.yam 文件，并添加以下行：

    
    
    apiVersion: certificates.k8s.io/v1beta1
    kind: CertificateSigningRequest
    metadata:
     name: alice
    spec:
     groups:
     - mycompany
     request: LS0tLS1CRUdJTiBDRVJUSUZJQ0F...   #< 这里添加第 4 步生成的 base64 编码的 csr 文件
     usages:
     - digital signature
     - key encipherment
     - client auth
    

6\. 使用 kubectl 向 API 服务器发送请求，方法如下：

    
    
       $ kubectl apply -f request.yaml
       certificatesigningrequest.certificates.k8s.io/alice created
    

7\. 作为集群管理员，可以使用以下命令批准这个证书请求：

    
    
       $ kubectl certificate approve alice
       certificatesigningrequest.certificates.k8s.io/alice approved
    

8\. 既然 csr 已经被批准了，我们需要下载实际的证书（注意，输出已经是 Base64 编码的。我们不需要解密它，因为我们将在以后的
kubeconfig 文件中以同样的形式再次使用它）:

    
    
       kubectl get csr alice -o jsonpath='{.status.certificate}'
       LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1...
    

9\. 现在，Alice 已经批准了她的证书。要使用它，她需要一个 kubeconfig 文件，除了 API 服务器的 API
之外，还需要一个引用她的证书、私钥和集群 CA 的 kubeconfig 文件，这个文件是用来签署这个请求的。我们已经有了私钥和证书，让我们从现有的
kubeconfig 文件中获取其他信息：

    
    
       $ kubectl config view --flatten --minify
       apiVersion: v1
       clusters:
       - cluster:
           certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0t...
           server: https://104.198.41.185
       --- the rest of the output was trimmed for brevity ---
    

10\. 鉴于我们现在所掌握的所有信息，我们可以为 Alice 创建一个包含以下几行的配置文件（在添加到配置文件之前，确保客户端-证书-日期、客户端-密钥-
数据和证书-权限-数据是 Base64 编码的）:

    
    
        apiVersion: v1
        kind: Config
        users:
        - name: alice
          user:
            client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ…
            client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFU...
        clusters:
        - cluster:
            certificate-authority-data: LS0tLS1CRUdJTiBDR...
            server: https://104.198.41.185
          name: gke
        contexts:
        - context:
            cluster: gke
            user: alice
          name: alice-context
        current-context: alice-context
    

最后一步是把 kube_config 文件交给 Alice，让她把它添加到~/.kube/config
下。然而，我们可以通过传递我们刚刚创建的配置文件来验证她的凭证是否有效（假设我们将其命名为 alice_config）。让我们尝试着列举一下 pods：

    
    
    $ kubectl  get pods --kubeconfig ./alice_config
    Error from server (Forbidden): pods is forbidden: User "alice" cannot list resource "pods" in API group "" in the namespace "default"
    

我们收到的结果提示是非常重要的，因为它可以让我们验证 API 服务器确实承认证书。输出显示 `"Forbidden"`，这意味着 API 服务器承认有一个叫
Alice 的用户存在。但是，她没有权限查看默认命名空间上的 pods。现在让我们给她必要的权限。

由于我们需要授予 Alice 整个集群的管理员权限，所以我们可以使用现成的 Cluster-admin 角色。因此，我们只需要一个
ClusterRoleBinding 资源。创建一个 YAML 文件，包含以下几行：

    
    
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1beta1
    metadata:
     name: cluster-admin-binding
    subjects:
    - kind: User
     name: alice
     apiGroup: ""
    roleRef:
     kind: ClusterRole
     name: cluster-admin
     apiGroup: ""
    

使用 kubectl 应用上述 YAML 文件：

    
    
    $ kubectl apply -f clusterrolebinding.yml
    clusterrolebinding.rbac.authorization.k8s.io/cluser-admin-binding created
    

现在，让我们来仔细检查一下 Alice 是否可以对集群执行命令：

    
    
    $ kubectl  get pods --kubeconfig ./alice_config
    NAME      READY   STATUS    RESTARTS   AGE
    testpod   1/1     Running   105        29h
    $ kubectl  get nodes --kubeconfig ./alice_config
    NAME                                          STATUS   ROLES    AGE   VERSION
    gke-security-lab-default-pool-46f98c95-qsdj   Ready       46h   v1.13.11-gke.9
    $ kubectl  get pods -n kube-system --kubeconfig ./alice_config
    NAME                                                     READY   STATUS    RESTARTS   AGE
    event-exporter-v0.2.4-5f88c66fb7-6l485                   2/2     Running   0          46h
    fluentd-gcp-scaler-59b7b75cd7-858kx                      1/1     Running   0          46h
    fluentd-gcp-v3.2.0-5xlw5                                 2/2     Running   0          46h
    heapster-5cb64d955f-mvnhb                                3/3     Running   0          46h
    kube-dns-79868f54c5-kv7tk                                4/4     Running   0          46h
    kube-dns-autoscaler-bb58c6784-892sv                      1/1     Running   0          46h
    kube-proxy-gke-security-lab-default-pool-46f98c95-qsdj   1/1     Running   0          46h
    l7-default-backend-fd59995cd-gzvnj                       1/1     Running   0          46h
    metrics-server-v0.3.1-57c75779f-dfjlj                    2/2     Running   0          46h
    prometheus-to-sd-k6627     
    

通过上面的几个命令，Alice 可以查看多个命名空间中的 Pod 容器组，同时也可以获得集群节点的相关信息。成功！

#### **启动引导令牌方式**

Kubernetes 包含了一种动态管理的持有者令牌类型，称为 **启动引导令牌（Bootstrap Token）** 。这些令牌以 Secret
的形式保存在 kube-system 名字空间中，可以被动态管理和创建。 控制管理器包含的 TokenCleaner
控制器能够在启动引导令牌过期时将其删除。

这些令牌的格式为 `[a-z0-9]{6}.[a-z0-9]{16}`。第一个部分是令牌的 ID；第二个部分 是令牌的
Secret。启动引导令牌认证组件可以通过 API 服务器上的如下标志启用：

    
    
    --enable-bootstrap-token-auth
    

你可以用如下所示的方式来在 HTTP 头部设置令牌：

    
    
    Authorization: Bearer 781292.db7bc3a58fc5f07e
    

启动引导令牌被定义成一个特定类型的 Secret （bootstrap.kubernetes.io/token）， 并存在于 kube-system
名字空间中。 这些 Secret 会被 API 服务器上的启动引导认证组件（Bootstrap Authenticator）读取。 控制管理器中的控制器
TokenCleaner 能够删除过期的令牌。 这些令牌被用来在节点发现的过程中当成特殊的 ConfigMap 对象。 BootstrapSigner
控制器也会使用这一 ConfigMap。

使用过社区内置的 Dashboard 的读者一定见过如下管理界面，显示用 Token 来登录：

![k8s
credentials](https://images.gitbook.cn/4a6e7ea0-4dca-11eb-a007-1b095f083930)

    
    
    TOKEN=$(kubectl -n kube-system describe secret default| awk '$1=="token:"{print $2}')
    kubectl config set-credentials docker-for-desktop --token="${TOKEN}"
    echo $TOKEN
    

其中 docker-for-desktop 是你的 contexts 的名字，可以通过如下命令获得：

    
    
    kubectl config current-context
    

#### **服务账号令牌方式**

服务账号（Service Account）是一种自动被启用的用户认证机制，使用经过签名的持有者令牌来验证请求。服务账号通常由 API 服务器自动创建并通过
ServiceAccount 准入控制器关联到集群中运行的 Pod 上。 持有者令牌会挂载到 Pod 中可预知的位置，允许集群内进程与 API 服务器通信。
服务账号也可以使用 Pod 规约的 serviceAccountName 字段显式地关联到 Pod 上。范例如下：

    
    
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
      namespace: default
    spec:
      replicas: 3
      template:
        metadata:
        spec:
          serviceAccountName: bob-the-bot
          containers:
          - name: nginx
            image: nginx:1.14.2
    

外部想使用服务账号也是可以的，需要提前在 Kubernetes 中创建服务账号。可以使用 `kubectl create serviceaccount
<名称>` 命令。此命令会在当前的名字空间中生成一个服务账号和一个与之关联的 Secret。

服务账号被身份认证后，所确定的用户名为 `system:serviceaccount:<名字空间>:<服务账号>`， 并被分配到用户组
`system:serviceaccounts` 和 `system:serviceaccounts:<名字空间>`。

**注意** ：由于服务账号令牌保存在 Secret 对象中，任何能够读取这些 Secret 的用户
都可以被认证为对应的服务账号。在为用户授予访问服务账号的权限时，以及对 Secret 的读权限时，要格外小心。

#### **OpenID Connect（OIDC）令牌方式**

OpenID Connect 是一种 OAuth2 认证方式， 被某些 OAuth2 提供者支持，例如 Azure 活动目录、Salesforce 和
Google。由于用来验证你是谁的所有数据都在 id_token 中，Kubernetes
不需要再去联系身份服务。在一个所有请求都是无状态请求的模型中，这一工作方式可以使得身份认证 的解决方案更容易处理大规模请求。不过，此访问也有一些挑战：

  * Kubernetes 没有提供用来触发身份认证过程的 "Web 界面"。 由于不存在用来收集用户的接口，你必须自己先行完成对身份服务的认证过程。
  * id_token 令牌不可收回。因其属性类似于证书，其生命期一般很短（只有几分钟）， 所以，每隔几分钟就要获得一个新的令牌也非常麻烦。
  * 如果不使用 `kubectl proxy` 命令或者一个能够注入 id_token 的反向代理， 向 Kubernetes 控制面板执行身份认证是很困难的。

由于这种认证方式是特定场景用途，实践意义不大，如需要查阅具体细节可以参考[官方文档](https://kubernetes.io/zh/docs/reference/access-
authn-authz/authentication/)。

#### **Webhook 令牌身份认证方式**

Webhook 身份认证是一种用来验证持有者令牌的回调机制。配置文件使用 kubeconfig 文件的格式。文件中，clusters
指代远程服务，users 指代远程 API 服务 Webhook。下面是一个例子：

    
    
    # Kubernetes API 版本
    apiVersion: v1
    # API 对象类别
    kind: Config
    # clusters 指代远程服务
    clusters:
      - name: name-of-remote-authn-service
        cluster:
          certificate-authority: /path/to/ca.pem         # 用来验证远程服务的 CA
          server: https://authn.example.com/authenticate # 要查询的远程服务 URL。必须使用 'https'。
    
    # users 指代 API 服务的 Webhook 配置
    users:
      - name: name-of-api-server
        user:
          client-certificate: /path/to/cert.pem # Webhook 插件要使用的证书
          client-key: /path/to/key.pem          # 与证书匹配的密钥
    
    # kubeconfig 文件需要一个上下文（Context），此上下文用于本 API 服务器
    current-context: webhook
    contexts:
    - context:
        cluster: name-of-remote-authn-service
        user: name-of-api-sever
      name: webhook
    

用户使用令牌完成身份认证时，身份认证 Webhook 会用 POST 请求发送一个 JSON 序列化的对象到远程服务。

### 匿名请求

启用匿名请求支持之后，如果请求没有被已配置的其他身份认证方法拒绝，则被视作 匿名请求（Anonymous Requests）。这类请求获得用户名
`system:anonymous` 和对应的用户组 `system:unauthenticated`。

### 用户伪装

一个用户可以通过伪装（Impersonation）头部字段来以另一个用户的身份执行操作。例如，管理员可以使用这一功能特性来临时伪装成另一个用户，查看请求是否被拒绝，从而调试鉴权策略中的问题。

在使用 kubectl 时，可以使用 `--as` 标志来配置 Impersonate-User 头部字段值， 使用 `--as-group` 标志配置
Impersonate-Group 头部字段值。

    
    
    kubectl drain mynode --as=superman --as-group=system:masters
    
    node/mynode cordoned
    node/mynode drained
    

对于启用了 RBAC 鉴权插件的集群，下面的 ClusterRole 封装了设置用户和组伪装字段 所需的规则：

    
    
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: impersonator
    rules:
    - apiGroups: [""]
      resources: ["users", "groups", "serviceaccounts"]
      verbs: ["impersonate"]
    

### 总结

围绕着 RBAC 角色定义的身份认证策略很多，从实践的角度来说，对于集群内部对象如 Deployment、StatefulSet 等需要和 API
服务交互的时候推荐使用服务账号体系来管理。对于外部用户来说，采用 X509 客户证书来管理用户更为实际可操作。对于企业级环境，需要统一认证的才需要关注
OIDC 令牌、Webhook 令牌、client-go 凭据插件等。因为场景的不同，实现的路径也有很多选择可供参考，具体细节可以参考官方文档作为支持。

### 参考：

  * <https://kubernetes.io/zh/docs/reference/access-authn-authz/authorization/>

