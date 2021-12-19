Kubernetes 集群内部跑的容器安全是通过 Security Context 和 Security Policy
来管理的。一般很少有人介绍这块的内容，大家在使用过程中通过模板复制就应用起来的，但是详细的分析并没有人来梳理，所以本篇的目的就是帮助大家一起梳理这块的概念。

### Pod Security Context

Kubernetes 中最小资源单位是 Pod，Pod 的安全配置是通过 Security Contexts 来配置的。Security Contexts
定义了每个容器的权限定义和访问控制。那么我们需要知道 Security Context 包括哪些东西才行。对于 Linux 环境来说，Security
Context 包括以下选项：

  * 用户 ID 和组 ID，用来控制访问对象或者文件的权限。755、644 等等，如果不太清楚，查询下 Linux 的用户权限定义就会更了解。
  * SELinux，给对象打上安全标签的服务。总结下来可以给所有文件打上细粒度的访问标签，控制力度更细。Linux 安全标配。
  * 运行在 privileged 或者 unprivileged，这是容器定义的变量，标识是否在特权下运行容器。
  * [Linux Capabilities](https://linux-audit.com/linux-capabilities-hardening-linux-binaries-by-removing-setuid/)， 这是 Linux 定义的安全能力，是把 privileged 赋予的权限细分了更多的类别，颗粒度更细的安全能力。
  * [AppArmor](https://kubernetes.io/docs/tutorials/clusters/apparmor/)，这是 Ubuntu 上配置的安全框架，也是用来控制 Linux Capabilities 的。
  * [Seccomp](https://en.wikipedia.org/wiki/Seccomp)，这是控制系统调用的安全框架，可以细粒度的过滤安全函数的调用。
  * AllowPrivilegeEscalation，是否允许子进程获得比父进程更多的权限。当容器配置了 Privileged 或者 CAP_SYS_ADMIN 权限后，AllowPrivilegeEscalation 也会变为 true。
  * readOnlyRootFilesystem，挂载容器的 root 系统为只读模式。

以上都是常用的安全环境配置，Kubernetes 支持的更详细的安全环境配置清单查询这里
[SecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-
api/v1.20/#securitycontext-v1-core)。

了解清楚定义后，以下范例给一个参考：

    
    
    apiVersion: v1
    kind: Pod
    metadata:
      name: security-context-demo-2
    spec:
      securityContext:
        runAsUser: 1000
      containers:
      - name: sec-ctx-demo-2
        image: gcr.io/google-samples/node-hello:1.0
        securityContext:
          runAsUser: 2000
          allowPrivilegeEscalation: false
    

### Pod Security Policy

Pod Security Policy 是集群级资源对象，控制 Pod 规范的安全敏感方面。策略类型按照高安全到灵活安全分为三级：特权级、基准、限制级。

#### **对于特权级的策略**

是有目的的开放，完全不受限制。这种类型的策略通常针对由特权、受信任的用户管理的系统和基础设施级工作负载。特权策略的定义是对于允许按默认执行机制（如守门人），特权配置文件是没有应用限制的策略。模板如下：

    
    
    apiVersion: policy/v1beta1
    kind: PodSecurityPolicy
    metadata:
      name: privileged
      annotations:
        seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
    spec:
      privileged: true
      allowPrivilegeEscalation: true
      allowedCapabilities:
      - '*'
      volumes:
      - '*'
      hostNetwork: true
      hostPorts:
      - min: 0
        max: 65535
      hostIPC: true
      hostPID: true
      runAsUser:
        rule: 'RunAsAny'
      seLinux:
        rule: 'RunAsAny'
      supplementalGroups:
        rule: 'RunAsAny'
      fsGroup:
        rule: 'RunAsAny'
    

#### **基准策略**

是便于采用常见的容器化工作负载，同时防止已知的特权升级。该策略针对的是应用程序操作员和非关键应用程序的开发人员。模板如下：

    
    
    apiVersion: policy/v1beta1
    kind: PodSecurityPolicy
    metadata:
      name: baseline
      annotations:
        # Optional: Allow the default AppArmor profile, requires setting the default.
        apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
        apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
        # Optional: Allow the default seccomp profile, requires setting the default.
        seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default,runtime/default,unconfined'
        seccomp.security.alpha.kubernetes.io/defaultProfileName:  'unconfined'
    spec:
      privileged: false
      # The moby default capability set, defined here:
      # https://github.com/moby/moby/blob/0a5cec2833f82a6ad797d70acbf9cbbaf8956017/oci/caps/defaults.go#L6-L19
      allowedCapabilities:
        - 'CHOWN'
        - 'DAC_OVERRIDE'
        - 'FSETID'
        - 'FOWNER'
        - 'MKNOD'
        - 'NET_RAW'
        - 'SETGID'
        - 'SETUID'
        - 'SETFCAP'
        - 'SETPCAP'
        - 'NET_BIND_SERVICE'
        - 'SYS_CHROOT'
        - 'KILL'
        - 'AUDIT_WRITE'
      # Allow all volume types except hostpath
      volumes:
        # 'core' volume types
        - 'configMap'
        - 'emptyDir'
        - 'projected'
        - 'secret'
        - 'downwardAPI'
        # Assume that persistentVolumes set up by the cluster admin are safe to use.
        - 'persistentVolumeClaim'
        # Allow all other non-hostpath volume types.
        - 'awsElasticBlockStore'
        - 'azureDisk'
        - 'azureFile'
        - 'cephFS'
        - 'cinder'
        - 'csi'
        - 'fc'
        - 'flexVolume'
        - 'flocker'
        - 'gcePersistentDisk'
        - 'gitRepo'
        - 'glusterfs'
        - 'iscsi'
        - 'nfs'
        - 'photonPersistentDisk'
        - 'portworxVolume'
        - 'quobyte'
        - 'rbd'
        - 'scaleIO'
        - 'storageos'
        - 'vsphereVolume'
      hostNetwork: false
      hostIPC: false
      hostPID: false
      readOnlyRootFilesystem: false
      runAsUser:
        rule: 'RunAsAny'
      seLinux:
        rule: 'RunAsAny'
      supplementalGroups:
        rule: 'RunAsAny'
      fsGroup:
        rule: 'RunAsAny'
    

#### **限制级别**

旨在执行当前的 Pod 硬化最佳实践，但要牺牲一些兼容性。它针对的是安全关键型应用程序的操作者和开发者，以及低信任度用户。例子如下：

    
    
    apiVersion: policy/v1beta1
    kind: PodSecurityPolicy
    metadata:
      name: restricted
      annotations:
        seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default,runtime/default'
        apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
        seccomp.security.alpha.kubernetes.io/defaultProfileName:  'runtime/default'
        apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
    spec:
      privileged: false
      # Required to prevent escalations to root.
      allowPrivilegeEscalation: false
      # This is redundant with non-root + disallow privilege escalation,
      # but we can provide it for defense in depth.
      requiredDropCapabilities:
        - ALL
      # Allow core volume types.
      volumes:
        - 'configMap'
        - 'emptyDir'
        - 'projected'
        - 'secret'
        - 'downwardAPI'
        # Assume that persistentVolumes set up by the cluster admin are safe to use.
        - 'persistentVolumeClaim'
      hostNetwork: false
      hostIPC: false
      hostPID: false
      runAsUser:
        # Require the container to run without root privileges.
        rule: 'MustRunAsNonRoot'
      seLinux:
        # This policy assumes the nodes are using AppArmor rather than SELinux.
        rule: 'RunAsAny'
      supplementalGroups:
        rule: 'MustRunAs'
        ranges:
          # Forbid adding the root group.
          - min: 1
            max: 65535
      fsGroup:
        rule: 'MustRunAs'
        ranges:
          # Forbid adding the root group.
          - min: 1
            max: 65535
      readOnlyRootFilesystem: false
    

### Kubernetes volumes 共享带来的问题

因为通过 Pod Security context 配置可以指定容器运行需要的 UID 和 GID。这样每个 Pod 指定的 UID
都是在自己的用户命令空间里面，保证了隔离。但是 Pod 还有大量的场景是需要挂载共享卷的。

考虑如下场景：

  * 容器 1 在 NFS 共享上写入文件。这些文件属于 ID 100000 的用户，因为这就是容器中映射的用户。
  * 容器 2 从 NFS 共享中读取文件。由于在容器 2 中没有映射用户 ID 100000，所以文件被视为属于伪用户 ID 65534，特殊代码为 "nobody"。这可能会引入一个文件访问权限问题。

![22-1-shared-nfs-
access.png](https://images.gitbook.cn/e707d850-4dc1-11eb-9891-63781b5fc5ff)

目前解决办法如下：对所有的 pods 使用相同的用户 id
映射。这降低了容器之间的隔离度，不过这仍然会比没有用户命名空间的现状提供更好的安全性。卷上的文件将由一个用户 id 拥有，比如
100000，管理员需要注意在卷的生命周期内，Kubernetes 配置中的用户 id 映射不会改变。

### 总结

对于 Kubernetes 来说，容器安全已经跨越容器升级为对容器组的管理。并且 Kubernetes
的安全策略控制的更多对象，比如存储卷的管理。所以这些基础安全知识还是要多查看并掌握。相信未来容器组的安全策略会越来越强大并支持更多样的配置策略组，免去复杂的配置选择。

