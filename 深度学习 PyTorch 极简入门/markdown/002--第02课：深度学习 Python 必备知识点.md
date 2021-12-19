无论是在机器学习还是深度学习中，Python 已经成为主导性的编程语言。而且，现在许多主流的深度学习框架，例如 PyTorch、TensorFlow
也都是基于 Python。这门课主要是围绕“理论 + 实战”同时进行的，所以本课程我将重点介绍深度学习中 Python 的必备知识点。

### 2.1 为什么选择 Python

![enter image description
here](http://images.gitbook.cn/cca85310-7adb-11e8-be93-2154b2b481b1)

Python 是一种面向对象的解释型计算机程序设计语言，由荷兰人 Guido van Rossum 于 1989 年发明，第一个公开发行版发行于 1991
年。Python 具有丰富和强大的库，它常被昵称为胶水语言，能够把用其他语言制作的各种模块（尤其是 C/C++）很轻松地联结在一起。

为什么人工智能、深度学习会选择 Python 呢？一方面是因为 Python 作为一门解释型语言，入门简单、容易上手。另一方面是因为 Python
的开发效率高，Python 有很多库很方便做人工智能，比如 Numpy、Scipy 做数值计算，Sklearn 做机器学习，Matplotlib
将数据可视化，等等。总的来说，Python 既容易上手，又是功能强大的编程语言。按照《Python 学习手册》作者的说法，Python
可以支持从航空航天器系统开发到小游戏开发的几乎所有的领域。

其实，人工智能的核心算法的底层还是由 C/C++ 编写的，因为是计算密集型，需要非常精细的优化，还需要 GPU、专用硬件之类的接口，这些都只有 C/C++
能做到。Python 实际上是实现 API 调用的功能，例如所有的深度学习框架 PyTorch、TensorFlow 等，底层都是由 C/C++
编写的。由于 Python 是顶层高级语言，它的缺点就是运行速度慢，但是这丝毫不影响 Python 的普及。如今，在 GPU 加速的前提下，Python
的运行速度已经很快了。在众多因素影响下，Python 毫无疑问成为了人工智能的最主要的编程语言。

下面这张图来自 TIOBE 编程社区 Top 10 编程语言 TIOBE 指数走势（2002年—2018年）：

![enter image description
here](http://images.gitbook.cn/3b0e0e40-7ae0-11e8-8a2d-af6f3f96c3a3)

如今，Python 已经仅次于 Java、C、C++ 之后排名第四，且呈逐年上升的趋势。而在人工智能领域，Python 是当之无愧的第一。

Python 目前有两个版本：2 和 3。人工智能领域主要使用 Python 3，建议安装 Python 3
版本。你也可以使用如下命令来查看当前安装的是哪个版本：

    
    
    python --version
    

### 2.2 函数 Function 与类 Class

Python 中的函数以关键字 def 来定义，例如：

    
    
    def sign(x):
        if x > 0:
            return 'positive'
        elif x < 0:
            return 'negative'
        else:
            return 'zero'
    
    for x in [-1, 0, 1]:
        print(sign(x))
    # Prints "negative", "zero", "positive"
    

上面呢，就是定义一个 sign 函数，根据输入 x 与 0 的大小关系，返回 positive、negative 或 zero。

函数的形参也可以设置成默认值，例如：

    
    
    def greet(name, loud=False):
        if loud:
            print('HELLO, %s!' % name.upper())
        else:
            print('Hello, %s' % name)
    
    greet('Will') # Prints "Hello, Will"
    greet('Tony', loud=True)  # Prints "HELLO, TONY!"
    

Python 中的类的概念和其他语言相比没什么不同，例如：

    
    
    class Greeter(object):
    
        # Constructor
        def __init__(self, name):
            self.name = name  # Create an instance variable
    
        # Instance method
        def greet(self, loud=False):
            if loud:
                print('HELLO, %s!' % self.name.upper())
            else:
                print('Hello, %s' % self.name)
    
    g = Greeter('Will')  # Construct an instance of the Greeter class
    g.greet()            # Call an instance method; prints "Hello, Will"
    g.greet(loud=True)   # Call an instance method; prints "HELLO, WILL!"
    

__init__函数是类的初始化函数，所有成员变量都是 self 的，所以初始化函数一般都包含 self 参数。name 是类中函数将要调用的输入参数。

Python 中类的继承也非常简单，最基本的继承方式就是定义类的时候把父类往括号里一放就行了：

    
    
    class Know(Greeter):
        """Class Know inheritenced from Greeter"""
        def meet(self):
            print('Nice to meet you!')
    k = Know('Will')  # Construct an instance of the Greater class
    k.greet()         # Call an instance method; prints "Hello, Will"
    k.meet()          # Call an instance method; prints "Nice to meet you!"
    

### 2.3 向量化和矩阵

深度学习神经网络模型包含了大量的矩阵相乘运算，如果使用 for 循环，运算速度会大大降低。Python 中可以使用 dot
函数进行向量化矩阵运算，来提高网络运算效率。我们用一个例子来比较说明 for 循环和矩阵运算各自的时间差异性。

    
    
    import numpy as np
    import time
    
    a = np.random.rand(100000)
    b = np.random.rand(100000)
    
    tic = time.time()
    for i in range(100000):
        c += a[i]*b[i]
    toc = time.time()
    
    print(c)
    print("for loop:" + str(1000*(toc-tic)) + "ms")
    
    c = 0
    tic = time.time()
    c = np.dot(a,b)
    toc = time.time()
    
    print(c)
    print("Vectorized:" + str(1000*(toc-tic)) + "ms")
    

输出结果为：

    
    
    >> 274877.869751
    >> for loop:99.99990463256836ms
    >> 24986.1673877
    >> Vectorized:0.9999275207519531ms
    

显然，两个矩阵相乘，使用 for 循环需要大约 100 ms ，而使用向量化矩阵运算仅仅需要大约 1 ms
，效率得到了极大的提升。值得一提的是，神经网络模型有的矩阵维度非常大，这时候，使用矩阵直接相乘会更大程度地提高速度。所以，在构建神经网络模型时，我们应该尽量使用矩阵相乘运算，减少
for 循环的使用。

顺便提一下，为了加快深度学习神经网络运算速度，可以使用比 CPU 运算能力更强大的 GPU。事实上，GPU 和 CPU
都有并行指令（Parallelization Instructions），称为 Single Instruction Multiple
Data（SIMD）。SIMD 是单指令多数据流，能够复制多个操作数，并把它们打包在大型寄存器的一组指令集。SIMD
能够大大提高程序运行速度，并行运算也就是向量化矩阵运算更快的原因。相比而言，GPU 的 SIMD 要比 CPU 更强大。

### 2.4 广播 Broadcasting

Python 中的广播（Broadcasting）机制非常强大，在神经网络模型矩阵运算中非常有用。广播机制主要包括以下几个部分：

  * 让所有输入数组都向其中 shape 最长的数组看齐，shape 中不足的部分都通过在前面加1补齐；
  * 输出数组的 shape 是输入数组 shape 的各个轴上的最大值；
  * 如果输入数组的某个轴和输出数组的对应轴的长度相同或者其长度为 1 时，这个数组能够用来计算，否则出错；
  * 当输入数组的某个轴的长度为 1 时，沿着此轴运算时都用此轴上的第一组值。

如果觉得上面几条机制比较晦涩难懂，没关系。简而言之，就是 Python
中可以对不同维度的矩阵进行四则混合运算，但至少保证有一个维度是相同的。下面我举几个简单的例子，你就明白了。

![enter image description
here](http://images.gitbook.cn/0b22ce10-7b84-11e8-95d0-05d53c09e644)

是不是觉得广播机制很方便？这也正是 Python 强大的地方，能够帮我们省很多事。

值得一提的是，在 Python 程序中为了保证矩阵运算正确，可以使用 reshape 函数设定矩阵为所需的维度。这是一个很好且有用的习惯。例如：

    
    
    x.reshape(2,3)
    

关于矩阵维度，还有一些需要注意的地方。例如，我们定义一个向量，可能会这样写：

    
    
    a = np.random.randn(6)
    

上面这条语句生成的向量维度既不是（6，1），也不是（1，6），而是（6，）。它既不是列向量也不是行向量，而是 rank 1 array。rank 1
array 的特点是它的转置还是它本身。这种定义实际应用中可能会带来一些问题，如果我们想要定义行向量或者列向量的话，最好这样写：

    
    
    a = np.random.randn(1,6)
    a = np.random.randn(6,1)
    

另外，我们还可以使用 assert 语句对向量或者数组维度进行判断。如果与给定的维度不同，则程序在此处停止运行。assert
的灵活使用可以帮助我们及时检查神经网络模型中参数的维度是否正确。

    
    
    assert(a == shape(6,1))
    

### 2.5 Matplotlib 绘图

Matplotlib 是 Python 一个强大的绘图库，下面我将简单介绍一下 matplotlib.pyplot 模块。

plot 是 Matplotlib 主要的 2D 绘图函数，举个简单的例子：

    
    
    import numpy as np
    import matplotlib.pyplot as plt
    
    # Compute the x and y coordinates
    x = np.arange(0, 4 * np.pi, 0.1)
    y = np.sin(x)
    
    # Plot the points using matplotlib
    plt.plot(x, y)
    plt.show()  # You must call plt.show() to make graphics appear.
    

![enter image description
here](http://images.gitbook.cn/07d636f0-7b8b-11e8-adef-b5d6c5f29056)

我们也可以在一张图片中同时画多个曲线：

    
    
    import numpy as np
    import matplotlib.pyplot as plt
    
    # Compute the x and y coordinates
    x = np.arange(0, 4 * np.pi, 0.1)
    y_sin = np.sin(x)
    y_cos = np.cos(x)
    
    # Plot the points using matplotlib
    plt.plot(x, y_sin)
    plt.plot(x, y_cos)
    plt.xlabel('x axis label')
    plt.ylabel('y axis label')
    plt.title('Sine and Cosine')
    plt.legend(['Sine', 'Cosine'])
    plt.show()
    

![enter image description
here](http://images.gitbook.cn/92a8d990-7b8b-11e8-b36a-d9daff3efc07)

最后介绍一下图片如何显示：

    
    
    import numpy as np
    from scipy.misc import imread, imresize
    import matplotlib.pyplot as plt
    
    img = imread('./dog.jpg')
    img_tinted = img * [0.9, 0.9, 0.8]
    
    # Show the original image
    plt.subplot(2, 1, 1)
    plt.imshow(img)
    
    # Show the tinted image
    plt.subplot(2, 1, 2)
    
    #  we explicitly cast the image to uint8 before displaying it.
    plt.imshow(np.uint8(img_tinted))
    plt.show()
    

![enter image description
here](http://images.gitbook.cn/47359820-7b8d-11e8-b075-95736529b697)

### 2.6 总结

本文主要介绍了一些 Python 的基础知识，包括为什么选择 Python、函数和类、向量化和矩阵、广播、Matplotlib 绘图等。Python
的功能非常强大，其包含的内容也太多了，我们不可能在一节课程里介绍所有的 Python
知识。我在本文介绍的几点内容是神经网络编程的必备知识点，这些内容都会在接下来的课程编程中用到。

