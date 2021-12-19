上一篇咱们好好聊了聊跳表结构，它是 HNSW 的基础之一。

HNSW 的全称为 Hierarchical Navigable Small World
Graphs，中间三个单词咱不管，第一个和最后一个单词的意思是分层的图结构，可见每一层都是一张图结构，正是这一层层（是不是想到了跳表的层？）的图结构，实现了
HNSW 的高效近邻搜索。这是一种什么样的图呢？Navigable Small World —— NSW$^1$。

NSW 本身也是一种近邻搜索算法，它之于 HNSW 就像链表之于跳表。我们首先来好好介绍下 NSW 算法，一旦掌握了它，那么 HNSW 就掌握了大半了。

### NSW

作为近邻搜索算法，那么就最好能满足以下几个要求：

  * 要快：查找效率高，要求有一种快速找到近邻的机制。
  * 要准：给出的近邻点尽可能为真实的近邻点，要求相近的点尽可能互为“友点”。
  * 要稳：不能有查不到近邻点的情况，要求每个点都得有“友点”，不能存在”孤点“。

这三个条件 NSW 是如何一一满足的呢？

![Fig1](https://images.gitbook.cn/24fa1740-3170-11eb-9e6e-492b4066adc9)

如上图所示，所有点位于一张图中，图的蓝色节点（node）代表了真实的数据点，顶点与顶点之间的黑色边（egde）代表了数据之间的连接（connection），红色线代表了两个节点之间存在”高速公路/远程连接“（long
range link，一会儿会细说）。有连接的两个点被称为“友点（friends）”。

了解了这几个基本概念之后，我们先厘清 NSW 的最近邻搜索逻辑，然后再回头来聊聊这个“ **高速公路** ”机制。

#### **最近邻搜索逻辑**

假设数据已经根据某种规则形成了一张类似上图的 GRAPH（后面会说这个 GRAPH 是怎么形成的，别着急），根据
[paper](https://www.hse.ru/data/2015/03/13/1325528089/Approximate%20nearest%20neighbor%20algorithm%20b..navigable%20\(Information%20Systems\).pdf)
第 4.2 小节的算法描述，其搜索逻辑如下：

    
    
    k-NNSearch(q, m, k)
    

  * 输入：待搜索近邻点 q，整数 m 和 k
  * 输出：q 的 k 个最近邻点
  * 临时变量：tempRes，candidates，visitedSet，result

    
    
    for (i = 0; i < m; i++):
    
        从所有数据点中随机抽取一个点 entry point，放入 candidates // 1
    
    ​    tempRes = null // 2
    
        repeat: // 3
    
    ​         从 candidates 中获取离 q 最近的点 c // 4
    
            从 candidates 中删除点 c // 5
    
            如果点 c 比 result 中第 k 个点还远: // 6
    
                break repeat // 7
    
            for friend in c.friends: // 8
    
                如果 c 的 friend 不在 visitedSet 中，则: // 9
    
                    将 friend 放入 visitedSet，candidates，tempRes // 10
    
        end repeat // 11
    
    ​    将本次循环中 tempRes 中的结果添加到 result 中 // 12
    
    end for // 13
    
    返回 result 中距离最小的 k 个点作为最近邻点 // 14
    

注意 m 和 k 的含义即可，这个搜索的逻辑还是比较直观的。

在弄清楚 NSW 最近邻搜索的逻辑之后，就可以来看看其中非常特别的“高速公路”机制。

#### **高速公路**

对照上图和上述算法逻辑。

目标是查找绿色点 query 的最近邻，m 和 k 均设为 1，查找开始（这里为了简单起见，所有的数据结构均画成数组形式，实际上是通过大/小顶堆来实现的）。

第 1、2 行：假定初始 entry point 点为 A 点，放入 candidates，tempRes 为空。

![](https://images.gitbook.cn/a26cff10-317c-11eb-bb22-a33d89826e30)

第 4、5 行：此时 candidates 中只有 A 点，因此变量 c 就是 A 点，取出 A 点，随后从 candidates 中删除 A 点。

第 8、9、10 行：遍历 A 的友点 B、E、F。

![](https://images.gitbook.cn/b5558ac0-317c-11eb-a3fe-4f57d7580ecb)

继续 repeat 又到第 4、5 行：从 candidates 中取出离 query 最近的点，B 点，随后从 candidates 中删除 B 点。

第 8、9、10 行：遍历 B 的友点 A、C、F、H、I、J。

![](https://images.gitbook.cn/c7c4a0b0-317c-11eb-83f3-59966fe2e163)

继续 repeat 又回到第 4、5 行：从 candidates 中取出离 query 最近的点，C 点，随后从 candidates 中删除 C 点。

第 8、9、10 行：遍历 C 的友点 B、D、K、M。

![](https://images.gitbook.cn/d7d6dae0-317c-11eb-bb22-a33d89826e30)

继续 repeat 又到第 4、5 行：从 candidates 中取出离 query 最近的点，D 点，随后从 candidates 中删除 D 点。

第 8、9、10 行：遍历 D 的友点 C、J、K、L、M。

此时 repeat 应该终止，因为 D 的友点都比 D 点离 query 远，但是算法描述中的终止条件（第 6 行）还没达到（因为此时 result
还是空的）。所以这个算法还存在一点小问题，终止条件似乎并不完善。这一点在 HNSW 的 paper 中得到了修正，后文再细说。

假设此时 repeat 结束（因为 D 的友点都比 D 点离 query 远），找到了最近邻点 D。

可以通过上述流程看到“高速公路”的好处，如果没有它，从 A 点到 B 点再到 C 点到 D
点可能要经过很多步。而现在只需要少数几个节点间的跳转就能到达目的地，查找效率大大提升。

现在我们已经掌握了最近邻的查找逻辑，那么 NSW 的 GRAPH 是如何形成的呢？

#### GRAPH 构建逻辑

图的构建逻辑，其实就是节点插入逻辑。

根据
[paper](https://www.hse.ru/data/2015/03/13/1325528089/Approximate%20nearest%20neighbor%20algorithm%20b..navigable%20\(Information%20Systems\).pdf)
第 5 小节的算法描述，节点插入逻辑如下：

    
    
    Nearest_Neighbor_Insert(q, w, f) // w 对应 k-NNSearch 中的 m，f 对应 k
    
    // 先查找到 q 的最近邻 neighbors = k-NNSearch(q, w, f)
    
    for (i = 0; i < f; i++) {
    
        // 将节点 q 和 节点 neighbors[i] 连接起来
    
        neighbors[i].connect(q) 
    
    ​    q.connect(neighbors[i])
    
    }
    

是不是很简单？

#### **高速公路的形成与消失**

现在我们以 5 个点来演示一遍 NSW GRAPH 的构建并以此来说明 GRAPH 中最为重要的“高速公路”，它的形成以及消失。

5 个点在空间中的分布如下所示：

![](https://images.gitbook.cn/07b50930-317d-11eb-9df3-714c11e54f62)

假定每个节点最多只能有 2 个邻居/友点。

首先 A 节点插入到 GRAPH 中，A 是第一个插入的节点，所以 GRAPH 如下：

![](https://images.gitbook.cn/171aafb0-317d-11eb-b804-ef8651c2512a)

B 插入，此时只有一个 A 点，于是 A 添加 B 为邻居节点，B 也添加 A 为邻居节点（双向箭头表示 A、B 互为邻居节点）。

![](https://images.gitbook.cn/21497f70-317d-11eb-90f6-fbd19bda6e6e)

C 插入，此时图中有 A 点和 B 点，且 A 和 B 的邻居均小于 2 个，于是 C 将 A、B 作为邻居节点，A 和 B 也各自将 C 添加为邻居节点。

![](https://images.gitbook.cn/2aa12910-317d-11eb-9df3-714c11e54f62)

D 插入，此时图上有 3 个节点，根据节点插入逻辑，D 要先找到其最近邻点，然后再与近邻点形成连接。通过 kNNSearch 算法，D 找到了 A 点和 C
点，于是 D 与 A、C 形成连接。问题随之而来：

  * A 此时的邻居数大于 2 个了（B、C、D），需要丢弃一个——显然 A 丢弃了 B，因为在 B、C、D 3 个邻居中，B 最远；
  * C 此时的邻居数也大于 2 个了（A、B、D），C 丢弃了 B。

此时的 GRAPH 如下所示：

![](https://images.gitbook.cn/408d30c0-317d-11eb-9be5-1150cb1fb404)

红色线表示原先 GRAPH 中的邻居关系发生了变化，此时 B 到 A、C 均边成了单向箭头， **A 到 B 的高速公路消失了** 、 **C 到 B
的高速公路消失了** 。

E 插入，E 要先找到其最近邻的 2 个点，得到 A 和 D。因为 A 和 D 已经有了 2 个邻居了，于是这两个点又要再次更新：

  * A 点会丢弃 C 点，意味着 **A 到 C 的高速公路也消失了** ；
  * D 点会丢弃 C 点。

![](https://images.gitbook.cn/54d6fa20-317d-11eb-9be5-1150cb1fb404)

于是我们会得到这样的结论：

  * ”高速公路“机制在 GRAPH 构建的早期更容易产生；
  * 随着加入 GRAPH 的节点越来越多，“高速公路”越来越少。

而我们知道，“高速公路”机制在提高近邻搜索效率中发挥着至关重要的作用，有没有办法——

**让“高速公路”即使在节点越来越多的情况下，也不会减少呢？**

有！HNSW！

### HNSW

还记得跳表吗？上一篇我们详细讲述了跳表的相关知识，如果有同学忘记的话，可以再回去温习一遍哦。

“高速公路”越来越少的原因在于插入的节点越来越多，于是 HNSW 的作者就想到了一个绝妙的想法——

**将跳表与 NSW 结合起来** 。

如下图所示：

![](https://images.gitbook.cn/674ee9b0-317d-11eb-bb22-a33d89826e30)

是不是高维空间的跳表结构？

图中蓝色节点为数据节点，绿色节点为待查询近邻节点，红色线为高速公路，黑色线为邻居连接。

每个节点会被随机分层，最底层（第 0 层）含有所有的数据节点，每往上一层，节点数以指数量级减少。每一层都是一张 NSW GRAPH。

在上层结构中，“高速公路”机制可以很好的发挥作用，能够快速定位到 **待查询节点附近**
，越往下层查找越精准，最终在最底层时，候选节点的数量不仅不会太多，而且都在带查询节点的附近，也即不仅保证了速度、也保证了精度。

到这里 HNSW 算法基本上也就讲述得差不多了，掌握了 NSW 之后，再来看 HNSW 会容易很多，每一层的操作与普通的 NSW 别无二致。

上文中提到的 **查找终止条件** 问题，在 HNSW 的 [paper](https://arxiv.org/pdf/1603.09320.pdf)
里做了修正，在第 4 节的 **Algorithm 2 SEARCH_LAYER** 算法中，跳出循环的条件是： **当前节点的邻居节点与 query
的距离都比 当前节点与 query 的距离远** 。

HNSW 的两篇 paper 非常值得一读，其中的思想也值得我们在工作中借鉴。下面我们再来简单的看看 HNSW 在实际工作中是如何使用的。

### 应用

作为 ANN 算法的一种，其用途与 LSH 没有太大的区别。我们可以将其用途分为两种：

  * 离线情况下，根据海量商品的 embedding 去计算每个商品的 topN 相似商品。
  * 在线情况下，在《看看 Youtube 怎么召回的》这篇文章中，我们提到：双塔结构中的 user vector 根据模型实时推理出来，然后使用 user vector 去海量的商品向量索引中去找最相似/最感兴趣的 topN 商品。

我们可以认为 user vector 中的每个维度代表的意思是：用户对该维度所代表商品属性的喜好程度（虽然不知道这个维度是什么），item vector
中的每个维度代表的意思是：该商品在该维度所代表商品属性的比重。user vector 某个维度的值越大，我们就希望推给 TA 的 item vector
对应维度的值越大。

那么，需要我们自己实现吗？当然不需要！GitHub 上有很多优秀的实现。

> HNSW java/scala/scala-spark/pyspark
> 实现的地址在[这里](https://github.com/jelmerk/hnswlib)，C++
> 实现的地址在[这里](https://github.com/nmslib/hnswlib)。

在这里特别推荐同学们好好研究下 HNSW 的源码，特别值得研究。

> [Java 代码](https://github.com/jelmerk/hnswlib/blob/master/hnswlib-
> core/src/main/java/com/github/jelmerk/knn/hnsw/HnswIndex.java) HnswIndex 类中的
> #add 方法是整个算法的 核心，有兴趣的一定要仔细研读这个方法。

### 小结

用了两个篇幅，我们详细介绍了 HNSW 这个性能卓越的 ANN 算法，可以看到其特别适用于在线环境，对延迟要求极高的常见中。

我们知道了跳表这种简单高效的数据结构，发现只要用额外 O(N) 的空间复杂度，就能够实现 O(logN) 的查询时间复杂度，这种空间换时间实在太划算了。

希望这两篇文章结束后，同学们对于一般双塔结构的召回套路，包括其 training、serving
都能够有比较好的掌握。在实际工作中，没条件也要尽量创建条件去尝试。

在将双塔模型和 HNSW 运用到地鼠商城的推荐系统之后，其线上指标完爆了之前的所有算法，虽然丢失了可解释性，但是当 boss
看到每日带来的收入的增长，谁还会去关心可解释性呢，毕竟谁会跟 money 过去呢？

讲了这么多的召回模型，好像我们一直都没有提到一个很重要的问题：

> 这些模型总不能训练出来就直接上线吧？不然这风险性也太高了。

是的，接下来，咱们好好聊聊——这些召回模型如何在线下评估，什么模型该用什么指标等问题。

下篇见。

注释 1：NSW 算法的 paper
[地址](https://www.hse.ru/data/2015/03/13/1325528089/Approximate%20nearest%20neighbor%20algorithm%20b..navigable%20\(Information%20Systems\).pdf)

注释 2：HNSW 算法 paper 地址在[这里](https://arxiv.org/pdf/1603.09320.pdf)

