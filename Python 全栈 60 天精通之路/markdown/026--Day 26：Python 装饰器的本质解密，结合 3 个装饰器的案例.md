### 装饰器应用广泛

Python 中使用 `@函数名字`，放在某些函数上面，起到增强它们功能的作用。

很多朋友觉得使用装饰器太魔幻，始终不知道怎么使用。装饰器带来的好处也显而易见，它会使得代码更加简洁，代码的复用性大大提升。

因此，装饰器被广泛使用。Python 中到处都能看到装饰器的身影。

不光在 Python，其他语言如 Java 中，装饰器被称作注解，也被应用广泛。

Python 前端最流行的框架之一 Flask，URL 地址和 controller 层的映射处理函数，就是使用装饰器。

`import Flask`，同时创建一个 app：

    
    
    from flask import Flask
    
    app = Flask(__name__)
    

URL 路径 `/` 与处理函数建立映射关系，正是通过装饰器 app.route 控制层处理响应前端，并返回字符串 hello world 到前端界面。

    
    
    @app.route('/')
    def index():
        return "hello world"
    

Python 支持异步编程，从中也能看到装饰器的身影。

下面例子，使用装饰器 @asyncio.coroutine，将一个生成器asyncio_open_conn 标记为 coroutine 类型。

    
    
    import asyncio
    
    @asyncio.coroutine
    def asyncio_open_conn(host:str):
        print("using asyncio builds web connection")
        connect = yield from asyncio.open_connection(host, 80)
        print("get connect of %s"%(host,))
    
    loop = asyncio.get_event_loop()
    connections = [asyncio_open_conn(host) for host in [
        'www.sina.com', 'www.baidu.com']]
    loop.run_until_complete(asyncio.wait(connections))
    loop.close()
    

异步请求建立 'www.sina.com'、'www.baidu.com' 两个连接，打印结果：

    
    
    using asyncio builds web connection
    using asyncio builds web connection
    get connect of www.baidu.com
    get connect of www.sina.com
    

装饰器还在日志管理、图像绘图等有重要的应用。因此，我们很有必要掌握装饰器，逐渐能做到灵活使用。

那么，今天我们就一步一步来理解装饰器的本质。我会尽量用最通俗的语言，通过代码和案例来辅助大家理解。

### call_print 装饰器

为了帮助大家更容易理解装饰器，以下函数包装，可能会让某些函数丢掉一些函数属性等信息。

记住，我们的主要目标：理解装饰器。

首先，定义函数 call_print，它的入参 f 为一个函数，它里面内嵌一个函数 g，并返回函数 g：

    
    
    def call_print(f):
        def g():
            print('you\'re calling %s function' % (f.__name__,))
        return g
    

Python 中，@call_print 函数，放在函数上面，函数 call_print 就变为装饰器。

变为装饰器后，我们不必自己去调用函数 call_print。

    
    
    @call_print
    def myfun():
        pass
    
    
    @call_print
    def myfun2():
        pass
    

直接调用被装饰的函数，就能调用到 call_print，观察输出结果：

    
    
    In [27]: myfun()
    you're calling myfun function
    
    In [28]: myfun2()
    you're calling myfun2 function
    

使用 call_print，@call_print 放置在任何一个新定义的函数上面。都会默认输出一行，输出信息：正在调用这个函数的名称。

有些朋友一定关心，这是怎么做到的。我们明明没有调用 call_print，但是却能输出实现它的功能？

下面，就来回答这个问题。

### 装饰器本质

call_print 装饰器实现效果，与下面调用方式，实现的效果是等效的。

    
    
    def myfun():
        pass
    
    
    def myfun2():
        pass
    
    
    def call_print(f):
        def g():
            print('you\'re calling %s function' % (f.__name__,))
        return g
    

下面两行代码，对于理解装饰器，非常代码：

    
    
    myfun = call_print(myfun)
    
    myfun2 = call_print(myfun2)
    

call_print(myfun) 后不是返回一个函数吗，然后，我们再赋值给被传入的函数 myfun。

