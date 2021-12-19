岛国，在我朝的东边，我朝臣民对它的心态可以说比较复杂了。就软件领域而言，它的存在感貌似不大，其实不然。就拿国内的一些大厂，有不少其背后会有一家岛国公司身影——软银。岛国技术人员为开源领域的贡献，也同样精彩，比如很有名的
[Ruby](https://en.wikipedia.org/wiki/Ruby_\(programming_language\)) 语言，发明者为
[Yukihiro
Matsumoto](https://en.wikipedia.org/wiki/Yukihiro_Matsumoto)，是一名岛国程序员。本课要介绍的
Bokeh 也是岛国程序员发明的，专用于数据可视化。

维基百科的词条 [Bokeh](https://en.wikipedia.org/wiki/Bokeh)
介绍了这个词的读音和含义，也可以不用理会什么含义，就当做一个名称罢了。

### 6.1.1 精彩一瞥

Bokeh 跟前面已经介绍过的 Plotly 和 pyecharts 类似，都是生成基于网页的可视化图示，并且也具有交互性。

起手要做的肯定是安装了。

    
    
    pip3 install bokeh
    

这是最简单的安装方式，如果不能安装，可以直接从 [GitHub](https://github.com/bokeh/bokeh) 上下载。

安装好之后，可以运行 jupyter notebook，之后执行下面的程序，通过示例体验一下 Bokeh 的应用。

    
    
    import numpy as np
    from bokeh.plotting import figure, output_file, show    # ①
    
    x = np.linspace(0, 2*np.pi, 50)
    y = np.sin(x)
    
    fig = figure(plot_width = 800, plot_height = 600, 
                 title = 'sin line', 
                 y_axis_label = 'y', x_axis_label = 'x')    # ②
    fig.line(x, y, line_width = 2, legend = 'sin')    # ③
    
    output_file('./sin.html')    # ④
    show(fig)    # ⑤
    

执行这段程序之后，会自动打开一个网页，并且网页文件的名称为“sin.html”。

![](https://images.gitbook.cn/9cc0afe0-4a55-11e9-8e81-8d1f6de9c7cd)

上述结果与前面介绍过的 Plotly 和 pyecharts 一样。当然，Bokeh 也有自己的特点，后续就能了解到，这里先简要地理解一下上面的程序。

  * ① 引入程序中需要的模块。
  * ② 中使用 figure 创建一个图示实例对象。figure 其实是 Bokeh 中的类 Figure 的简单封装，用于初始化图示对象，默认情况下会包含坐标系的坐标轴、坐标网格，以及图示右侧的有关工具等。
  * ③ 调用了 fig 对象的一个方法 fig.line，意即绘制曲线。
  * ④ 使用 output_file 设置输出文件名称，如果不写此句，Bokeh 会默认一个名称。
  * ⑤ 的效果则是要显示所绘制的图示对象。

Plotly 和 pyecharts 都能将 HTML 中的图嵌入到当前正在运行的 Jupyter 浏览器中， **Bokeh 有这个功能吗？**

这个可以有！

    
    
    from bokeh.io import output_notebook
    output_notebook()
    

引入 output_notebook 之后，执行 output_notebook()，用它替代 ④。执行之后，结果如下图所示。

![](https://images.gitbook.cn/5c10a410-4a59-11e9-a807-d3eeb3ad6d3c)

此后在执行第一段程序，则代码块之下就能插入图示对象。

除了能够绘制一条曲线之外，在同一个坐标系中绘制多条曲线也是很容易的事情。

    
    
    from bokeh.plotting import figure, show
    
    x = [0.1, 0.5, 1.0, 1.5, 2.0, 2.5, 3.0]
    y0 = [i**2 for i in x]
    y1 = [10**i for i in x]
    y2 = [10**(i**2) for i in x]
    
    p = figure(tools="pan,box_zoom,reset,save",    # ⑥
               y_axis_type="log", 
               y_range=[0.001, 10**11],
               title="多函数曲线",
               x_axis_label='sections', 
               y_axis_label='particles',)
    
    p.line(x, x, legend="y=x")    # ⑦
    p.circle(x, x, legend="y=x", fill_color="white", size=8)    # ⑧
    p.line(x, y0, legend="y=x^2", line_width=3)
    p.line(x, y1, legend="y=10^x", line_color="red")
    p.square(x, y1, legend="y=10^x", fill_color="red", line_color="blue", size=10)    # ⑨
    p.line(x, y2, legend="y=10^x^2", line_color="orange", line_dash="4 4")
    
    show(p)
    

![](https://images.gitbook.cn/bab60080-4a5b-11e9-8e81-8d1f6de9c7cd)

相对前面的示例，此示例做了更多的自定义设置。在 ⑥ 创建图示实例的时候，使用的参数比较多。

  * tools="pan,box_zoom,reset,save"：前面示例的图示右侧，默认把所有可以出现的工具都显示了出来，而这里仅仅显示指定的工具。
  * y_axis_type：设置 Y 轴的数据类型，同样的还有参数 x_axis_type。默认值是 'auto'，此外还可以为 'linear'、'datetime'、'log'、'mercatro'。

⑦ 绘制 y = x 的曲线，这跟前面示例一样；⑧ 则是根据 x 的值以及函数 y = x 得到的 y
值，绘制相应的坐标“点”，只是这里的“点”的形状是“圆形”——用 p.circle 实现。

  * size：每个“点”的大小（直径）。
  * fill_color：每个“点”的填充色。

与 ⑧ 具有同样功能的是 ⑨，区别在于 ⑨ 把“点”绘制成了方形—— p.square，其中的参数，可依据“望文生义”原则理解。

总览上述制图过程，如果在同一个图示对象中绘制多种图像，只需要使用该对象多次调用相应的制图方法即可。

### 6.1.2 基本要素

通过上面的两个示例，已经对 Bokeh 绘图有了初步感受，正如第 6 部分标题中所言“岛国薄纱”，对于学习者而言，因为有了前面使用各种工具的基础，再学习
Bokeh 就不困难了，它只不过是一层“薄纱”罢了，通过简单示例就已经能看到某些了——如果要深入，还需要继续研习，必须动手“拨云”，才能最终“见日”。

通常而言，Bokeh 制图的基本要素或者基本步骤如下。

（1）图示对象

前面的示例中已经显示，都是要先创建图示对象。bokeh.plotting 中有创建图示图像的类 Figure，并且还提供了相应的简化函数
figure，创建图示对象常用参数如下。

  * **tools** ：字符串。用于设置图示对象中提供的工具（称为 “图示工具”）。默认值为 'DEFAULT_TOOLS'，前面示例中演示了设置为其他值的情况。此外，还可以通过 toolbar_location 参数设置图示工具的位置（'above'、'below'、'left'、'right'）。更多关于图示工具的配置说明，请参阅：[Configuring Plot Tools](https://bokeh.pydata.org/en/latest/docs/user_guide/tools.html)。
  * **x_range、y_range** ：分别设置 X 轴和 Y 轴的数值范围。
  * **x_minor_ticks、y_minor_ticks** ：默认值 'auto'，设置各 X 轴或者 Y 轴主刻线之间的副刻线数量。
  * **x_axis_location** ：设置 X 轴的位置，默认值 'below'，即 X 轴在图示的下方。
  * **y_minor_ticks** ：设置 Y 轴的位置，默认值 'left'。
  * **x_axis_label、y_axis_label** ：设置对 X 轴和 Y 轴的描述。
  * **x_axis_type、y_axis_type** ：设置 X 轴或者 Y 轴的数据类型，前面示例中已经解释。
  * 针对交互操作中鼠标动作的参数设置，主要有 **active_drag、active_inspect、active_scroll、active_tap** ，它们的默认值都是 'auto'，可以根据需要，对这些值进行修改（通常修改较少），具体可以参阅[官方文档中有关说明](https://bokeh.pydata.org/en/latest/docs/user_guide/tools.html)。

（2）图线模型

Bokeh 提供了多种制图模型—— Bokeh 中将这些称为 glyphs，并且，在 Figure
类的实例对象——即“图示对象”的方法中，提供了相应的接口，以在图示对象中实现各种图线模型。对各种方法的说明，请参阅[官方文档](https://bokeh.pydata.org/en/latest/docs/reference/plotting.html)。

### 6.1.3 小结

本课对 Bokeh 制图做了简要介绍，了解了其制图的基本要素。在此基础上，如果凭借已经掌握的有关知识和官方文档，读者已经能够使用 Bokeh
完整数据可视化基本工作了。

### 答疑与交流

> **为了方便与作者交流与学习，GitChat 编辑团队组织了一个《Python数据可视化》读者交流群，添加小助手-
> 伽利略微信：「GitChatty6」，回复关键字「288」给小助手伽利略获取入群资格。**

