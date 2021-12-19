Matplotlib 是基于 Python 的开源项目，旨在为 Python 提供一个数据绘图包。

> Matplotlib 的对象体系严谨而有趣，为使用者提供了巨大的发挥空间。在熟悉核心对象之后，可以轻易的定制图像。

接下来，介绍 Matplotlib API 的核心对象，并介绍如何使用这些对象来实现绘图和相关案例。

### 绘图必备

先看一段代码：

    
    
    from matplotlib.figure import Figure
    from matplotlib.backends.backend_agg import FigureCanvasAgg as FigureCanvas
    
    fig = Figure()
    canvas = FigureCanvas(fig)
    ax = fig.add_axes([0.1, 0.1, 0.8, 0.8])
    line,  = ax.plot([0,1], [0,1])
    ax.set_title("a straight line ")
    ax.set_xlabel("x label")
    ax.set_ylabel("y label")
    canvas.print_figure('chatpic1.jpg')
    

上面这段代码，至少构建了四个对象：fig（Figure 类）、canvas（FigureCanvas 类）、ax（Axes 类）、line（Line2D
类）。

在 Matplotlib 中，整个图像为一个 Figure 对象，在 Figure 对象中可以包含一个或多个 Axes 对象：

  * Axes 对象 axes1 都是一个拥有自己坐标系统的绘图区域
  * Axes 由 xAxis、yAxis、title、data 构成 
    * xAxis 由 XTick、Ticker 以及 label 构成
    * yAxis 由 YTick、Ticker 以及 label 构成
  * Axes 对象 axes2 也是一个拥有自己坐标系统的绘图区域
  * Axes 由 xAxis、yAxis、title、data 构成 
    * xAxis 由 XTick、Ticker 以及 label 构成
    * yAxis 由 YTick、Ticker 以及 label 构成

如下图所示：