也就是 myfun 指向了被包装后的函数 g，并移花接木到函数 g，使得 myfun 额外具备了函数 g 的一切功能，变得更强大。

以上就是装饰器的本质。

再次调用 myfun、myfun2 时，与使用装饰器的效果完全一致。

    
    
    In [32]: myfun()
    you're calling myfun function
    
    In [33]: myfun2()
    you're calling myfun2 function
    

### wraps 装饰器

在使用上面装饰器 call_print 后，

    
    
    def call_print(f):
        def g():
            print('you\'re calling %s function' % (f.__name__,))
        return g
    
    @call_print
    def myfun():
        pass
    
    @call_print
    def myfun2():
        pass
    

我们打印 myfun：

    
    
    myfun()
    
    myfun2()
    
    print(myfun)
    

打印结果：

    
    
    you're calling myfun function
    you're calling myfun2 function
    <function call_print.<locals>.g at 0x00000215D90679D8>
    

发现，被装饰的函数 myfun 名字竟然变为 g，也就是定义装饰器 call_print 时使用的内嵌函数 g。

Python 的 functools 模块，wraps 函数能解决这个问题。

解决方法，如下，只需重新定义 call_print，并在 g 内嵌函数上，使用 wraps 装饰器。

    
    
    from functools import wraps
    
    def call_print(f):
        @wraps(f)
        def g():
            print('you\'re calling %s function' % (f.__name__,))
        return g
    

当再次调用打印 myfun 时，函数 myfun 打印信息正常：

    
    
    print(myfun)
    # 结果
    <function myfun at 0x000002A34A8479D8>
    

### 案例 1：异常次数

这个装饰器，能统计出某个异常重复出现到指定次数时，历经的时长。

    
    
    import time
    import math
    
    def excepter(f):
        i = 0
        t1 = time.time()
        def wrapper():
            try:
                f()
            except Exception as e:
                nonlocal i
                i += 1
                print(f'{e.args[0]}: {i}')
                t2 = time.time()
                if i == n:
                    print(f'spending time:{round(t2-t1,2)}')
        return wrapper
    

关键词 nonlocal 在前面我们讲到了，今天就是它的一个应用。

它声明变量 i 为非局部变量；如果不声明，根据 Python 的 LEGB 变量搜寻规则（这个规则我们在前面的专栏中也讲到过），`i+=1` 表明 i
为函数 wrapper 内的局部变量，因为在 `i+=1` 引用（reference）时，i 未被声明，所以会报 `unreferenced
variable` 的错误。

使用创建的装饰函数 excepter，n 是异常出现的次数。

共测试了两类常见的异常：被零除和数组越界。

    
    
    n = 10# except count
    
    @excepter
    def divide_zero_except():
        time.sleep(0.1)
        j = 1/(40-20*2)
    
    # test zero divived except
    for _ in range(n):
        divide_zero_except()
    
    
    @excepter
    def outof_range_except():
        a = [1,3,5]
        time.sleep(0.1)
        print(a[3])
    
    # test out of range except
    for _ in range(n):
        outof_range_except()
    

打印出来的结果如下：

    
    
    division by zero: 1
    division by zero: 2
    division by zero: 3
    division by zero: 4
    division by zero: 5
    division by zero: 6
    division by zero: 7
    division by zero: 8
    division by zero: 9
    division by zero: 10
    spending time:1.01
    list index out of range: 1
    list index out of range: 2
    list index out of range: 3
    list index out of range: 4
    list index out of range: 5
    list index out of range: 6
    list index out of range: 7
    list index out of range: 8
    list index out of range: 9
    list index out of range: 10
    spending time:1.01
    

### 案例 2：绘图

导入本次实验所用的 4 种常见分布，连续分布的代表：beta 分布、正态分布、均匀分布，离散分布的代表：二项分布。

    
    
    import numpy as np
    from scipy.stats import beta, norm, uniform, binom
    import matplotlib.pyplot as plt
    from functools import wraps
    

#### **定义带参数的装饰器**

绘图装饰器带有四个参数分别表示，legend 的两类说明文字，y 轴 label，保存的 PNG 文件名称。

