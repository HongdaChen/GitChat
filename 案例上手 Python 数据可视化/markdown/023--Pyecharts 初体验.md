从本课开始，我们终于能够阅读用中文写的文档了。因为 pyecharts 是国产的，它来自于百度的 ECharts 项目。

> ECharts，一个使用 JavaScript 实现的开源可视化库，可以流畅地运行在 PC
> 和移动设备上，兼容当前绝大部分浏览器（IE8/9/10/11、Chrome、Firefox、Safari 等），底层依赖轻量级的矢量图形库
> ZRender，提供直观、交互丰富、可高度个性化定制的数据可视化图表。

ECharts 官方网站：<https://echarts.baidu.com/>，这个网站界面很酷，值得欣赏。

pyecharts 是 ECharts 的 Python 版。因此，我们可以用它绘制酷炫的各种图表。

pyecharts 官方网站：<http://pyecharts.org/>。

因为 pyecharts 的文档是中文的，这给学习者带来了很大的便利。

### 5.1.1 安装 pyecharts

以下安装过程是作者的经验之谈，供读者参考。

（1）安装 Node.js，请根据本地计算机的情况至[官网下载相应安装程序](https://nodejs.org/en/download/)。

（2）安装 PhantomJS，可以到[官网下载安装包](http://phantomjs.org/download.html)，或者用下列命令：

    
    
    $ sudo npm install -g phantomjs-prebuilt --upgrade --unsafe-perm
    

（3）安装 pyecharts：

    
    
    $ pip install pyecharts
    

（4）为了保存 PNG、PDF、GIF 格式文件，需要安装 pyecharts-snapshot：

    
    
    $ pip install pyecharts-snapshot
    

（5）为了显示，需要安装主题插件：

    
    
    $ pip install echarts-themes-pypkg
    

### 5.1.2 基本绘制步骤

这里以一个示例来说明用 pyecharts
绘制图示的基本过程。这个示例来自官方文档，但是解释方式稍有不同。之所以用这个示例，就是要时刻提示，pyecharts
的文档是中文的，而且写得非常明了，直接阅读即可知晓如何绘图。因此，本达人课会很快地完成基本知识的介绍。

    
    
    from pyecharts import Bar                               # ①
    
    clothes = ["衬衫", "羊毛衫", "雪纺衫", "裤子", "高跟鞋", "袜子"]
    clothes_v1 = [5, 20, 36, 10, 75, 90]
    clothes_v2 = [10, 25, 8, 60, 20, 80]
    
    bar = Bar("柱状图数据堆叠示例")                            # ②
    bar.use_theme('dark')                                   # ③
    bar.add("Shop-A", clothes, clothes_v1, is_stack=True)   # ④
    bar.add("Shop-B", clothes, clothes_v1, is_stack=True)   
    bar                                                     # ⑤
    

![](https://images.gitbook.cn/25c5c380-495b-11e9-9675-055a4c4b91fa)

通过本示例，可以简要归纳制图流程。

  * ① 引入相应类型的图表对象类。这里使用了 Bar，它是一个类，如果要实现具体的图示，必须要用这个类创建实例，这就是 ②。
  * ② 创建了 Bar 的实例。如果在 Jupyter 中进行下图所示操作，即查看官方文档，会看到中文帮助文档。

![](https://images.gitbook.cn/4d686690-495b-11e9-9675-055a4c4b91fa)

因此，只要勤看文档，必然知道某个对象的使用方法，② 中为参数 title 提供了值。如果喜欢尝试，可以根据文档，增加并尝试其他的参数。

  * 已经得到一个图示的实例之后，关于图示的其他所有内容，都是这个实例的属性，或者通过实例的方法实现某些操作，即方法。如果按照下图所示，输入 bar. + TAB，就可以看到这个对象的所有属性和方法。

![enter image description
here](https://images.gitbook.cn/1cbffbb0-4961-11e9-9f9a-07ff224f37b2)

  * ③ 是使用其中一个方法，为 bar 对象设置了主题的颜色。
  * ④ 也是使用 bar 的方法，增加数据，以及相关属性设置。
  * ⑤ 是在当前页面调用此对象，即把图示对象插入到当前位置。
  * 如果 ⑤ 用 bar.render() 替代，会在当前工作目录生成文件名为 render.html 的网页，图示结果会在其中。如果使用 bar.render(path='path/name.html') 格式，可以自定义文件保存路径和文件名称。
  * 如果要生成 PNG 图片，可以使用 bar.render(path="./temp.png") 实现，用这方式还可以生成 “SVG/PDF/GIF” 等格式文件。

根据上述分解说明，读者不妨回头再审视一番代码，理解制图的基本流程——面向对象思想依然是这里的主流。

### 5.1.3 基本图表举例

在[官方文档](http://pyecharts.org/#/zh-cn/charts_base)中列出了 pyecharts
能够绘制的基本图表，并且包括各项细节，建议参考。此处仅以几个示例进行展示。

**1\. 滑块效果**

具有滑块效果的图，在 Plotly 中曾经有过，这里使用 pyecharts
实现，看看效果是否有差别（下述示例数据来源[请点击这里查看](https://github.com/qiwsir/DataSet/tree/master/appl%EF%BC%89%E3%80%82)）。

    
    
    import pandas as pd
    appl_df = pd.read_csv("/Users/qiwsir/Documents/Codes/DataSet/appl/appl.csv", 
                          index_col=['date'], parse_dates=['date'])
    df20 = appl_df.iloc[:20, :]
    
    bar = Bar('苹果公司近日股票收盘价')
    bar.add("", df20.index, df20['open'], is_label_show=True, datazoom_type='both', is_datazoom_show=True)
    bar
    

输出结果：

![](https://images.gitbook.cn/ca083930-4962-11e9-9675-055a4c4b91fa)

如果在 Jupyter 中调试上述程序，不仅可以移动滑块观察动态交互效果，还可以用鼠标在柱形图中左右拖动（datazoom_type='both'
的效果）。与 Plotly 的类似效果相比，pyecharts 更丰富多样——官方网站中还有更丰富的示例，可以参考。

滑块效果，不仅可以在柱形图中实现，在其他类型的图像中也可以实现，比如股票中的蜡烛图（或曰 K 线图）。

    
    
    from pyecharts import Kline
    kline = Kline("K线图")
    kline.add("APPL", df20.index, df20[['open','close','high','low']].values,
              mark_line = ['max'],
              datazoom_orient="vertical",
              is_datazoom_show=True,
             )
    kline
    

输出结果：

![enter image description
here](https://images.gitbook.cn/169fc470-4963-11e9-a9ef-73aa0bf0b745)

这里实现的滑块效果与以往不同之处，在于 Y 轴方向滑动（datazoom_orient="vertical" 控制效果），并且在图示中增加了一条横线
mark_line = ['max']。

**2\. 新类型**

除了常规的统计图之外，pyecharts 还提供了一些新类型的图示，如果用其他工具实现这些图示，可能都要费一番周折。但是在 pyecharts
中简单了很多。

    
    
    from pyecharts import Gauge
    gauge = Gauge("仪表盘示例")
    gauge.add(
        "业务指标",
        "完成率",
        166.66,
        angle_range=[180, 0],
        scale_range=[0, 200],
        is_legend_show=False,
    )
    gauge
    

![](https://images.gitbook.cn/447c5660-4963-11e9-99d4-dfadbb5b8dbf)

这是来自官方网站提供的 **仪表盘** 示例，使用 Gauge 类实现。

还有 **水球图** ，操作也非常简单。

    
    
    from pyecharts import Liquid
    
    liquid = Liquid("水球图示例")
    liquid.add("Liquid", [0.6])
    liquid
    

![](https://images.gitbook.cn/58d7fa10-4963-11e9-99d4-dfadbb5b8dbf)

如果在 Jupyter 中看图中的效果，水波具有动态效果。

此外还有 **雷达图** 、 **桑基图** 等，此处就不赘述，因为有完整的文档，其中对每种类型的图示都有详细的示例。

### 5.1.4 小结

本课介绍了一款国产可视化工具
pyecharts，它的优点很多，不过我认为其最大优点就是文档写的好：不仅是中文文档，而且有完整的示例。当然，得到的图示比以往工具更精美，也是必须要提的一个优点。

### 答疑与交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《Python数据可视化》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「288」给小助手伽利略获取入群资格。**

