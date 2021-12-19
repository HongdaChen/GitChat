在上一课中，我们解决了身份认证的问题，现在，我们能够保证访问集群的是合法用户，但是任何用户都可以访问集群的全部索引，在多个业务使用同一个集群时，可能某个业务的数据不希望被其他人看到，也不希望被其他人写入，删除数据，这就需要控制用户的权限，让某个用户只能访问某些索引。

在 X-Pack 中，这通过 RBAC 实现，他支持字段级别的权限配置，并且可以限制用户对集群本身的管理操作。

**X-Pack 中对用户进行授权包括两个步骤：**

  * 创建一个角色，为角色赋予某些权限。
  * 将用户映射到角色下。

### 1\. 内置角色

如同内置用户一样，X-Pack 中内置一些预定义角色，这些角色是只读的，不可修改。内置用户与角色的映射关系如下图所示：

![avatar](https://images.gitbook.cn/FsJW9j7QwD6qzLc-ycUwJkC-aawj)

**下面介绍一些常用的内置角色：**

  * **superuser**

对集群具有完全的访问权限，包括集群管理、所有的索引，模拟其他用户，并且可以管理其他用户和角色。默认情况下，只有 elastic 用户具有此角色。

  * **kibana_system**

具有读写Kibana索引，管理索引模板，检查集群可用性等权限，对 `.monitoring-*` 索引具有读取权限，对 `.reporting-*`
索引具有读写权限。

  * **logstash_system**

允许向 Elasticsearch 发送系统级数据，例如监控数据等。

### 2\. Security privilege

我们首先需要了解一下对 Elasticsearch 中的资源都有哪些权限，例如在 Linux
文件系统中，对文件有读/写，执行等权限，Elasticsearch 中资源的权限也有特定的名称，下面列出一些常用的
privilege，完整的列表请参考官方手册。

#### 2.1 Cluster privileges

对集群的操作权限常见的有下列类型：

  * **`all`**

所有集群管理操作，如快照、节点关闭/重启、更新设置、reroute 或管理用户和角色。

  * **`monitor`**

所有的对集群的只读操作，例如集群健康，集群状态，热点线程，节点信息、节点状态、集群状态，挂起的集群任务等。

  * **`manage`**

基于 monitor 权限，并增加了一些对集群的修改操作，例如快照，更新设置，reroute，获取快照及恢复状态，但不包括安全相关管理权限。

  * **`manage_index_templates`**

对索引模块的所有操作

  * **`manage_rollup`**

所有的 rollup 操作，包括创建、启动、停止和删除 rollup 作业。

  * **`manage_security`**

所有与安全相关的操作，如用户和角色上的 CRUD 操作和缓存清除。

  * **`transport_client`**

传输客户端所需的所有权限，在启用跨集群搜索时会用到。

#### 2.2 Indices privileges

  * **`all`**

对索引的所有操作。

  * **`create`**

索引文档，以及更新索引映射的权限。

  * **`create_index`**

创建索引的权限。创建索引请求可能包含添加到别名的信息。在这种情况下，请求还需要对相关索引和别名的 **manage** 权限。

  * **`delete`**

删除文档的权限。

  * **`delete_index`**

删除索引。

  * **`index`**

索引，及更新文档，以及更新索引映射的权限。

  * **`monitor`**

监控所需的所有操作（recovery, segments info, index stats and status）。

  * **`manage`**

在 **monitor** 权限基础上增加了索引管理权限（aliases, analyze, cache clear, close, delete,
exists, flush, mapping, open, force merge, refresh, settings, search shards,
templates, validate）。

  * **`read`**

只读操作（count, explain, get, mget, get indexed scripts, more like this, multi
percolate/search/termvector, percolate, scroll, clear_scroll, search, suggest,
tv）

  * **`read_cross_cluster`**

对来着远程集群的只读操作。

  * **`view_index_metadata`**

对索引元数据的只读操作（aliases, aliases exists, get index, exists, field mappings,
mappings, search shards, type exists, validate, warmers, settings, ilm）此权限主要给
Kibana 用户使用。

  * **write**

对索引的所有写操作，包括索引，更新，删除文档，bulk 操作，更新索引映射。

#### 2.3 Run as privilege

该权限允许一个已经进行了身份认证的合法用户代表另一个用户提交请求。取值可以是一个用户名，或用户名列表。

#### 2.4 Application privileges

应用程序权限在 Elasticsearch 管理，但是与 Elasticsearch 中的资源没有任何关系，他的目的是让应用程序在
Elasticsearch 的角色中表示和存储自己的权限模型。

### 3\. 定义新角色

你可以在 Kibana中 创建一个新角色，也可以用 REST API 来创建，为了比较直观的演示，我们先来看一下 Kibana
中创建角色的界面，如下图所示。

![avatar](https://images.gitbook.cn/FhIXy4wjTNTmdCqdqYqdeIxKFxFK)

需要填写的信息包括：

角色名称：此处我们定义为 weblog_user；

集群权限：可以多选，此处我们选择 monitor，让用户可以查看集群信息，也可以留空；

Run As 权限：不需要的话可以留空；

索引权限：填写索引名称，支持通配，再从索引权限下拉列表选择权限，可以多选。如果要为多个索引授权，通过 “Add index privilege”
点击按钮来添加。

如果需要控制字段级权限，在字段栏中填写字段名称，可以填写多个。

如果希望角色只能访问索引的部分文档怎么办？可以通过定义一个查询语句，让角色只能访问匹配查询结果的文档。

类似的，通过 REST API 创建新角色时的语法基本上就是上述要填写的内容：

    
    
    {
      "run_as": [ ... ], 
      "cluster": [ ... ], 
      "global": { ... }, 
      "indices": [ ... ], 
      "applications": [ ... ] 
    }
    

其中 global 只在 applications 权限中才可能会使用，因此暂时不用关心 global 字段与 applications
字段。indices 字段中需要描述对哪些索引拥有哪些权限，他有一个单独的语法结构：

    
    
    {
      "names": [ ... ], 
      "privileges": [ ... ], 
      "field_security" : { ... }, 
      "query": "..." 
    }
    

names：要对那些索引进行授权，支持索引名称表达式；

privileges：权限列表；

field_security：指定需要授权的字段；

query：指定一个查询语句，让角色只能访问匹配查询结果的文档；

引用官方的一个的例子如下：

    
    
    POST /_xpack/security/role/clicks_admin
    {
      "run_as": [ "clicks_watcher_1" ],
      "cluster": [ "monitor" ],
      "indices": [
        {
          "names": [ "events-*" ],
          "privileges": [ "read" ],
          "field_security" : {
            "grant" : [ "category", "@timestamp", "message" ]
          },
          "query": "{\"match\": {\"category\": \"click\"}}"
        }
      ]
    }
    

  * 创建的角色名称为 `clicks_admin`；
  * 以 `clicks_watcher_1` 身份执行请求；
  * 对集群有 `monitor` 权限；
  * 对索引 `events-*` 有 read 权限；
  * 查询语句指定，在匹配的索引中，只能读取 `category` 字段值为 `click` 的文档；
  * 在匹配的文档中只能读取 `category`，`@timestamp message` 三个字段；

除了使用 REST API 创建角色，你也可以把将新建的角色放到本地配置文件 `$ES_PATH_CONF/roles.yml` 中，配置的例子如下：

    
    
    click_admins:
      run_as: [ 'clicks_watcher_1' ]
      cluster: [ 'monitor' ]
      indices:
        - names: [ 'events-*' ]
          privileges: [ 'read' ]
          field_security:
            grant: ['category', '@timestamp', 'message' ]
          query: '{"match": {"category": "click"}}'
    

**总结一下 X-Pack 中创建角色的三种途径：**

  * 通过 REST API 来创建和管理，称为 Role management API，角色信息保存到 Elasticsearch 的一个名为 `.security-` 的索引中。这种方式的优点是角色集中存储，管理方便，缺点是如果需要维护的角色非常多，并且需要频繁操作时，REST 接口返回可能会比较慢，毕竟每个角色都需要一次单独的 REST 请求。

  * 通过记录到本地配置文件 `roles.yml` 中，然后自己用 Ansible 等工具同步到各个节点，Elasticsearch 会定期加载这个文件。

这种方式的优点是适合大量的角色更新操作，缺点是由于需要自己将 `roles.yml`
同步到集群的各个节点，同步过程中个别节点遇到的异常可能会导致部分节点的角色更新不够及时，最终表现是用户访问某些节点可以操作成功，而某些访问节点返回失败。

  * 通过 Kibana 界面来创建和管理，实际上是基于 Role management API 来实现的。

你也可以将角色信息存储到 Elasticsearch 之外的系统，例如存储到 S3，然后编写一个 Elasticsearch
插件来使用这些角色，这种方式的优点是角色集中存储，并且适合大量角色更新操作，缺点是你需要自己动手开发一个插件，并且对引入了新的外部依赖，对外部系统稳定性也有比较高的要求。

### 4\. 将用户映射到角色

经过上面步骤，我们已经创建了一个新角色 weblog_user，现在需要把某个用户映射到这个角色下。仍然以 Kibana 图形界面为例，我们可以直接在
Roles 的下拉列表中为用户选取角色，可以多选。

![](media/15474561832698/15498900722741.jpg)

这是一个非常简单的映射关系，但是映射方法很多：

  * 对于 native 或 file 两种类型 realms 进行验证的用户，需要使用 User Management APIs 或 users 命令行来映射。

  * 对于其他 realms ，需要使用 role mapping API 或者 一个本地文件 `role_mapping.yml` 来管理映射关系（早期的版本中只支持本地配置文件方式）。

  * role mapping API ：基于 REST 接口，映射信息保存到 Elasticsearch 的索引中。

  * 本地角色映射文件 `role_mapping.yml`：需要自行同步到集群的各个节点，Elasticsearch 定期加载。

通过REST API 或 `role_mapping.yml` 本地配置文件进行映射的优缺点与创建角色时使用 API
或本地文件两种方式优缺点相同，不再赘述。

使用角色映射文件时，需要在 `elasticsearch.yml` 中配置 `role_mapping.yml` 文件的路径，例如：

    
    
    xpack:
      security:
        authc:
          realms:
            ldap1:
              type: ldap
              order: 0
              url: "ldap://ldap.xxx.com:389"
              bind_dn: "cn=admin,dc=sys,dc=example, dc=com"
              user_search:
                base_dn: "dc=sys,dc=example,dc=com"
                filter: "(cn={0})"
              group_search:
                base_dn: "dc=sys,dc=example,dc=com"
              files:
                role_mapping: "ES_PATH_CONF/role_mapping.yml"
              unmapped_groups_as_roles: false
    

在 `role_mapping.yml` 配置中简单地描述某个角色下都有哪些用户，以 LDAP 用户为例：

    
    
    monitoring: 
      - "cn=admins,dc=example,dc=com" 
    weblog_user:
      - "cn=John Doe,cn=contractors,dc=example,dc=com" 
      - "cn=users,dc=example,dc=com"
      - "cn=admins,dc=example,dc=com"
    

上面的例子中，admins 组被映射到了 monitoring 和 `weblog_user` 两个角色，users 组和 John Doe 用户被映射到了
`weblog_user` 角色。这个例子使用角色映射 API 的话则需要执行以下两个：

    
    
    PUT _xpack/security/role_mapping/admins
    {
      "roles" : [ "monitoring", "weblog_user" ],
      "rules" : { "field" : { "groups" : "cn=admins,dc=example,dc=com" } },
      "enabled": true
    }
    
    
    
    PUT _xpack/security/role_mapping/basic_users
    {
      "roles" : [ "user" ],
      "rules" : { "any" : [
          { "field" : { "dn" : "cn=John Doe,cn=contractors,dc=example,dc=com" } },
          { "field" : { "groups" : "cn=users,dc=example,dc=com" } }
      ] },
      "enabled": true
    }
    

### 总结

本章重点介绍了 X-Pack 中创建角色，已经将用户映射到角色下的方法和注意事项，无论使用 REST
API还是使用本地配置文件，每种方式都有它的优缺点，如果你的环境中已经有一些文件同步任务，那么可以统一使用同步本地配置文件的方式，或者无论创建角色，还是映射角色，全都使用
REST API。

X-Pack
中无法为用户指定特定的索引模板权限，用户要么可以读写所有的模板，要么无法读写模板。而通常来说对于日志等按日期滚动生成索引的业务都需要先创建自己的索引模板，这可能是因为无法预期用户创建索引模板的时候会将模板匹配到哪些索引。

因此推荐不给业务分配索引模板写权限，由管理员角色的用户来检查索引模板规则，管理索引模板。

下一节我们介绍一下节点间的通讯如何加密，在早期版本中，这是一个可选项，现在的版本中要求必须开启。

### 参考

[Defining roles](https://www.elastic.co/guide/en/elastic-stack-
overview/6.5/defining-roles.html)

[Security privileges](https://www.elastic.co/guide/en/elastic-stack-
overview/current/security-privileges.html)

[Mapping users and groups to roles](https://www.elastic.co/guide/en/elastic-
stack-overview/6.5/mapping-roles.html)

[Custom roles provider extension](https://www.elastic.co/guide/en/elastic-
stack-overview/6.5/custom-roles-provider.html)

[Setting up field and document level
security](https://www.elastic.co/guide/en/elastic-stack-overview/6.5/field-
and-document-access-control.html)