注意使用 wraps，包装了内嵌函数 myplot。这样做的好处，被包装的函数名字不会被改变。

    
    
    # 定义带四个参数的画图装饰器
    def my_plot(label0=None, label1=None, ylabel='probability density function', fn=None):
        def decorate(f):
            @wraps(f)
            def myplot():
                fig = plt.figure(figsize=(16, 9))
                ax = fig.add_subplot(111)
                x, y, y1 = f()
                ax.plot(x, y, linewidth=2, c='r', label=label0)
                ax.plot(x, y1, linewidth=2, c='b', label=label1)
                ax.legend()
                plt.ylabel(ylabel)
                # plt.show()
                plt.savefig('./img/%s' % (fn,))
                plt.close()
            return myplot
        return decorate
    

#### **均匀分布**

从图中可看出，红色概率密度函数只在 0~1 才会发生，曲线与 x 轴的 0~1 区间所封闭的面积为全概率 1.0。

    
    
    @my_plot(label0='b-a=1.0', label1='b-a=2.0', fn='uniform.png')
    def unif():
        x = np.arange(-0.01, 2.01, 0.01)
        y = uniform.pdf(x, loc=0.0, scale=1.0)
        y1 = uniform.pdf(x, loc=0.0, scale=2.0)
        return x, y, y1
    

![](https://images.gitbook.cn/7d0df630-5c1f-11ea-93b5-3993d9eab61e)

#### **二项分布**

红色曲线表示发生一次概率为 0.3，重复 50 次的密度函数，二项分布期望值为 0.3*50=15 次。看到这 50 次实验，很可能出现的次数为
10~20。可与蓝色曲线对比分析。

    
    
    @my_plot(label0='n=50,p=0.3', label1='n=50,p=0.7', fn='binom.png', ylabel='probability mass function')
    def bino():
        x = np.arange(50)
        n, p, p1 = 50, 0.3, 0.7
        y = binom.pmf(x, n=n, p=p)
        y1 = binom.pmf(x, n=n, p=p1)
        return x, y, y1
    

![](https://images.gitbook.cn/aa990cc0-5c1f-11ea-aae6-17c0629b6dc0)

#### **高斯分布**

红色曲线表示均值为 0，标准差为 1.0 的概率密度函数，蓝色曲线的标准差更大，所以它更矮胖，显示出取值的多样性，和不稳定性。

    
    
    # 高斯分布
    @my_plot(label0='u=0.,sigma=1.0', label1='u=0.,sigma=2.0', fn='guass.png')
    def guass():
        x = np.arange(-5, 5, 0.1)
        y = norm.pdf(x, loc=0.0, scale=1.0)
        y1 = norm.pdf(x, loc=0., scale=2.0)
        return x, y, y1
    

![](https://images.gitbook.cn/c9c405f0-5c1f-11ea-a695-8f4c079b036d)

#### **beta 分布**

beta 分布的期望值如下，可从下面的两条曲线中加以验证：

![](https://images.gitbook.cn/e70f69b0-5c1f-11ea-8155-9d6d04886d5b)

    
    
    @my_plot(label0='a=10., b=30.', label1='a=4., b=4.', fn='beta.png')
    def bet():
        x = np.arange(-0.1, 1, 0.001)
        y = beta.pdf(x, a=10., b=30.)
        y1 = beta.pdf(x, a=4., b=4.)
        return x, y, y1
    

![](https://images.gitbook.cn/12d59ec0-5c20-11ea-aae6-17c0629b6dc0)

统一调用以上四个函数，分别绘制概率曲线：

    
    
    distrs = [unif, bino, guass, bet]
    for distri in distrs:
        distri()
    

### 小结

今天与大家一起讨论了应用广泛的装饰器：

  * 使用自定义的 call_print 装饰器
  * 讨论装饰器的本质：myfun 指向被包装后的函数 g，并移花接木到函数 g
  * 使用 wraps 装饰器，解决被装饰函数属性信息被篡改的问题
  * 两个实际应用装饰器的案例

