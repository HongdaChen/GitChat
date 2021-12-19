在 Web 框架中，路由器也称为多路复用器，路由器是非常关键的功能组件。

## Go Web 路由器 HttpRouter

与 Go 语言标准库 net/http 包中默认路由器相比，HttpRouter 路由器支持路由模式中的变量并与请求方法匹配，并可以更好地扩展。

HttpRouter 是使用 Go 语言开发的开源的轻量级高性能 HTTP 请求路由器（也称为多路复用器或简称为 mux）。

HttpRouter 路由器针对高性能和内存占用进行了优化。在同类开源产品中相对来说还是不错的，比如相对于标准库原生的 http.ServeMux
来说，HttpRouter 无论是在使用体验上，还是性能上都领先不少。当然这不是在贬低 http.ServeMux，它也很不错，而 Gin 框架选择
HttpRouter 路由器作为自己的多路复用器，足可以证明 HttpRouter 的优异，虽然 Gin 根据自己的情况做了较多的定制开发。

但无论怎样，Gin 框架的 HTTP 请求路由器是由 HttpRouter 来承担的，通过对 Gin 框架源代码的研读，可以看到在请求路由器这部分，Gin
框架中有较多定制开发的痕迹，但主要的实现方法和思路还是与 HttpRouter 开源版本契合度非常高。

从以上或许可以看出，Gin 对其他优秀的第三方包比较开放，至少目前来看选择了同类中比较优秀的两款产品作为自己的组件。

开发优秀的产品不一定非得完全靠自己造轮子，在众多轮子中找到合适的那个轮子也是一个不错的好方法。但是轮子该造的还是要造的，毕竟合适最重要。另外造的轮子多了，才会让我们有更多的选择。

下面来看看 Gin 官方对比同类产品做的性能测试数据报告。表 3-1 的基准测试数据表明 Gin 在性能指标上的领先水平。

Benchmark name | -1 | -2 | -3 | -4  
---|---|---|---|---  
BenchmarkGin_GithubAll | 30000 | 48375 | 0 | 0  
BenchmarkAce_GithubAll | 10000 | 134059 | 13792 | 167  
BenchmarkBear_GithubAll | 5000 | 534445 | 86448 | 943  
BenchmarkBeego_GithubAll | 3000 | 592444 | 74705 | 812  
BenchmarkBone_GithubAll | 200 | 6957308 | 698784 | 8453  
BenchmarkDenco_GithubAll | 10000 | 158819 | 20224 | 167  
BenchmarkEcho_GithubAll | 10000 | 154700 | 6496 | 203  
BenchmarkGocraftWeb_GithubAll | 3000 | 570806 | 131656 | 1686  
BenchmarkGoji_GithubAll | 2000 | 818034 | 56112 | 334  
BenchmarkGojiv2_GithubAll | 2000 | 1213973 | 274768 | 3712  
BenchmarkGoJsonRest_GithubAll | 2000 | 785796 | 134371 | 2737  
BenchmarkGoRestful_GithubAll | 300 | 5238188 | 689672 | 4519  
BenchmarkGorillaMux_GithubAll | 100 | 10257726 | 211840 | 2272  
BenchmarkHttpRouter_GithubAll | 20000 | 105414 | 13792 | 167  
BenchmarkHttpTreeMux_GithubAll | 10000 | 319934 | 65856 | 671  
BenchmarkKocha_GithubAll | 10000 | 209442 | 23304 | 843  
BenchmarkLARS_GithubAll | 20000 | 62565 | 0 | 0  
BenchmarkMacaron_GithubAll | 2000 | 1161270 | 204194 | 2000  
BenchmarkMartini_GithubAll | 200 | 9991713 | 226549 | 2325  
BenchmarkPat_GithubAll | 200 | 5590793 | 1499568 | 27435  
BenchmarkPossum_GithubAll | 10000 | 319768 | 84448 | 609  
BenchmarkR2router_GithubAll | 10000 | 305134 | 77328 | 979  
BenchmarkRivet_GithubAll | 10000 | 132134 | 16272 | 167  
BenchmarkTango_GithubAll | 3000 | 552754 | 63826 | 1618  
BenchmarkTigerTonic_GithubAll | 1000 | 1439483 | 239104 | 5374  
BenchmarkTraffic_GithubAll | 100 | 11383067 | 2659329 | 21848  
BenchmarkVulcan_GithubAll | 5000 | 394253 | 19894 | 609  
  
有关于上表的说明：

  1. 在一定的时间内实现的总调用数，越高越好

  2. 单次操作耗时（ns/op），越低越好

  3. 堆内存分配 （B/op）, 越低越好

  4. 每次操作的平均内存分配次数（allocs/op），越低越好

**本节总结**

  * HttpRouter 路由器介绍。

**本节问题**

  * 阅读源码，尝试解释标准库中 net/http 包中的 http.ServeMux 是什么？
  * ### 分享交流

我们为本专栏付费读者创建了微信交流群，以方便更有针对性地讨论专栏相关的问题。入群方式请添加 GitChat
小助手伽利略的微信号：GitChatty6（或扫描以下二维码），然后给小助手发「630」消息，即可拉你进群～

![R2Y8ju](https://images.gitbook.cn/R2Y8ju.jpg)

