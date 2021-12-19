### pyecharts 快速入门

时常翻看 pyecharts 的源码，感叹框架写的真棒，思路清晰、设计简洁、通俗易懂，推荐读者们有空也阅读下。被 pyecharts 官档介绍 5
个特性所吸引：

  * 简洁的 API 设计，使用如丝滑般流畅，支持链式调用
  * 囊括了 30+ 种常见图表，应有尽有
  * 支持主流 Notebook 环境，Jupyter Notebook 和 JupyterLab
  * 可轻松集成至 Flask、Django 等主流 Web 框架
  * 高度灵活的配置项，可轻松搭配出精美的图表

pyecharts 确实也如上面五个特性介绍那样，使用起来非常方便。那么，有些读者不禁好奇会问，pyecharts 是如何做到的？

我们不妨从 pyecharts 官档 5 分钟入门 pyecharts 章节开始，由表（最高层函数）及里（底层函数也就是所谓的源码），一探究竟。

#### **第 1 例**

不妨从官档给出的第一个例子说起：

    
    
    from pyecharts.charts import Bar
    from pyecharts.globals import ThemeType    
    
    from pyecharts.faker import Faker    
    from pyecharts.commons.utils import JsCode
    
    bar = Bar()
    bar.add_xaxis(["衬衫", "羊毛衫", "雪纺衫", "裤子", "高跟鞋", "袜子"])
    bar.add_yaxis("商家A", [5, 20, 36, 10, 75, 90])
    # render 会生成本地 HTML 文件，默认会在当前目录生成 render.html 文件
    # 也可以传入路径参数，如 bar.render("mycharts.html")
    bar.render()
    

第一行代码：`from pyecharts.charts import Bar`，先上一张源码中“包的结构图”：