![](https://images.gitbook.cn/84a51300-7237-11ea-97af-c3b20af12573)

canvas 对象，代表真正进行绘图的后端(backend)

`ax = fig.add_axes([0.1, 0.1, 0.8, 0.8])`，分别表示：图形区域的左边界距离 figure 左侧 10%，底部
10%，宽度和高度都为整个 figure 宽度和高度的 80%。

在具备这些绘图的基本理论知识后，再去使用 Matplotlib 库就会顺手很多。

### 绘图分解

下面介绍使用 Matplotlib 绘图时，常用的功能，按照绘图元素将之分解。

#### **导入**

    
    
    import matplotlib
    import matplotlib.pyplot as plt
    import numpy as np   
    

#### **数据**

    
    
    x = np.linspace(0, 5, 10) 
    y = x ** 2  
    

#### **折线图**

    
    
    plt.plot(x, y) 
    plt.show() 
    

![](https://images.gitbook.cn/a9f05890-7237-11ea-97f8-8df0bef76179)

#### **线条颜色**

    
    
    plt.plot(x, y, 'r') 
    plt.show() 
    

![](https://images.gitbook.cn/c2f40080-7237-11ea-af73-378397689c85)

#### **线型**

    
    
    plt.plot(x, y, 'r--') 
    plt.show() 
    

![](https://images.gitbook.cn/dd357fa0-7237-11ea-b39b-4d2cde13353b)

    
    
    plt.plot(x, y, 'g-*') 
    plt.show() 
    

![](https://images.gitbook.cn/edead7a0-7237-11ea-b39b-4d2cde13353b)

#### **标题**

    
    
    plt.plot(x, y, 'r-*') 
    plt.title('title')  
    plt.show() 
    

![](https://images.gitbook.cn/14800250-7238-11ea-b8cb-b108c7f0b146)

#### **x、y 轴 label**

    
    
    plt.plot(x, y, 'r-*') 
    plt.title('title') 
    plt.xlabel('x') 
    plt.ylabel('y')
    plt.show() 
    

![](https://images.gitbook.cn/26378810-7238-11ea-ab1a-832c54b4f266)

#### **文本**

    
    
    plt.plot(x, y, 'r--') 
    plt.text(1.5,10,'y=x*x')
    

![](https://images.gitbook.cn/369d1260-7238-11ea-ab1a-832c54b4f266)

#### **注解**

    
    
    plt.plot(x, y, 'r') 
    plt.annotate('this is annotate',xy=(3.5,12),xytext=(2,16),arrowprops={'headwidth':10,'facecolor':'r'})
    

![](https://images.gitbook.cn/461f4190-7238-11ea-b39b-4d2cde13353b)

#### **显示中文**

    
    
    # 显示中文
    from pylab import mpl
    mpl.rcParams['font.sans-serif'] = ['FangSong']
    mpl.rcParams['axes.unicode_minus'] = False # 解决保存图像是负号'-'显示为方块的问题
    
    plt.plot(x, y, 'r') 
    plt.title('显示中文标题')
    plt.show()
    

![](https://images.gitbook.cn/5be60e00-7238-11ea-ab1a-832c54b4f266)

#### **双 data**

    
    
    plt.plot(x, y, 'r--') 
    plt.plot(x,x**2+6,'g')
    plt.show() 
    

![](https://images.gitbook.cn/6a6c29f0-7238-11ea-97af-c3b20af12573)

#### **图例**

    
    
    plt.plot(x, y, 'r--') 
    plt.plot(x,x**2+6,'g')
    plt.legend(['y=x^2','y=x^2+6'])
    plt.show() 
    

![](https://images.gitbook.cn/7ac30e40-7238-11ea-83c3-675e02f948ee)

#### **网格**

    
    
    plt.plot(x, y, 'r--') 
    plt.plot(x,x**2+6,'g')
    plt.grid(linestyle='--',linewidth=1)
    plt.show() 
    

![](https://images.gitbook.cn/8abf0150-7238-11ea-83c3-675e02f948ee)

#### **范围**

    
    
    plt.plot(x,x**3,color='g')
    plt.scatter(x, x**3,color='r') 
    plt.xlim(left=1,right=3)
    plt.ylim(bottom=1,top = 30)
    plt.show()
    

![](https://images.gitbook.cn/a020ca60-7238-11ea-83c3-675e02f948ee)

#### **格式**

    
    
    x=pd.date_range('2020/01/01',periods=30)
    y=np.arange(0,30,1)**2 
    plt.plot(x,y,'r')
    plt.gcf().autofmt_xdate()
    plt.show()
    

![](https://images.gitbook.cn/b02c26c0-7238-11ea-bf44-ef79e0369ea9)

#### **双轴**

    
    
    x = np.linspace(1, 5, 10) 
    y = x ** 2  
    
    plt.plot(x,y,'r')
    plt.text(3,10,'y=x^2')
    plt.twinx()
    plt.plot(x,np.log(x),'g')
    plt.text(1.5,0.4,'y=logx')
    plt.show()
    

![](https://images.gitbook.cn/c189f3c0-7238-11ea-8377-13f07d2f46fb)

#### **双图**

    
    
    plt.subplot(1,2,1) 
    plt.plot(x, y, 'r--') 
    plt.subplot(1,2,2) 
    plt.plot(y, x, 'g*-') 
    plt.show()  
    

![](https://images.gitbook.cn/d180b6b0-7238-11ea-97f8-8df0bef76179)

#### **嵌入图**

    
    
    fig = plt.figure()
    
    axes1 = fig.add_axes([0.1, 0.1, 0.8, 0.8]) # main axes
    axes2 = fig.add_axes([0.2, 0.5, 0.4, 0.3]) # insert axes
    
    # 主图
    axes1.plot(x, y, 'r')
    axes1.set_xlabel('x')
    axes1.set_ylabel('y')
    axes1.set_title('title')
    
    # 插入的图
    axes2.plot(y, x, 'g')
    axes2.set_xlabel('y')
    axes2.set_ylabel('x')
    axes2.set_title('insert title')
    

![](https://images.gitbook.cn/9e8dee90-0526-11eb-b8a7-43f9b203a4ce)

#### **Matplotlib 绘制动画**

animation 模块能绘制动画。

    
    
    from matplotlib import pyplot as plt
    from matplotlib import animation
    from random import randint, random
    

生成数据，frames_count 是帧的个数，data_count 每个帧的柱子个数

    
    
    class Data:
        data_count = 32
        frames_count = 2
    
        def __init__(self, value):
            self.value = value
            self.color = (0.5, random(), random()) #rgb
    
        # 造数据
        @classmethod
        def create(cls):
            return [[Data(randint(1, cls.data_count)) for _ in range(cls.data_count)]
                    for frame_i in range(cls.frames_count)]
    

绘制动画：animation.FuncAnimation 函数的回调函数的参数 fi 表示第几帧，注意要调用 axs.cla() 清除上一帧。

    
    
    def draw_chart():
        fig = plt.figure(1, figsize=(16, 9))
        axs = fig.add_subplot(111)
        axs.set_xticks([])
        axs.set_yticks([])
    
        # 生成数据
        frames = Data.create()
    
        def animate(fi):
            axs.cla()  # clear last frame
            axs.set_xticks([])
            axs.set_yticks([])
            return axs.bar(list(range(Data.data_count)),        # X
                           [d.value for d in frames[fi]],       # Y
                           1,                                   # width
                           color=[d.color for d in frames[fi]]  # color
                           )
        # 动画展示
        anim = animation.FuncAnimation(fig, animate, frames=len(frames))
        plt.show()
    
    
    draw_chart()
    

### 小结

今天总结了 Matplotlib：

  * 绘图基本原理
  * 绘图常用的 18 种技巧

