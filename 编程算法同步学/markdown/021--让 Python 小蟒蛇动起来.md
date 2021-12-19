### 你好世界

#### 打印 “hello world”

上一章我们写了第一个Python程序：

![enter image description
here](https://images.gitbook.cn/e8edc830-8cfe-11e9-a405-9b70cbe980f0)

是的，整个程序只有一行：print（“hello world“）

这里的print()是一个内置函数——函数概念会在后面解释，这里暂且理解成一个命令。

print()函数的功能就是打印出（在默认输出设备上输出）它后面括号中的内容。

本程序中，print后面括号里是一个字符串“hello world“。

整个程序的功能非常简单，就是在输出端输出一串字符——“hello world”，而已。

#### 程序对世界的问候

别看功能简单，它的正确运行却可以证明你的Python运行环境和IDE都安装成功且配置正确了。

对任何编程语言而言，输出都是最基本的功能。因此在所有编程语言里，都有诸如Python中print()函数功能的关键字或者函数用来完成类似的功能。直接输出一个确定的字符串，是所有有输出的程序中最简单的。

因此，大多数时候，不管用什么语言编程，在安装好（编译）运行环境以后，第一个要写的程序就是打印一个字符串。一则测试环境，二则也是给初学者一个最基础的代码示例。

### 历史由来

那为什么非要打印“hello world“（世界你好）呢？大概这是来自于程序员们虽然才只会写第一行代码却已经在期待改变世界的雄心的呼喊吧。

1972年加拿大计算机科学家Brian Kernighan在贝尔实验室用BCPL语言（C语言的前身）写出了第一个hello world程序。

几年之后他和C语言的创造者dmr合作，一同撰写了第一本关于C语言的书（下图），书中有一段样例程序打印了“hello，world“。

![enter image description
here](https://images.gitbook.cn/8b9bdbd0-8b51-11e9-abd4-3359f30b3591)

从那儿以后，打印“hello world”就逐渐成了各语言编程的开篇传统。

## 运行Python程序的几种方式

Python程序的运行有多种不同的方式：

#### 【1】在IDE中直接运行

就是上一章讲到的，在PyCharm中直接右键点击程序文件，选择运行当前程序。

![enter image description
here](https://images.gitbook.cn/20c71d60-8cff-11e9-9124-573977cc3c83)

当我们使用IDE的时候，这种方式非常 **直接** ，而且，可以在IDE中逐步Debug，十分方便。

对应的 **缺点**
是：在IDE中运行程序，相当于在原本的程序外面又“包”了一层。这样，Python程序的运行不仅受运行环境，而且还受IDE的影响，运行效率有所下降，甚至可能因为IDE的bug导致原本正确程序出问题。

> bug，debug——这些都是什么呀？别急，我们以后都会讲到。在此处先知道一下这些名词就好。

虽然缺点客观存在，不过真正出现问题的几率很小。对于我们来说， **最方便的最好用** 。而且，本课中代码都很简单，所以，我们尽可以放心用IDE直接运行。

采用这种运行方式，所有的输出会显示在专门的输出窗口里：

![enter image description
here](https://images.gitbook.cn/6f669bb0-8d01-11e9-a405-9b70cbe980f0)

#### 【2】 命令行运行Python文件

在命令行直接通过Python的运行命令执行.py文件。

Python的运行命令非常简单，就是“python”，后面跟上要运行的文件的相对（或者绝对）路径即可。如同下图中第一行那样。

![enter image description
here](https://images.gitbook.cn/a3efd0f0-8d00-11e9-a405-9b70cbe980f0)

所有的输出都在命令行界面直接现实。上图中第二行就是运行 HelloWorld.py 的输出。

这种运行方式直接在运行环境中运行Python脚本（Python文件），不受IDE影响。对于已经确认正确且需要反复运行的Python脚本非常合适。

不过如果要边编辑程序边试运行的话，稍微有点麻烦。

#### 【3】运行Python命令

我们还可以在Python运行环境中直接运行一句句的Python语句。

具体方法是在命令行输入“python” —— 注意，后面不跟随任何文件路径：

![enter image description
here](https://images.gitbook.cn/4b181030-8d02-11e9-9124-573977cc3c83)

只要出现了">>>"，就说明已经进入了运行环境，可以直接键入python语句啦。比如像下图这样：

![enter image description
here](https://images.gitbook.cn/b388f8a0-8d02-11e9-9124-573977cc3c83)

每输入一个Python语句，对应的输出就会直接显示在下一行。

一般我们用这种方式来运行一两个临时语句；验证某条语句语法的正确性；或者查看某个支持库是否被安装等。真的用这种方式来编程、写算法挺费劲的。

* * *

本课中后续程序代码，都推荐采用第【1】中运行方式运行。

#### 编程语言的基本概念

下面是一个编程语言和自然语言中概念的大致对照，不严格但直观：

![enter image description
here](https://images.gitbook.cn/9b77ead0-8b51-11e9-b5ce-69c389c366b3)

上面这些，不是Python专有的，而是绝大多数编程语言都会涉及到的概念。

这么多东西，一口吃不成胖子，我们只能一点点逐步去接触。

传统的教学方法会在一开始一个个讲解这些概念。作者以为这种讲法比较低效，倒不如先建立一个大致的轮廓，用到什么再具体去看什么。

今天我们先着重讲讲print()函数的用法。

### Python的print()函数

#### 内置函数

print()是Python3的一个非常重要的内置函数。

> **NOTE** ：print() 在 Python2
> 只是一个关键字，到了Python3中才成为了一个函数。这也就导致到了在Python2和Python3中打印输出的语法很不相同。
>
> 因为我们的课程采用Python3，因此后续说的Python，没有特殊说明，都指Python3。

**内置函数** ，指那些在Python最核心的运行环境中已经被实现了功能，无须安装任何额外的支持库，就可以拿来就用的函数。

虽然Python的功能异常强大，画图、制表、数据处理、机器学习、深度学习无所不能，但其实大部分的功能都是靠支持库来实现的，真正的内置函数还不到70个。

这些内置函数每一个都非常常用，而其中最常用的，大概就是我们的print()了。毕竟从第一个程序开始，我们就需要打印输出。

### 功能强大的print()函数

print()函数可以打印出各式各样的花样。其中很多花样和我们后要讲的变量概念以及列表等数据类型有关，那些到时候再讲。在这里，我们先看看最基本的字符串和数字的打印。

#### 打印字符串

print()函数可以直接打印字符串，例如：

`print("hello world")`

还可以把几个字符串拼接起来打印。具体的拼接办法有多种，我们先来看最简单的几种：

  * 用“ + ”连缀字符串，被相加的字符串首尾相接。

  * 用“ ，”分割多个字符串，被分割的字符串打印出来的效果也是首尾相接，不过每个字符串之间会多一个空格。

  * 用 "%s" 符号将一些小字符串嵌入到一个大字符串中。

我们来看一下代码实现的例子，下面这个程序中包含了上述三种字符串拼接方法：

    
    
    print("Today is " + "Monday")
    
    print("Today:", "Monday")
    
    print("Today is %s, and the main course of our dinner will be %s." % ("Monday", "Fish"))
    

上面这些代码我们可以放在一个新创建的Python文件里：

![enter image description
here](https://images.gitbook.cn/1ebe1dd0-8d8a-11e9-bfdc-ab83f29258da)

然后还是右键点击py文件运行，输出结果如下：

> Today is Monday Today: Monday Today is Monday, and the main course of our
> dinner will be Fish.

print()函数打印字符串的时候，被打印字符串需要被引号引起来，这个引号是双引号或者是单引号都可以，下面的两行代码是同样的含义：

    
    
    print("Today is %s, and the main course of our dinner will be %s." % ("Monday", "Fish"))
    
    print('Today is %s, and the main course of our dinner will be %s.' % ('Monday', 'Fish'))
    

输出也是同样的：

> Today is Monday, and the main course of our dinner will be Fish. Today is
> Monday, and the main course of our dinner will be Fish.

这里就有个问题了：无论是以单引号还是双引号作为引用的标识，如果我们要打印的字符串里有同样的引号，那不会让Python解释器误会吗？

如果不做任何处理，当然难免误会。所以，我们要告诉解释器，哪个引号是用来把打印目标“引起来”的，哪个是打印内容。

方法很简单，在作为打印内容的引号前面前面加一个反斜杠——此处的反斜杠叫做转义字符。下面是这样处理的例子：

    
    
    print('Tom asked: "Where is my ball?"')
    print("Jack's brother hided a ball behind Jack's body.")
    print("Jack told his brother: \"Bob, don't do that. You should return the ball to Tom.\"")
    print('Bob cried. Tom\'s sister tried to console him, but didn\'t succeed.')
    print('Tom shouted loudly: "Return my ball! Otherwise, I will call you \'The Thief\' all the time!"')
    print("Tom's sister said: \"No, he is a little boy, not a thief. Let's play together with the ball.\"")
    

对应的输出是：

> Tom asked: "Where is my ball?" Jack's brother hided a ball behind Jack's
> body. Jack told his brother: "Bob, don't do that. You should return the ball
> to Tom." Bob cried. Tom's sister tried to console him, but didn't succeed.
> Tom shouted loudly: "Return my ball! Otherwise, I will call you 'The Thief'
> all the time!" Tom's sister said: "No, he is a little boy, not a thief.
> Let's play together with the ball."

通过上面的示例，怎么用反斜杠（\）转义符，是不是很清楚了？

#### 打印数字

print()函数打印数字更方便一些，直接在括号里写数字（整数、小数都可以）就行，不用加引号。代码：

    
    
    print(3)
    print(5.96)
    

输出：

> 3 5.96

而且，还可以直接在print()函数的括号内放置算式，打印的结果就是计算结果。代码：

    
    
    print(6-2)
    print(7.5/3)
    print((3-1.12)*10/4*(2-3.325)+3*2-5/10)
    

输出是：

> 4 2.5 -0.7275

#### 打印数字与字符串的混合

数字也可以和字符串一样，嵌入到字符串里面去。有所不同的是：

  * 字符串嵌入字符串，在嵌入位置用“%s”标识；
  * 数字嵌入字符串，在嵌入位置可以用“%s”标识，也可以用“%d”标识。

如果用%s标识，则数字完全被当作字符串处理；如果用%d标识，则对应数字的整数部分会被作为字符串输出，而小数部分则根本不会输出。

请注意下面几行代码中的%s，%d，以及对应数字的现实：

print("The price of this %s is %d dollars." % ("hotdog", 4.0)) print("The
price of this %s is %s dollars." % ("hotdog", 4.0)) print("The price of this
%s is %d dollars." % ("meal", 12.25)) print("The price of this %s is %s
dollars." % ("meal", 12.25))

输出是：

> The price of this hotdog is 4 dollars. The price of this hotdog is 4.0
> dollars. The price of this meal is 12 dollars. The price of this meal is
> 12.25 dollars.

各种区别，可以多做几组尝试来自己体会，也可以图省事，干脆所有字符串或者数字都用%s标识好了！

前面讲的另两个拼接字符串的方法：用“+”和用“，”，也可以用于数字与字符串的拼接。两种方法但不同之处在于：

  * 如果用“+”，则需要将数字也转化为字符串——具体方法是用另一个Python内置函数：str()；
  * 如果用“，”，则数字可以直接作为一个分项，而无需事先转化为字符串。

下面的例子是这两种用法的实现，除了str()，大家请注意各个小字符串中的空格：

    
    
    print("Monday Food: " + str(2) + " Apples " + "and " + str(3) + " Carrots")
    print("Monday Food:", 2, "Apples", "and", 3, "Carrots")
    

输出如下：

> Monday Food: 2 Apples and 3 Carrots Monday Food: 2 Apples and 3 Carrots

#### 记住print()函数的用法

上面介绍的，不过是print()函数用法中很少的一部分而已——当然是最常用的那部分。

是不是已经觉得有点混乱了？会不会又在想：学这么多这儿一个百分号那儿一个空格的东西到底有什么用啊？

回顾一下我们之前说的：编程语言中的函数就像自然语言中的短语、成语或者俗语，是一种约定的用法。不用的时候，感觉背记的是一堆没用的东西，等到需要在程序中打印的时候，就是“书到用时方恨少”了。

print()是程序最直接的输出，也是在不使用任何额外工具的情况下debug程序的最直接手段。这些我们在今后的代码中会慢慢见识。

现在要做的，仅仅是把今天学过的这些用法记住！也许一开始是死记硬背，但是在毫无基础的时候能指望什么巧记巧背呢？

