Timelion 是细粒度的一种时间序列可视化方式，使用 Timelion 就相当于使用编程来控制时间序列可视化。
既然涉及到编程那就肯定就会有变量，我们就可以在时间序列数据可视化之前做运算处理。比如说，金钱单位换算，或者计算一阶或者二阶差分等。这么细粒度的控制时间序列可视化方法，很有必要进行学习掌握。

### Timelinon 创建

实践是最好的学习方式，因此抛开一堆的参数，先使用 Timelion 绘制一个简单图开始。

**给定需求：** 使用 Timelion 绘制折线图。

1\. 选择新建可视化，选择 Timelion：

![image-20191223202908176](https://images.gitbook.cn/2020-04-07-063039.png)

2\. 下图是 Timelion 的初始化界面，与之前的相比明显简洁很多，能够控制的只有时间间隔与 Timelion 表达式。

![image-20191223202959785](https://images.gitbook.cn/2020-04-07-063041.png)

3\. 方便的是 Timelion 表达式在书写过程中，都会有提示，因此写起来也方便。以下单日期为 order_date 为 x 轴，下单总额的平均值为 y
轴，时间间隔暂时设置为自动，日期范围设置长一点，我这里设置的是 60 天之前。折线图效果如下。

Timelion 表达式如下：

    
    
    *.es(index=kibana_sample_data_ecommerce, timefield='order_date', metric=avg:taxful_total_price)*
    

理解起来也比较简单，函数是 `*.es(\*)*`，一共设置了三个参数，index 索引、timefield 时间字段、metric 指标。

指定 index 为 kibana_sample_data_ecommerce 索引，X 轴设置为 order_date，Y 轴设置为
taxful_total_price 的平均值。

![image-20191223210358939](https://images.gitbook.cn/2020-04-07-063042.png)

### 差分算法平滑曲线

我们经常会碰到这种需求，折线的不规律波动太大，能不能减小这种波动， **差分的作用是减轻数据之间的不规律波动，使其波动曲线更平稳**
。这种需求常见于机器学习中的回归分析，比如说时间序列的的预测，像 ARMA 算法，全称是 **自回归滑动平均模型**
，是一种预测时间序列数据的模型，这个模型其中就有一步叫做一阶差分，简单的说就是前面的数据减去后面的数据，使得曲线变得平稳波动。

**给定需求：** 使用差分算法，减轻不规律波动。

差分就是自己减自己，如下图，自己减去自己后结果为 0，但是我们不能用相同时刻的自己减去自己，要错开去减去，比如说当前时间的自己，减去过去 1 个小时的自己。

下面这个表达式，我只是用了 substract 函数，然后把自己作为参数放进去，结果都是 0，想要减去 1 个小时的自己，这里需要设置一个
offset，作为一个偏置的时间区间。

    
    
    *.es(index=kibana_sample_data_ecommerce, timefield='order_date', metric=avg:taxful_total_price).subtract(.es(index=kibana_sample_data_ecommerce, timefield='order_date', metric=avg:taxful_total_price))*
    

![image-20191223215536560](https://images.gitbook.cn/2020-04-07-063043.png)

红线是我偏移 10 个小时的数据，黄线是原始数据，绿线是差分后的数据，很明显，差分后数据都围绕着 0
波动。这就是一阶差分效果图，如果想要更平稳的波动可以再次差分，叫做二阶差分。这里不是解析什么是差分，而是想要表明，Timelion
有这种方法，可以两个时间序列加减乘除。很有趣不是吗？

表达式如下：

    
    
    *.es(index=kibana_sample_data_ecommerce, timefield='order_date', metric=avg:taxful_total_price).subtract(.es(offset=-10h,index=kibana_sample_data_ecommerce, timefield='order_date', metric=avg:taxful_total_price)),.es(index=kibana_sample_data_ecommerce, timefield='order_date', metric=avg:taxful_total_price),.es(offset=-50h,index=kibana_sample_data_ecommerce, timefield='order_date', metric=avg:taxful_total_price)*
    

![image-20191223222457732](https://images.gitbook.cn/2020-04-07-063044.png)

曲线波动过于大，有没有一种方式能够平缓曲线，方便我们去预测曲线的走势呢？

有这里介绍一个简单的平缓曲线的方法， **指数加权移动平均** ，公式如下：

$$v_t=βv_{t−1}+(1−β)θ_t$$

这是个很有用的工具，我希望你在学习使用 Kibana 分析数据之前，也能够了解一些数据分析的原理。

t 指的是时间，t-1 指的是上一刻的时间，$\beta$ 是自定义的权重，$\theta$ 是当前的时刻的值，是原始值，未经处理的值，$V$
是累计的结果。

假设初始时间的值是 $\theta_0$，那么依次的计算过程如下：

$v_0=0$

$v_1=\beta v_{0}+(1-\beta)\theta_0$

$v_2=\beta v_1+(1-\beta)\theta_1$

$......$

$v_t=βv_{t−1}+(1−β)θ_t$

理解这个公式有助于你理解移动平均原理，Timeline 中提供了 mvavg
移动平均的函数，并内置了一些参数，逻辑和指数加权移动平均差不多。黑色的线代表平滑的效果。

![image-20191223231937771](https://images.gitbook.cn/2020-04-07-063045.png)

表达式如下：

    
    
    .es(index=kibana_sample_data_ecommerce,timefield='order_date',metric=avg:taxful_total_price).subtract(es(offset=-10h,index=kibana_sample_data_ecommerce,timefield='order_date',metric=avg:taxful_total_price)),
    
    .es(index=kibana_sample_data_ecommerce,timefield='order_date',metric=avg:taxful_total_price),.es(offset=-10h,index=kibana_sample_data_ecommerce, timefield='order_date', metric=avg:taxful_total_price),
    
    .es(index=kibana_sample_data_ecommerce, timefield='order_date', metric=avg:taxful_total_price).mvavg(10, right)
    

异常值是值得注意的一个指标，但是异常并不仅仅是值的异常，也有可能是增长的异常，比如说程序占用内存突然急剧增高？会不会是内存泄漏呢？如何对这个现象做一个监控呢？答案是可以的。

**给定需求：监控增长异常。**

导数是反映数据变化快慢的一个指标，我们可以求导，得出曲线，值越大那么增长的越快，反之越快减小的越快。

导数的每个峰值点，都对应原始数据变化较快的点，导数的每个为 0
的值，都对应原始数据的峰值，即极大值或者极小值点，鞍点出现的几率还是比较小的。因此可以通过导数的这个特点，来监控更多的指标健康状况。

![image-20191226205837104](https://images.gitbook.cn/2020-04-07-063047.png)

### Timeline 计算导数

    
    
    *.es(index=kibana_sample_data_ecommerce, timefield='order_date', metric=avg:taxful_total_price).derivative()*
    

Timeline 不仅仅可以使用折线图显示，也可以使用柱状图显示：

    
    
    *.es(index=kibana_sample_data_ecommerce, timefield='order_date', metric=avg:taxful_total_price).bars()*
    

![image-20191226211729489](https://images.gitbook.cn/2020-04-07-063049.png)

对于折线图的导数图也是折线图，这点看起来可能会比较怪，因为直观的从折线图来看，很多地方线是直的那么导数应该也是同一个值，所以导数图应该是阶梯状的图，而不是斜率不为
0 的折线图。实际上在 timeline 中，只是计算了折线图中点的导数，因此导数图用柱状图可能看起来会比较直观，因为不管怎说数据是离散的，并不是连续的。

![image-20191226212302382](https://images.gitbook.cn/2020-04-07-063050.png)

### 小结

上节中介绍了针对不同的需求提出了不同的可视化方法，像热力图以及时间序列可视化方法，本节最大的亮点还是在于 Timeline
时间序列可视化，灵活多变，满足更有针对性的需求。

学完本节，相信你已经能够应对不同的需求检测可视化。本章也是数据分析中重要的一节，不仅仅在介绍可视化方法的同时，穿插数据分析原理，像移动平均，导数。让你不仅仅会可视化，还懂得了背后的原理，只有懂得了原理，才能够分析得更有针对性，对不同的指标特征有更深刻的理解。

