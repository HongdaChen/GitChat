本节课我们主要讨论节点间通讯加密的细节，以及如何配置，使用它们。TLS 使用 X.509
证书来加密通讯数据，为了避免攻击者伪造或篡改证书，还需要验证证书的有效性，与常见的 HTTPS 类似，这里使用 CA 证书来验证节点加密证书的合法性。

因此，当一个新的节点加入到集群时，如果他使用的证书使用相同的 CA 进行签名，节点就可以加入集群。这里涉及到两种证书，为了便于区分理解，暂且这样称呼他们：

  * 节点证书，用于节点间的通讯加密
  * CA 证书，用于验证节点证书的合法性

### 1\. 生成节点证书

Elasticsearch 提供了 elasticsearch-certutil 命令来简化证书的生成过程。它负责生成 CA 并使用 CA
签署证书。你也可以不生成新的 CA，而是使用已有的 CA 来签名证书。我们这里以生成新的 CA 为例。

**1\. 执行下面的命令可以生成 CA 证书：**

    
    
    bin/elasticsearch-certutil ca
    

该命令在当前路径下生成一个 PKCS#12 格式的单个文件，默认文件名为 `elastic-stack-ca.p12` 这是一种复合格式的文件，其中包含
CA 证书，以及私钥，这个私钥用于为节点证书进行签名。执行上述命令时会要求输入密码，用来保护这个复合格式的文件，密码可以直接回车留空。

**2\. 接下来执行下面的命令生成节点证书：**

    
    
    bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
    

`--ca` 参数用于指定 CA 证书文件。

该命令同样生成一个 PKCS#12 格式的单个文件，这个文件中包含了节点证书、节点秘钥、以及 CA 证书。此处同样会提示输入密码保护这个复合文件。

节点证书生成完毕后，你可以把他拷贝到每个 Elasticsearch 的节点上，每个节点使用相同的节点证书。

为了进一步增强安全性，也可以为每个节点生成各自的节点证书，这种情况下，需要在生成证书时指定每个节点的 ip 地址或者主机名称。这可以在
`elasticsearch-certutil cert` 命令添加 `--name`， `--dns` 或 `--ip` 参数来指定。

如果准备为每个节点使用相同的节点证书，也可以将上面的两个步骤合并为一个命令：

    
    
    bin/elasticsearch-certutil cert
    

上述命令会输出单个节点证书文件，其中包含自动生成的 CA 证书，以及节点证书，节点密钥。但不会输出单独的 CA 证书文件。

**3\. 将节点证书拷贝到各个节点**

将第二步生成的扩展名为 .p12 节点证书文件（默认文件名 elastic-certificates.p12）拷贝到 Elasticsearch 各个节点。

例如：`$ES_HOME/config/certs/` 后续我们需要在 `elasticsearch.yml`
中配置这个证书文件路径。注意只拷贝节点证书文件就可以，无需拷贝 CA 证书文件，因为 CA 证书的内容已经包含在节点证书文件中。

### 2\. 配置节点间通讯加密

节点证书生成完毕后，我们首先在 `elasticsearch.yml` 配置文件中指定证书文件的位置。

**1\. 启用 TLS 并指定节点的证书信息。**

根据节点证书的不同格式，需要进行不同的配置，以 PKCS#12 格式的证书为例，配置如下：

    
    
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate 
    xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12 
    xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12 
    

**`xpack.security.transport.ssl.verification_mode`**

验证所提供的证书是否由受信任的授权机构 (CA) 签名，但不执行任何主机名验证。如果生成节点证书时使用了 --dns 或 --ip 参数，则可以开启主机名或
ip 地址验证，这种情况下需要将本选项配置为 `full`

**`xpack.security.transport.ssl.keystore.path`**
**`xpack.security.transport.ssl.truststore.path`**

由于我们生成的节点证书中已经包含 CA 证书，因此 keystore 和 truststore
配置为相同值。如果为每个节点配置单独的证书，有可能在每个节点上使用不同的证书文件名，这会有些不方便管理，但是可以使用变量的方式，例如：`certs/${node.name}.p12`

如果生成证书的时候指定为 PEM 格式（在 `bin/elasticsearch-certutil cert`中添加 `-pem`
参数），则对应的配置略有不同：

    
    
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate 
    xpack.security.transport.ssl.key: /home/es/config/node01.key 
    xpack.security.transport.ssl.certificate: /home/es/config/node01.crt 
    xpack.security.transport.ssl.certificate_authorities: [ "/home/es/config/ca.crt" ] 
    

**`xpack.security.transport.ssl.key`**

指定节点秘钥文件路径；

**`xpack.security.transport.ssl.certificate`**

指定节点证书文件路径；

**`xpack.security.transport.ssl.certificate_authorities`**

指定 CA 证书文件路径；

**2\. 配置节点证书密码**

