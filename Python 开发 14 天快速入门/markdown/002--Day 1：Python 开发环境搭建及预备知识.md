### 特别说明

如果读者已经搭建了 Python
开发环境，可跳过本章第一部分，但需要注意，本专栏列举的实例代码中部分有中文注释，有些集成开发环境可能不支持中文注释，因此，运行本专栏实例时请注意将注释去掉。另外，如果读者觉得搭建开发环境比较繁琐，可采用
Python 自带的 IDLE
作为开发环境，安装方法请访问：[《Python3入门笔记》](https://www.cnblogs.com/weven/p/7252917.html)。

### Python 开发环境搭建

开发 Python 的 IDE 有很多，本文介绍基于 Eclipse+PyDev+Python 搭建开发环境的方法。

  * Eclipse 简介

Eclipse 是一款基于 Java 的可扩展开发平台。其官方下载中包括 J2EE 方向版本、Java 方向版本、C/C++
方向版本、移动应用方向版本等诸多版本。除此之外，Eclipse 还可以通过安装插件的方式进行诸如 Python、Android、PHP
等语言的开发。本文将要介绍的就是使用 Eclipse 与 PyDev 插件，安装 Python 开发环境的方法。

  * 环境

> OS：Windows 7
>
> Python：3.6.2
>
> Java：8u31
>
> Win7 32位，Mac OS 操作系统同下述安装方法。

#### 软件下载

##### **Eclipse 下载**

我们可以进入 [Eclipse 官网下载界面](http://www.eclipse.org/downloads/eclipse-packages/)下载
Eclipse 软件。在该页面，可以看到针对不同开发需求的 Eclipse 版本，本文采用的是 Eclipse IDE for Java and DSL
Developers。目前，最新的版本是 Eclipse Oxygen.2 (4.7.2)
Release，为2017年10月放出的版本。另外，还需要注意的是，需要根据自己的操作系统选择正确的系统位数（32/64bits）。

##### **PyDev 离线下载**

我们可以在 [PyDev 项目下载页面](http://www.pydev.org/download.html)看到一些有价值的信息：

  1. Eclipse、Java、PyDev 的版本对应关系；
  2. Eclipse 在线安装 PyDev 的 URL；
  3. 离线安装 PyDev 下载地址（Get zip releases），点击可以进入 SourceForge 的下载页面。本文介绍离线下载方法。

![enter image description
here](http://images.gitbook.cn/7886e960-3029-11e8-82cd-2720f7d588df)

#### Eclipse 安装

这里要注意，Eclipse 安装需要 Java 环境，如果还没有安装 Java 环境，请先去下载安装
JDK（[点击这里](http://www.oracle.com/technetwork/java/javase/downloads/index.html)）。

Eclipse 实际上是一个基于 OSGI 框架的软件，在 Windows 系统上无需安装，只需要将其解压，双击打开 eclipse.exe 即可。在
Mac OS 上则有所不同，需要双击 .dmg 文件安装。在第一次运行时，会要求你输入工作路径，你可以自定义也可以接受默认路径。

#### PyDev 插件安装

Eclipse 插件的安装方式有离线和在线两种，本文介绍在线安装方法。

打开 Eclipse，选择“Help”->“Install New Software”。在弹出的对话框中，点击 Add 按钮，添加新的安装源，如下图所示。

![enter image description
here](http://images.gitbook.cn/65b1a220-302a-11e8-82cd-2720f7d588df)

在 Location
处填写安装源的网址：http://www.pydev.org/updates/（很多博客中写的是http://pydev.org/updates），这个地址并非一成不变，因此，最好到官网确认一下。确认方法在上面“软件下载”小节中已有说明。

此外，需取一个名字填写在 Name 处，比如我这里写的是 PyDev。把“connect all update sites during install
to find required
software”的勾选去掉，否则在安装新插件时会联网寻找所有可能的更新站点搜索，导致安装时间不可预估，并可能导致安装失败。确定后可以看到一个
Pending 过程，然后得到如下图所示的插件，一般来说，我们只需选择 PyDev 即可，这里我两个都安装了（不推荐）：

![enter image description
here](http://images.gitbook.cn/bd9c0ac0-302a-11e8-82cd-2720f7d588df)

勾选后，点击 Next 进行安装。不过，由于网络的原因，这种方法安装 PyDev 极有可能失败，提示网络连接错误等。

#### Python 安装

Python 的安装比较简单，前往 Python
[官网](https://www.python.org/getit/)下载安装包。进入官网之后，官网会根据你的计算机操作系统推荐 Python
安装包的版本，如下图所示，你可以根据推荐下载最新的安装包，需要注意的是，Python 目前有 Python2 和 Python3
两个大版本，差异显著，本课程基于 Python3 编写，因此，请读者选择 Python3.X 安装包，具体内容安装步骤可参考[博文《Python3
入门笔记——Windows 安装与运行》](https://www.cnblogs.com/weven/p/7252917.html)。

![enter image description
here](http://images.gitbook.cn/f62efab0-3506-11e8-b802-85bd9873a174)

#### PyDev 插件配置

安装好 PyDev 插件后，并不能正常使用，还需要配置 Python 的解释器。

打开Eclipse，选择“Window” -> “Preferences”（如果是 Mac，则同时按下 Command 和 `，` 键唤出
Preference），找到“PyDev”，选择其中的“Interpreter” –> “Python”。点击“New”，添加一个系统里已有的 Python
解释器的路径（根据自己安装的路径确定）。确定后会经过短暂的处理，得到它的 Libraries、Buildins 等。当然，还可以根据自己的编程习惯对
PyDev 进行一些其他的配置，这里就不再说了。

![enter image description
here](http://images.gitbook.cn/798fc680-302c-11e8-8959-abc93f97b6a7)

#### 创建一个 Python 项目

前面就已经配置好了 Python 的开发环境，下面新建一个项目，来测试一下，确实可以运行。

点击“File” -> “New” -> “Other”，找到“PyDev”，选择“PyDev Project”，点击“Next”。取一个项目名称，比如
helloPython，此外，还可以选择 Python 的语言版本和解释器版本，如下图所示：

![enter image description
here](http://images.gitbook.cn/671a2b70-302d-11e8-82cd-2720f7d588df)

点击“Finish”，完成项目创建。然后你会进入 PyDev 视图，进行 Python 开发。这里，我们就写一个最简单的程序，进行测试。右键项目的 src
目录，选择“New” -> “PyDev Package”，创建一个 Python 包，此处也命名为 helloPython。再右键该
package，“New” -> “PyDev Module”，此处也命名为 helloPython。双击打开 helloPython.py，添加如下代码。

    
    
    if __name__ == '__main__': 
        print("hello world!")  
    

右键项目，选择“Run As” -> “Python Run”，或 Ctrl+F11 运行项目。此时，可以在下方的 Console
窗口，看到项目的运行结果：hello world!。

### Python 预备知识

#### 编程语言

编程语言（Programming
Language），是用来定义计算机程序的形式语言。它是一种被标准化的交流技巧，用来向计算机发出指令。计算机语言让程序员能够准确地定义计算机所需要使用的数据，并精确地定义在不同情况下所应当采取的行动。

##### **解释**

上面的定义读起来比较晦涩，下面我通俗的解释一下。

在人类发展史上，一般将文字的出现视为文明的标志，无论是汉语，英语还是法语，它们都是人类交流沟通的工具。文字使得人类之间的沟通和交流变得有章可循，极大的提高了信息传播的效率。自计算机诞生之后，人们一直希望给计算机赋予语言的特性，使计算机像人一样的沟通，而编程语言则可看作人与计算机之间“交流沟通”的工具，它使得人类和计算机之间的高效交流得以实现。

##### **高级编程语言**

在实践中，人们意识到人和计算机直接“交流”相当困难，计算机能够直接理解的语言是0和1构成的机器码，而这种机器码并不符合人类的认知习惯，因此，高级编程语言应运而生。

何为高级呢？指的是符合人类的理解习惯和思维方式，远离计算机底层。高级编程语言（如
Java，Python）使得人们能够以相对容易的方式将期望计算机执行的指令表达成文。但是，这种高级语言写成的“文章”，计算机是无法直接理解的，因此，还需要一个“翻译”，将人们编写的高级语言按照规则翻译成计算机能够理解的机器语言。根据编程语言的不同，这里的“翻译”也有显著区别。关于人与计算机之间的“交流”，简略的示意图如下：

![enter image description
here](http://images.gitbook.cn/ec18fe90-302e-11e8-8959-abc93f97b6a7)

#### 与 Java 和 C++ 差别显著

学习过 C/C++、Java 的读者一定还记得定义变量的方法，如下例子，定义 a、b、c、d 四个变量，其数值类型分别为
int、double、long、float，定义变量的时候，我们需要“显式”的声明变量的类型，以便在内存中为该变量开辟空间，不同的数据类型占用的内存空间是不同的，所以必须要声明。

    
    
    int a;
    double b;
    long c;
    float d;
    

与 C++ 和 Java 不同，Python 中变量没有类型，更谈不上声明变量类型，变量只是对象的引用，而 Python
的所有类型都是对象，包括函数、模块、数字、字符串、列表、元组、字典等等。

如下所示，定义变量 a
并赋值，同一个变量在同一段程序中三次被赋值不同的类型，之所以能如此操作，就是因为变量没有类型，它可以指向任何类型，单从这一点来看，Python 与
C++、Java 差别是很明显的。

    
    
    # -*- coding: UTF-8 -*
    
    a = 12
    print("a=", a)
    a = "ABCDE"
    print("a=", a)
    a = 1 + 4j
    print("a=", a)
    

**执行结果：** a= 12 a= ABCDE a= 1+4j

#### 本专栏实例的说明

为了便于读者更好的理解，本专栏列举了很多程序实例，其中部分程序涉及中文注释，另外，为了直观的展现程序执行的结果，课程中的实例大量使用 print
语句将结果打印到控制台，便于读者观察。

关于本专栏的实例风格，这里举一个简单的例子：

    
    
    # -*- coding: UTF-8 -*
    
    #自定义函数，求两个数的和
    def add(a, b):
        return a + b;
    
    #直接调用内建函数pow(x,y),计算x的y次方
    print('10的3次方 = %d' %pow(10,3))
    print('1 + 2 = %d' %add(1,2))
    
    执行结果：
    10的3次方 = 1000
    1 + 2 = 3
    

##### **特别说明**

上面例子中，有两点需要解释一下。

  * `# -*- coding: UTF-8 -*`：

Python 默认是以 ASCII 作为编码方式的，如果在自己的 Python 源码中包含了中文（或者其他非英语系的语言），即使你把自己编写的 Python
源文件以 UTF-8 格式保存了，运行时仍可能会报错：SyntaxError: Non-ASCII character '\xe5' in file
XXX.py on line YY, but no encoding declared;

解决办法：在每一个包含中文或其它语言注释的 Python 脚本文件的第一行，加入以下声明：

    
    
    # -*- coding: UTF-8 -*-
    

  * `#`

“#”表示注解，在代码中输入它的时候，它右边的所有内容都将被忽略，注解是极为有用的，不仅有助于别人理解自己的代码，也可以提示自己。

#### 通过命令运行 Python 脚本

上面的例子中，我们运行 Python 脚本都是通过 Eclipse IDE（集成开发环境）完成的，事实上运行程序的方法很多，对于 Python
语言而言，我们可以直接以命令的方式运行。

**Windows 系统** ：

进入 Python 脚本 test.py 所在路径下，键入命令：Python test.py。

注意：Python 首字母大写。

**Linux 或 Unix 系统：**

进入 Python 脚本 test.py 所在路径下，键入命令：python test.py。

注意：Python 首字母小写。

