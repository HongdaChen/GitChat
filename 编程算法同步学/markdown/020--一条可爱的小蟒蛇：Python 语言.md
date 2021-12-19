#### 多姿多彩的编程语言

有人类比着化学元素周期表制作了一张“编程语言周期表”

![enter image description
here](https://images.gitbook.cn/906d7300-8b4f-11e9-b38f-03c8201e19f7)

表中每一行表示一个年代。第一行是前1950年代，第二行是1950年代，之后分别是1960，1970……2000年代。不同的颜色则表示不同的编程范型。

全表第一个语言（左上角）是1837年由Charles Baggage和Ada
Lovelace创造的分析机代码——说是代码，其实就是后者用前者发明的机械式分析机计算伯努利数的详细算法说明而已。

后来，这段算法说明被认为是世界上第一个计算机程序。因此，英国数学家兼作家Ada
Lovelace也被认为是世界上第一位程序员。顺便说一句，她是诗人拜伦的女儿（长得下图这样）。

![enter image description
here](https://images.gitbook.cn/9aa98930-8b4f-11e9-b38f-03c8201e19f7)

为了纪念她，1980年12月10日，美国国防部创造了一种以Ada为名的计算机编程语言——排在周期表22位。

#### 主流编程语言

存在这么多编程语言，如果真的打算从事编程相关的工作的话，要学多少种啊？

别急，这些语言并非都那么常用。其中经常被用到的，也不过就十来种。

下图是2018年的TIOBE编程语言排行榜：

![enter image description
here](https://images.gitbook.cn/a80aad20-8b4f-11e9-b38f-03c8201e19f7)

> **小贴士**
> ：TIOBE排行榜是根据互联网上有经验的程序员、课程和第三方厂商的数量，并使用搜索引擎（如Google、Bing、Yahoo!）以及Wikipedia、Amazon、YouTube统计出的排名数据，反映了编程语言的热门程度。

从中不难看出，有几种语言——Java，C/C++, Python等——多年来一直占据主导。

我们如果学，当然要学主流啦。本门课程中，我们的选择就是——Python。

#### 为什么选Python

##### Python的发展趋势

根据以高收入国家 Stack Overflow 问题阅读量为基础的主要编程语言趋势统计（下图），可以看出，近年来，Python 已然力压 Java 和
Javascript，成为发达国家增长最快的编程语言。

![enter image description
here](https://images.gitbook.cn/c3de6500-8b4f-11e9-abd4-3359f30b3591)

也就是说，Python的普及率存量虽然还不是第一，但增量（至少在发达/高收入国家）已经是第一位的了！我们是不是应该顺应潮流，学势头最猛的呢？

Python为什么能够有这样的势头？这是和它自身的特点分不开。

#### Python的特性

![enter image description
here](https://images.gitbook.cn/d9180ca0-8b4f-11e9-a560-030275e59b13)

关于Python，之前我们已经介绍了很多，总结一下：

  * 它是一种解释型语言；
  * 对过程式、面向对象、函数式编程都有支持；
  * 支持动态类型，不需要专门声明变量类型，而可以直接赋值；
  * 提供垃圾自动回收机制，能够自动管理内存；
  * 它的解释器几乎可以在所有的操作系统中运行，这使得Python具备了跨平台属性。

Python的 **设计哲学** 是：优雅、明确、简单——强调代码的可读性和语法简洁。

回头看看上一章中的9份同功（能）异（编）码的程序，有没有发现，Python是最短的？而且，Python看起来最像人话，对吧？因此从最直观的角度，Python就一种“亲人”的语言。

Python的 **设计者希望** ：

  * Python程序员能够用尽量少的代码表达想法，而无须顾忌底层实现；

  * 而且同样的功能，不同的人实现起来最好代码尽量一致。

这就使得我们在使用Python编程时，可以把重点放在程序的逻辑上，而不必一边担心怎么实现功能，一边还要考虑这些数据放在内存哪里，有没有操作会互相影响等。

Python还有一个很重要的特征： **可扩展性** ：

  * 它的语言核心只包含最基本的特性和功能，而大量复杂的针对性功能，例如：系统管理、网络通信、文本处理、数据库管理、图形界面、XML/JSON处理等等，都由其标准支持库来实现。

  * Python提供了丰富的API和工具，使得程序员能够用C、C++等语言编写扩展模块，再通过API与Python集成。也正是如此，Python又被成为“胶水语言”，常被用做其他语言和工具之间的胶水。

这一特性也使得，Python社区提供了大量的第三方支持库，功能覆盖科学计算、机器学习、Web开发等多个热点领域——有了Python，你简直可以做任何事情。

当然，Python也不是没有 **缺点** ，最大的缺点就是慢！它的解释执行机制、动态数据类型设定、可扩展性等设计都造成了它运行效率的低下。

不过，其实大多数程序并不需要很快的运行速度。尤其是对于我们这门课而言，我们需要编写的都是基础的算法，输入数据更是少得可怜，Python的速度对我们是足够用了。

### Python的编辑、运行环境

##### 两个软件

要学习编写Python程序，我们当然要安装Python的编辑和运行环境。为此，我们需要两个软件——Python 3.6.6 和PyCharm
（Community Version）。

这两个其实在前面都已经提到过了。现在再给一遍下载地址：

https://www.python.org/downloads/release/python-366/
https://www.jetbrains.com/pycharm/download/download-
thanks.html?platform=windows&code=PCC

放心，它们都是免费的！

#### 顺序安装

下载之后直接在你准备用来编程的机器上运行安装就好。

先装Python 3.6.6，再装PyCharm，这样在PyCharm里选择运行环境就很方便了。

![enter image description
here](https://images.gitbook.cn/f715a960-8b4f-11e9-a560-030275e59b13)

![enter image description
here](https://images.gitbook.cn/07934f90-8b50-11e9-abd4-3359f30b3591)

#### 创建项目

我们打开PyCharm后直接创建一个项目（Project），并指定它的解释器路径：

![enter image description
here](https://images.gitbook.cn/11912440-8b50-11e9-b5ce-69c389c366b3)

![enter image description
here](https://images.gitbook.cn/185ca4c0-8b50-11e9-b38f-03c8201e19f7)

以后我们的代码都放在这个项目下面，既方便管理又方便运行：

#### 开始编写第一个程序

![enter image description
here](https://images.gitbook.cn/419bb9c0-8b50-11e9-a560-030275e59b13)

都安装好以后，需要测试一下我们的运行环境是否正常。

怎么测试呢？写一个程序运行一下，看看运行过程是否正常，能否得到预期的输出。

可是我们还没有学过编程啊，怎么写程序呢？

这时候，就轮到我们的helloworld程序出场了！

【1】首先先创建一个python文件——后缀为.py的文本文件。

你可以用notepad创建，再用PyCharm打开。

也可以直接用PyCharm创建：右键点击PyCharm中的项目名（Project Name）-> New -> Python File （点击）——

![enter image description
here](https://images.gitbook.cn/ad39d0e0-8cfe-11e9-9124-573977cc3c83)

然后在新出现的空白文件里填写：

![enter image description
here](https://images.gitbook.cn/b6b9ea10-8cfe-11e9-bddd-6354f353caa8)

然后直接右键点击helloworld.py文件的标题，选择“Run ‘helloworld’”，就可以直接运行了。

![enter image description
here](https://images.gitbook.cn/c050bf90-8cfe-11e9-a405-9b70cbe980f0)

结果如下：

![enter image description
here](https://images.gitbook.cn/c9a11630-8cfe-11e9-88ad-6bea0e60fb76)

红圈圈中的部分就是这个程序的输出。

能够正常输出，就说明我们的安装是成功的，运行环境和IDE都已经可以工作了。

那么这个程序是什么意思呢？我们下回再讲。