如果生成节点证书的时候设置了密码，还需要在 Elasticsearch keystore 中添加密码配置。

对于 PKCS#12 格式的证书，执行如下命令，以交互的方式配置证书密码：

    
    
    bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
    
    bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
    

对于 PEM 格式的证书，则应使用如下命令：

    
    
    bin/elasticsearch-keystore add xpack.security.transport.ssl.secure_key_passphrase
    

你无需再每个节点上以交互方式配置 Elasticsearch keystore，只需要在一个节点配置完毕后，将
`config/elasticsearch.keystore` 拷贝到各个节点的相同路径下覆盖文件即可。

**3\. 对集群执行完全重启**

要启用 TLS，必须对集群执行完全重启，启用 TLS 的节点无法与未启用 TLS 的节点通讯，因此无法通过滚动重启来生效。

### 3\. 加密与 HTTP 客户端的通讯

你同样也可以通过 TLS 来加密 HTTP 客户端与 Elasticsearch 节点之间的通讯，让客户端使用 HTTPS，而非 HTTP 来访问
Elasticsearch 节点。

这是可选项，不是必须的。当配置完节点间的通讯加密后，我们可以使用同一个节点证书来进行 HTTP，也就是 9200 端口加密。

**1\. 启用 TLS 并指定节点证书信息。**

对于 PKCS#12 格式的节点证书，我们在 `elasticsearch.yml` 中添加如下配置：

    
    
    xpack.security.http.ssl.enabled: true
    xpack.security.http.ssl.keystore.path: certs/elastic-certificates.p12 
    xpack.security.http.ssl.truststore.path: certs/elastic-certificates.p12 
    

配置项与节点间通讯加密中的类似，在此不再重复解释。类似的，对于 PEM 格式的证书，配置方式如下：

    
    
    xpack.security.http.ssl.enabled: true
    xpack.security.http.ssl.key:  /home/es/config/node01.key 
    xpack.security.http.ssl.certificate: /home/es/config/node01.crt 
    xpack.security.http.ssl.certificate_authorities: [ "/home/es/config/ca.crt" ] 
    

**2\. 在 Elasticsearch keystore 中配置证书密码**

与节点间通讯加密配置类似，对于 PKCS#12 格式的证书，执行如下命令，以交互的方式配置证书密码：

    
    
    bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password
    
    bin/elasticsearch-keystore add xpack.security.http.ssl.truststore.secure_password
    

对于 PEM 格式证书，使用如下命令进行配置：

    
    
    bin/elasticsearch-keystore add xpack.security.http.ssl.secure_key_passphrase
    

**3.重启节点**

修改完 `elasticsearch.yml` 配置文件后，需要重启节点来使他生效。此处可以使用滚动重启的方式，每个节点的 HTTP 加密是单独开启的。

如果只有部分节点配置了 HTTP 加密，那么没有配置 HTTP 加密的节点只能通过 HTTP 协议来访问节点。

当 HTTP 加密生效后，客户端只能通过 HTTPS 协议来访问节点。

### 4\. 加密与活动目录或 LDAP 服务器之间的通讯

为了加密 Elasticsearch 节点发送到认证服务器的用户信息，官方建议开启 Elasticsearch 到活动目录（AD）或 LDAP 之间的通讯。

通过 TLS/SSL 连接可以对 AD 或 LDAP 的身份进行认证，防止中间人攻击，并对传输的数据进行加密。因此这是一种服务端认证，确保 AD 或
LDAP 是合法的服务器。

这需要在 Elasticsearch 节点中配置 AD 或 LDAP 服务器证书或服务器 CA 根证书。同时将 ldap 改为 ldaps。例如：

    
    
    xpack:
      security:
        authc:
          realms:
            ldap1:
              type: ldap
              order: 0
              url: "ldaps://ldap.example.com:636"
              ssl:
                certificate_authorities: [ "ES_PATH_CONF/cacert.pem" ]
    

### 总结

本章重点叙述了开启节点间通讯加密的具体操作方式，由于这是开启 X-Pack 安全特性的必须项，因此需要读者理解节点证书与 CA
证书，大家可以按自己的安全性需求为节点配置不同证书或者相同证书，HTTP 加密虽然不是必须，但是官方极力推荐开启。

到本节为止，安全这部分主题介绍完毕，本课程的最后一部分与大家分享一些我们在多年运维过程中的常见问题，给读者一些借鉴和参考，这些问题都是比较通用的。

### 参考

[1] [Installing X-Pack in
Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/installing-
xpack-es.html)

[2] [Setting Up TLS on a
Cluster](https://www.elastic.co/guide/en/x-pack/6.2/ssl-tls.html)

[3] [Encrypting communications in
Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/configuring-
tls.html#tls-transport)

[4][Security settings in
Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/security-
settings.html#ssl-tls-settings)