![](https://images.gitbook.cn/7a59d700-7cbe-11ea-bee1-ab7e0d8cd403)

bar.py 模块中定义了类 Bar(RectChart)，如下所示：

    
    
    class Bar(RectChart):
        """
        <<< Bar Chart >>>
    
        Bar chart presents categorical data with rectangular bars
        with heights or lengths proportional to the values that they represent.
        """
    

这里有读者可能会有以下 2 个问题。

**1\. 为什么根据图 1 中的包结构，为什么不这么写：**

    
    
    from pyecharts.charts.basic_charts import Bar
    

![](https://images.gitbook.cn/8a8ae3d0-7cbe-11ea-9dcd-17c924164c9c)

答：请看图 2 中 __init__.py 模块，文件内容如下，看到导入 charts 包，而非 charts.basic_charts：

    
    
    from pyecharts import charts, commons, components, datasets, options, render, scaffold
    from pyecharts._version import __author__, __version__
    

**2\. Bar(RectChart) 是什么意思？**

答：RectChart 是 Bar 的子类。

下面 4 行代码，很好理解，没有特殊性。

pyecharts 主要两个大版本，0.5 基版本和 1.0 基版本，从 1.0 基版本开始全面支持链式调用，链式调用模式代码看起来更加紧凑。

    
    
    from pyecharts.charts import Bar
    
    bar = (
        Bar()
        .add_xaxis(["衬衫", "羊毛衫", "雪纺衫", "裤子", "高跟鞋", "袜子"])
        .add_yaxis("商家A", [5, 20, 36, 10, 75, 90])
    )
    bar.render()
    

实现链式调用也没有多难，保证返回类本身 self 即可，如果非要有其他返回对象，那么要提到类内以便被全局共享，add_xaxis 函数返回 self：

    
    
        def add_xaxis(self, xaxis_data: Sequence):
            self.options["xAxis"][0].update(data=xaxis_data)
            self._xaxis_data = xaxis_data
            return self
    

add_yaxis 函数同样返回 self。

#### **皆 options**

pyecharts 用起来很爽的另一个重要原因，参数配置项封装的非常优秀，通过定义一些列基础的配置组件，比如 global_options.py
模块中定义的配置对象有以下 27 个：

    
    
        AngleAxisItem,
        AngleAxisOpts,
        AnimationOpts,
        Axis3DOpts,
        AxisLineOpts,
        AxisOpts,
        AxisPointerOpts,
        AxisTickOpts,
        BrushOpts,
        CalendarOpts,
        DataZoomOpts,
        Grid3DOpts,
        GridOpts,
        InitOpts,
        LegendOpts,
        ParallelAxisOpts,
        ParallelOpts,
        PolarOpts,
        RadarIndicatorItem,
        RadiusAxisItem,
        RadiusAxisOpts,
        SingleAxisOpts,
        TitleOpts,
        ToolBoxFeatureOpts,
        ToolboxOpts,
        TooltipOpts,
        VisualMapOpts,
    

了解上面的配置对象后，再看官档给出的第二个例子，与第一个例子相比，增加了一行代码：set_global_opts 函数。

    
    
    from pyecharts.charts import Bar
    from pyecharts import options as opts
    
    # V1 版本开始支持链式调用
    # 你所看到的格式其实是 `black` 格式化以后的效果
    # 可以执行 `pip install black` 下载使用
    bar = (
        Bar()
        .add_xaxis(["衬衫", "羊毛衫", "雪纺衫", "裤子", "高跟鞋", "袜子"])
        .add_yaxis("商家A", [5, 20, 36, 10, 75, 90])
        .set_global_opts(title_opts=opts.TitleOpts(title="主标题", subtitle="副标题"))
    
    bar.render()
    

set_global_opts 函数在 pyecharts 中被高频使用，它定义在底层基础模块 Chart.py 中，它是前面说到的 RectChart
的子类、Bar 类的孙子类。

浏览下函数的参数：

    
    
    def set_global_opts(
            self,
            title_opts: types.Title = opts.TitleOpts(),
            legend_opts: types.Legend = opts.LegendOpts(),
            tooltip_opts: types.Tooltip = None,
            toolbox_opts: types.Toolbox = None,
            brush_opts: types.Brush = None,
            xaxis_opts: types.Axis = None,
            yaxis_opts: types.Axis = None,
            visualmap_opts: types.VisualMap = None,
            datazoom_opts: types.DataZoom = None,
            graphic_opts: types.Graphic = None,
            axispointer_opts: types.AxisPointer = None,
        ):
    

以第二个参数 title_opts 为例，说明 pyecharts 中参数赋值的风格。

首先，title_opts 是默认参数，默认值为 opts.TitleOpts()，这个对象在上一节我们提到过，global_options.py
模块中定义的 27 个配置对象之一。

其次，pyecharts 中为了增强代码可读性，参数的类型都显示的给出。此处它的类型为：types.Title。这是什么类型？它的类型不是
TitleOpts 吗？

看看 Title 这个类型的定义：

    
    
    Title = Union[opts.TitleOpts, dict]
    

原来 Title 可能是 opts.TitleOpts, 也可能是 Python 原生的 dict。通过 Union
实现的就是这种类型效果。所以这就清楚官档中为什么说，也可以使用字典配置参数的问题。

    
    
      # 或者直接使用字典参数
        # .set_global_opts(title_opts={"text": "主标题", "subtext": "副标题"})
    )
    

最后，真正的关于图表的标题相关的属性都被封装到 TitleOpts 类中，比如 title、subtitle 属性，查看源码，TitleOpts
对象还有更多属性：

    
    
    class TitleOpts(BasicOpts):
        def __init__(
            self,
            title: Optional[str] = None,
            title_link: Optional[str] = None,
            title_target: Optional[str] = None,
            subtitle: Optional[str] = None,
            subtitle_link: Optional[str] = None,
            subtitle_target: Optional[str] = None,
            pos_left: Optional[str] = None,
            pos_right: Optional[str] = None,
            pos_top: Optional[str] = None,
            pos_bottom: Optional[str] = None,
            padding: Union[Sequence, Numeric] = 5,
            item_gap: Numeric = 10,
            title_textstyle_opts: Union[TextStyleOpts, dict, None] = None,
            subtitle_textstyle_opts: Union[TextStyleOpts, dict, None] = None,
        ):
    

结合 2 个例子实现的背后源码，探讨了：

  * 与包结构组织相关的 __init__.py
  * 类的继承关系：Bar->RectChart->Chart
  * 链式调用
  * 重要的参数配置包 options，以 TitleOpts 类为例，set_global_opts 将它装载到 Bar 类中实现属性自定义。

### 属性设置之道

经常有读者朋友问 pyecharts 中属性设置的问题，比如 y 轴如何显示在右侧。授人以鱼不如授人以渔，今天就以如何使 y 轴显示在右侧为例，介绍
pyecharts 的属性设置之道。

#### **y 轴靠右**

那么，如何设置 y 轴显示在右侧，添加一行代码：

    
    
    .set_global_opts(yaxis_opts=opts.AxisOpts(position='right'))
    

也就是：

    
    
    c = (
            Bar()
            .add_xaxis(Faker.choose())
            .add_yaxis("商家A", Faker.values())
            .set_global_opts(title_opts=opts.TitleOpts(title="Bar-基本示例", subtitle="我是副标题"))
            .set_global_opts(yaxis_opts=opts.AxisOpts(position='right'))
        )
    

如何锁定这个属性，首先应该在 set_global_opts 函数的参数中找，它一共有以下 11 个设置参数，它们位于模块 charts.py:

    
    
    title_opts: types.Title = opts.TitleOpts(),
    legend_opts: types.Legend = opts.LegendOpts(),
    tooltip_opts: types.Tooltip = None,
    toolbox_opts: types.Toolbox = None,
    brush_opts: types.Brush = None,
    xaxis_opts: types.Axis = None,
    yaxis_opts: types.Axis = None,
    visualmap_opts: types.VisualMap = None,
    datazoom_opts: types.DataZoom = None,
    graphic_opts: types.Graphic = None,
    axispointer_opts: types.AxisPointer = None,
    

因为是设置 y 轴显示在右侧，自然想到设置参数 yaxis_opts，因为其类型为 types.Axis，所以再进入 types.py，同时定位到
Axis：

    
    
    Axis = Union[opts.AxisOpts, dict, None]
    

查看第一个 opts.AxisOpt 类，它共定义以下 25 个参数：

    
    
    type_: Optional[str] = None,
    name: Optional[str] = None,
    is_show: bool = True,
    is_scale: bool = False,
    is_inverse: bool = False,
    name_location: str = "end",
    name_gap: Numeric = 15,
    name_rotate: Optional[Numeric] = None,
    interval: Optional[Numeric] = None,
    grid_index: Numeric = 0,
    position: Optional[str] = None,
    offset: Numeric = 0,
    split_number: Numeric = 5,
    boundary_gap: Union[str, bool, None] = None,
    min_: Union[Numeric, str, None] = None,
    max_: Union[Numeric, str, None] = None,
    min_interval: Numeric = 0,
    max_interval: Optional[Numeric] = None,
    axisline_opts: Union[AxisLineOpts, dict, None] = None,
    axistick_opts: Union[AxisTickOpts, dict, None] = None,
    axislabel_opts: Union[LabelOpts, dict, None] = None,
    axispointer_opts: Union[AxisPointerOpts, dict, None] = None,
    name_textstyle_opts: Union[TextStyleOpts, dict, None] = None,
    splitarea_opts: Union[SplitAreaOpts, dict, None] = None,
    splitline_opts: Union[SplitLineOpts, dict] = SplitLineOpts(),
    

观察后尝试参数 position，结合官档：

> <https://pyecharts.org/#/zh-
> cn/global_options?id=axisopts%ef%bc%9a%e5%9d%90%e6%a0%87%e8%bd%b4%e9%85%8d%e7%bd%ae%e9%a1%b9>

介绍 x 轴设置 position 时有 bottom、top，所以 y 轴设置很可能就是 left、right。

![](https://images.gitbook.cn/ab7d2f80-7cbe-11ea-9792-81939fbf7f0c)

#### **14 步案例**

1\. 柱状图显示效果动画对应控制代码：

    
    
    animation_opts=opts.AnimationOpts(
                        animation_delay=500, animation_easing="cubicOut"
                    )
    

2\. 柱状图显示主题对应控制代码：

    
    
    theme=ThemeType.MACARONS
    

3\. 添加 x 轴对应的控制代码：

    
    
    add_xaxis( ["草莓", "芒果", "葡萄", "雪梨", "西瓜", "柠檬", "车厘子"]
    

4\. 添加 y 轴对应的控制代码：

    
    
    add_yaxis("A", Faker.values(),
    

5\. 修改柱间距对应的控制代码：

    
    
    category_gap="50%"
    

6\. A 系列柱子是否显示对应的控制代码：

    
    
    is_selected=True
    

7\. A 系列柱子颜色渐变对应的控制代码：

    
    
    itemstyle_opts={
                "normal": {
                    "color": JsCode("""new echarts.graphic.LinearGradient(0, 0, 0, 1, [{
                        offset: 0,
                        color: 'rgba(0, 244, 255, 1)'
                    }, {
                        offset: 1,
                        color: 'rgba(0, 77, 167, 1)'
                    }], false)"""),
                    "barBorderRadius": [6, 6, 6, 6],
                    "shadowColor": 'rgb(0, 160, 221)',
                }}
    

8\. A 系列柱子最大和最小值标记点对应的控制代码：

    
    
    markpoint_opts=opts.MarkPointOpts(
                    data=[
                        opts.MarkPointItem(type_="max", name="最大值"),
                        opts.MarkPointItem(type_="min", name="最小值"),
                    ]
                )
    

9\. A 系列柱子最大和最小值标记线对应的控制代码：

    
    
    markline_opts=opts.MarkLineOpts(
                    data=[
                        opts.MarkLineItem(type_="min", name="最小值"),
                        opts.MarkLineItem(type_="max", name="最大值")
                    ]
                )
    

10\. 柱状图标题对应的控制代码：

    
    
    title_opts=opts.TitleOpts(title="Bar-参数使用例子"
    

11\. 柱状图非常有用的 toolbox 显示对应的控制代码：

    
    
    toolbox_opts=opts.ToolboxOpts()
    

12\. Y 轴显示在右侧对应的控制代码：

    
    
    yaxis_opts=opts.AxisOpts(position="right")
    

13\. Y 轴名称对应的控制代码：

    
    
    yaxis_opts=opts.AxisOpts(,name="Y轴")
    

14\. 数据轴区域放大缩小设置对应的控制代码：

    
    
    datazoom_opts=opts.DataZoomOpts()
    

完整代码：

    
    
    def bar_border_radius():
        c = (
            Bar(init_opts=opts.InitOpts(
                    animation_opts=opts.AnimationOpts(
                        animation_delay=500, animation_easing="cubicOut"
                    ),
                    theme=ThemeType.MACARONS))
            .add_xaxis( ["草莓", "芒果", "葡萄", "雪梨", "西瓜", "柠檬", "车厘子"])
            .add_yaxis("A", Faker.values(),category_gap="50%",markpoint_opts=opts.MarkPointOpts(),is_selected=True)
            .set_series_opts(itemstyle_opts={
                "normal": {
                    "color": JsCode("""new echarts.graphic.LinearGradient(0, 0, 0, 1, [{
                        offset: 0,
                        color: 'rgba(0, 244, 255, 1)'
                    }, {
                        offset: 1,
                        color: 'rgba(0, 77, 167, 1)'
                    }], false)"""),
                    "barBorderRadius": [6, 6, 6, 6],
                    "shadowColor": 'rgb(0, 160, 221)',
                }}, markpoint_opts=opts.MarkPointOpts(
                    data=[
                        opts.MarkPointItem(type_="max", name="最大值"),
                        opts.MarkPointItem(type_="min", name="最小值"),
                    ]
                ),markline_opts=opts.MarkLineOpts(
                    data=[
                        opts.MarkLineItem(type_="min", name="最小值"),
                        opts.MarkLineItem(type_="max", name="最大值")
                    ]
                ))
            .set_global_opts(title_opts=opts.TitleOpts(title="Bar-参数使用例子"), toolbox_opts=opts.ToolboxOpts(),yaxis_opts=opts.AxisOpts(position="right",name="Y轴"),datazoom_opts=opts.DataZoomOpts(),)
    
        )
    
        return c
    
    bar_border_radius().render()
    

### 10 类图

接下来介绍使用 pyecharts 绘制出的 10 类图，重点解释数据是如何呈现在不同类型图中。

使用 `pip install pyecharts` 安装，安装后的版本为 v1.6。

pyecharts 几行代码就能绘制出有特色的的图形，绘图 API 链式调用，使用方便。

#### **仪表盘**

    
    
    from pyecharts import charts
    
    # 仪表盘
    gauge = charts.Gauge()
    gauge.add('Python全栈60天', [('Python机器学习', 30), ('Python基础', 70.),
                            ('Python正则', 90)])
    gauge.render(path="./img/仪表盘.html")
    print('ok')
    

仪表盘中共展示三项，每项的比例为 30%、70%、90%，如下图默认名称显示第一项：Python 机器学习，完成比例为 30%

![](https://images.gitbook.cn/d6c13c40-7cbe-11ea-9687-397b36d0cdab)

#### **漏斗图**

    
    
    from pyecharts import options as opts
    from pyecharts.charts import Funnel, Page
    from random import randint
    
    def funnel_base() -> Funnel:
        c = (
            Funnel()
            .add("豪车", [list(z) for z in zip(['宝马', '法拉利', '奔驰', '奥迪', '大众', '丰田', '特斯拉'],
                                             [randint(1, 20) for _ in range(7)])])
            .set_global_opts(title_opts=opts.TitleOpts(title="豪车漏斗图"))
        )
        return c
    
    funnel_base().render('./img/car_funnel.html')
    print('ok')
    

以 7 种车型及某个属性值绘制的漏斗图，属性值大越靠近漏斗的大端。

![](https://images.gitbook.cn/e68a5f80-7cbe-11ea-9687-397b36d0cdab)

#### **日历图**

    
    
    import datetime
    import random
    
    from pyecharts import options as opts
    from pyecharts.charts import Calendar
    
    
    def calendar_interval_1() -> Calendar:
        begin = datetime.date(2019, 1, 1)
        end = datetime.date(2019, 12, 27)
        data = [
            [str(begin + datetime.timedelta(days=i)), random.randint(1000, 25000)]
            for i in range(0, (end - begin).days + 1, 2)  # 隔天统计
        ]
    
        calendar = (
            Calendar(init_opts=opts.InitOpts(width="1200px")).add(
                "", data, calendar_opts=opts.CalendarOpts(range_="2019"))
            .set_global_opts(
                title_opts=opts.TitleOpts(title="Calendar-2019年步数统计"),
                visualmap_opts=opts.VisualMapOpts(
                    max_=25000,
                    min_=1000,
                    orient="horizontal",
                    is_piecewise=True,
                    pos_top="230px",
                    pos_left="100px",
                ),
            )
        )
        return calendar
    
    
    calendar_interval_1().render('./img/calendar.html')
    print('ok')
    

绘制 2019 年 1 月 1 日到 12 月 27 日的步行数，官方给出的图形宽度 900px 不够，只能显示到 9 月份，本例使用
`opts.InitOpts(width="1200px")` 做出微调，并且 visualmap 显示所有步数，每隔一天显示一次：

![](https://images.gitbook.cn/f3c134a0-7bf9-11ea-8a35-fda221135a5a)

#### **图（graph）**

    
    
    import json
    import os
    
    from pyecharts import options as opts
    from pyecharts.charts import Graph, Page
    
    
    def graph_base() -> Graph:
        nodes = [
            {"name": "cus1", "symbolSize": 10},
            {"name": "cus2", "symbolSize": 30},
            {"name": "cus3", "symbolSize": 20}
        ]
        links = []
        for i in nodes:
            if i.get('name') == 'cus1':
                continue
            for j in nodes:
                if j.get('name') == 'cus1':
                    continue
                links.append({"source": i.get("name"), "target": j.get("name")})
        c = (
            Graph()
            .add("", nodes, links, repulsion=8000)
            .set_global_opts(title_opts=opts.TitleOpts(title="customer-influence"))
        )
        return c
    

构建图，其中客户点 1 与其他两个客户都没有关系（link），也就是不存在有效边：

![](https://images.gitbook.cn/e3e80bd0-7bf9-11ea-b348-9df247d9e896)

#### **水球图**

    
    
    from pyecharts import options as opts
    from pyecharts.charts import Liquid, Page
    from pyecharts.globals import SymbolType
    
    
    def liquid() -> Liquid:
        c = (
            Liquid()
            .add("lq", [0.67, 0.30, 0.15])
            .set_global_opts(title_opts=opts.TitleOpts(title="Liquid"))
        )
        return c
    
    
    liquid().render('./img/liquid.html')
    

水球图的取值 [0.67, 0.30, 0.15] 表示下图中的“三个波浪线”，一般代表三个百分比：

![](https://images.gitbook.cn/cc036870-7bf9-11ea-a711-0f902cb8434c)

#### **饼图**

    
    
    from pyecharts import options as opts
    from pyecharts.charts import Pie
    from random import randint
    
    def pie_base() -> Pie:
        c = (
            Pie()
            .add("", [list(z) for z in zip(['宝马', '法拉利', '奔驰', '奥迪', '大众', '丰田', '特斯拉'],
                                           [randint(1, 20) for _ in range(7)])])
            .set_global_opts(title_opts=opts.TitleOpts(title="Pie-基本示例"))
            .set_series_opts(label_opts=opts.LabelOpts(formatter="{b}: {c}"))
        )
        return c
    
    pie_base().render('./img/pie_pyecharts.html')
    

![](https://images.gitbook.cn/b9bff660-7bf9-11ea-99ef-3bc3936801ae)

#### **极坐标**

    
    
    import random
    from pyecharts import options as opts
    from pyecharts.charts import Page, Polar
    
    def polar_scatter0() -> Polar:
        data = [(alpha, random.randint(1, 100)) for alpha in range(101)] # r = random.randint(1, 100)
        print(data)
        c = (
            Polar()
            .add("", data, type_="bar", label_opts=opts.LabelOpts(is_show=False))
            .set_global_opts(title_opts=opts.TitleOpts(title="Polar"))
        )
        return c
    
    
    polar_scatter0().render('./img/polar.html')
    

极坐标表示为 (夹角,半径)，如 (6,94) 表示“夹角”为 6、半径 94 的点：

![](https://images.gitbook.cn/a4bdd980-7bf9-11ea-8a35-fda221135a5a)

#### **词云图**

    
    
    from pyecharts import options as opts
    from pyecharts.charts import Page, WordCloud
    from pyecharts.globals import SymbolType
    
    
    words = [
        ("Python", 100),
        ("C++", 80),
        ("Java", 95),
        ("R", 50),
        ("JavaScript", 79),
        ("C", 65)
    ]
    
    
    def wordcloud() -> WordCloud:
        c = (
            WordCloud()
            # word_size_range: 单词字体大小范围
            .add("", words, word_size_range=[20, 100], shape='cardioid')
            .set_global_opts(title_opts=opts.TitleOpts(title="WordCloud"))
        )
        return c
    
    
    wordcloud().render('./img/wordcloud.html')
    

`("C",65)` 表示在本次统计中 C 语言出现 65 次。

![](https://images.gitbook.cn/68f5e500-7bf9-11ea-bb65-a9795596171f)

#### **系列柱状图**

    
    
    from pyecharts import options as opts
    from pyecharts.charts import Bar
    from random import randint
    
    
    def bar_series() -> Bar:
        c = (
            Bar()
            .add_xaxis(['宝马', '法拉利', '奔驰', '奥迪', '大众', '丰田', '特斯拉'])
            .add_yaxis("销量", [randint(1, 20) for _ in range(7)])
            .add_yaxis("产量", [randint(1, 20) for _ in range(7)])
            .set_global_opts(title_opts=opts.TitleOpts(title="Bar的主标题", subtitle="Bar的副标题"))
        )
        return c
    
    
    bar_series().render('./img/bar_series.html')
    

![](https://images.gitbook.cn/586ad150-7bf9-11ea-bb65-a9795596171f)

#### **热力图**

    
    
    import random
    from pyecharts import options as opts
    from pyecharts.charts import HeatMap
    
    
    def heatmap_car() -> HeatMap:
        x = ['宝马', '法拉利', '奔驰', '奥迪', '大众', '丰田', '特斯拉']
        y = ['中国','日本','南非','澳大利亚','阿根廷','阿尔及利亚','法国','意大利','加拿大']
        value = [[i, j, random.randint(0, 100)]
                 for i in range(len(x)) for j in range(len(y))]
        c = (
            HeatMap()
            .add_xaxis(x)
            .add_yaxis("销量", y, value)
            .set_global_opts(
                title_opts=opts.TitleOpts(title="HeatMap"),
                visualmap_opts=opts.VisualMapOpts(),
            )
        )
        return c
    
    heatmap_car().render('./img/heatmap_pyecharts.html')
    

![](https://images.gitbook.cn/4963c590-7bf9-11ea-9687-397b36d0cdab)

### 小结

今天与大家一起学习绘图神器 pyecharts：

  * 快速上手
  * 参数快速配置的技巧
  * 绘制常用的 10 类图

