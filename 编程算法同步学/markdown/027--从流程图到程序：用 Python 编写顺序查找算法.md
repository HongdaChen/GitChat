前面学了一大堆变量啊，赋值啊，到底有什么用呢？

马上用处就来了——

### 修改顺序查找流程图——用变量和赋值替换其中的自然语言

#### 以前的流程图

还记得我们之前已经学习了顺序查找算法吧（还记得吗）？

当时我们是画了一张流程图。先来回顾一下：

![enter image description
here](https://images.gitbook.cn/80401040-9311-11e9-9417-bf698e8a446d)
**_流程图-1_**

在这种图里，我们其实已经使用了数组（列表）变量arr，和整型变量i，只不过当时我们还不知道它们是 **变量**
，而仅仅将其当作数组和数组下标的形式化写法而已。

#### 重画流程图

通过前面两章的学习我们知道，上图中大部分关于arr和i的操作就是对数组和整型变量赋值的。

在Python中我们用列表来实现数组操作，其实此处arr是列表类型的变量。而len(arr)则是Python的一个内置函数，是用来计算列表长度的。

那么我们是不是可以利用已经学过的Python语言的要素，将上图中余下的自然语言文字也替换掉，都换成赋值语句或者内置函数的调用操作呢？

我们来试试：

![enter image description
here](https://images.gitbook.cn/9bb0a410-9312-11e9-bf22-7d49113c7821)
**_流程图-2_**

    
    
    在程序开始的时候，先初始化两个变量：tn——整型，目标数（target number）和arr——列表型，存放数列的数组。
    
    然后给i赋初值0，之后进入循环，循环的准入条件是i小于arr的长度——即元素个数。
    
    在循环内部，将tn和arr[i]比较，如果相等则打印出对应元素的索引，然后退出循环（break表示退出当前循环），否则i递增1之后继续循环。
    
    一旦到了i的值和arr的元素个数相等，就说明整个arr里面所有的元素都与tn比较过，且都不相等，此时打印出“failed”告知用户查找失败了。
    

果然可以哦！

大家发现没有，上面这张重新画的流程图里，每一个矩形或者菱形框里面的内容，都变成了几乎可以直接写在程序中的语句。

因此，只要有办法把流程图中的控制结构写成程序语句，整个算法的实现就有了！

怎么在程序中体现顺序、条件、循环三种不同的结构呢？

### 关键字与控制结构

#### 关键字

在介绍具体控制结构的写法之前，我们需要先了解一个概念——关键字！

**关键字（Keyword）** ，也叫保留字（Reserved Word），是编程语言中的一类 **语法结构** 。

基本每一种编程语言都会有一系列关键字，它们在语言的格式说明里被预先定义，通常具有特殊的语法意义。

这些意义中非常重要的内容就是用来识别代码块、函数和流程控制结构。

#### Python的关键字

Python2和Python3的关键字大体相同，既然用的是Python3我们只看Python3就好了。

其实，我们可以通过Python语句直接获得当前版本的关键字，具体代码如下：

    
    
    import keyword
    print(keyword.kwlist)
    

这么简单的代码，我们直接在命令行运行就好了：

![enter image description
here](https://images.gitbook.cn/56c0af30-9317-11e9-b9bd-69ad1684fc56)

我们可以看到，Python3的关键字总共有33个。这些关键字当然各有用处。

关于Python关键字的详解，网上到处都有，大家感兴趣可以自己搜索。

不过，作者认为，现在我们就是把每一个的作用都详细说一遍，你也记不住。不如先简单浏览一下，用到什么再具体学什么。

下面是从使用角度，对这33个关键字做的简单分类：

  * 布尔类型字面量： True False

  * 表示“不存在”的特殊型字面量： None

  * **关于流程控制** ： if elif else for while break continue

  * 关于逻辑判断： and or not

  * 关于函数： def return yield lambda

  * 关于类型判断： is

  * 关于面向对象编程： class pass

  * 关于代码块控制： with … as 

  * 关于List操作： del in

  * 关于变量作用域： global nonlocal

  * 关于异常处理： assert except finally raise try

  * 关于导入模块： from import

目前我们马上要用到的，就是控制流程相关的几个关键字。也就是if，elif，else，for，while，break，continue。其中
if，else和while是最最基础的流程控制关键字。

我们现在就来看看，用这几个关键字，怎么构造各种流程控制结构。

### Python中的三种控制结构

下面这几幅图就是直观的转换方法：

#### 顺序结构

顺序结构非常简单，实现它都不需要关键字，直接把各个步骤的代码按照从前到后的顺序罗列出来即可。

![enter image description
here](https://images.gitbook.cn/6c54c0f0-9319-11e9-9417-bf698e8a446d)
**_顺序结构_**

下面是一个顺序结构的例子：

    
    
    length = 10
    width = 5
    space = length * width
    print("Space is: %s square meters" % space)
    

**_代码-1_**

输出为：

> Space is: 50 square meters

套用我们之前学过的代码块的概念，在这段代码里，四行代码都属于同一个代码块，四行代码地位均等:

![enter image description
here](https://images.gitbook.cn/50407dd0-933e-11e9-9417-bf698e8a446d)

因此，它们要运行，就一条不差地按从上到下的顺序运行。

#### 条件结构

大多数情况下，条件结构需要用到2个关键字：if和else；有时只需要if就可以；还有的时候，另需要elif关键字。

我们先来看看大多数的情况：

![enter image description
here](https://images.gitbook.cn/9ecd2220-9319-11e9-877d-6fb1f1c96f1d)
**_条件结构_**

if关键字是条件结构必须有的关键字，if标志了一个控制结构的开始。if之后是一个空格，然后是条件语句的条件——对应流程图中菱形框内的部分。

> **小贴士** ：代码中的条件部分其实不必用括号括起来，图中这么画是为了让你看清楚条件。

if语句的结尾是一个 **冒号** ——这个冒号是必须有的，初学者经常会忽视这个冒号，结果导致程序报错却不知道错在哪里，这一点要注意。

在if行之下，有一个子代码块，这个代码块的内容对应流程图中YES之后的部分。

而NO对应的部分，需要用另一个关键字：else标识出来，else后面只有一个冒号。else行下面是原本属于NO分支的子代码块。 **子代码块** 需要 **
_一级缩进_** ，即后退4个空格。

下面是一段示例代码：

    
    
    if space > 30:
        print("This is a big room.")
    else:
        print("This is a normal room.")
    

**_代码-2_**

> **NOTE** ：这段代码实际上延续代码-1的，在代码-1中space已经被赋值了，所以此处无须再赋值。
>
> 这里例子里每个子代码块都只有一个语句，如果有多个语句的话，这些语句全部相对于“if”一级缩进，它们互相之间是对齐的。

输出是：

> This is a big room.

从代码块的角度解释，在这段代码里面，if 所在一行和else所在一行仍然属于Block1，但打印“big room”的语句和打印“normal
room”的语句就成了if语句和else语句的子代码块。结构如下：

![enter image description
here](https://images.gitbook.cn/3e6f80d0-9337-11e9-bf22-7d49113c7821)

Block2-1
由if语句控制，到了if语句时，Python解释器要检查一下变量space的值是否大于30。如果是，则进入Block2-1执行；否则则跳过Block2-1。

if语句及其子代码块整个运行过后（无论Block2-1是否真的被执行了），else语句解释器会检查一下space > 30条件的逆条件，即space <=
30，如果符合这个逆条件，则进入Block2-2，否则跳过Block2-2。

如果NO分支无内容，也可以完全没有else行及其子代码块。例如这样的代码：

    
    
    if space > 30:
        print("What a big room!")
    

**_代码-3_**

也就是只有space > 30时才打印，否则就什么都不打印。

* * *

如果有超过两种情况要考虑，则可以用elif语句，elif 实际上就是“else if”。比如下面例子：

    
    
    if space > 30:
        print("This is a big room.")
    elif space > 10:
        print("This is a normal room.")
    else:
        print("This is a small room.")
    

**_代码-4_**

代码-4的细节如下：

  * if语句：

space > 30 时输出“This is a big room.”

  * elif 语句：

space > 30 条件不满足的情况下，也就是在space <=30的时候，再检查是否space > 10，如果是（也就是 10 < space <=
30) 的时候，输出"This is a normal room."

  * else语句：

本语句的隐含条件是前面所有条件所限定的情况的并集的逆。

就是说到了else这条语句，就说名当前space变量的值，对于之前的（1）space > 30 或者 （2）10 < space <= 30
这两个条件都不满足。也就是说此时的状况是：space <= 10，针对这种情况，输出"This is a small room."

#### 循环结构

循环结构对应的关键字是while或者for。

在开始学的时候，我们先关注尽量少的关键字，暂时只用while来构造循环。实际上所有用for构造的循环都可以用while重写。

先来看最简单的情况：

![enter image description
here](https://images.gitbook.cn/c7205b10-931a-11e9-bf22-7d49113c7821)
**_循环结构-1_**

循环结构以while关键字开始，while语句格式类似if语句，也是while+空格+循环条件+冒号。

while行下也是一个子代码块，对应流程图中形成环路的YES分支。在YES分支完成后，while对应代码块也就结束了。

如果NO分支如果有内容的话，将NO分支的部分直接放在while所控制的整段代码后面。不必再回退4格。

下面是一个例子：

    
    
    arr = ["apple", "orange", "watermelon"]
    i = 0
    while i < len(arr):
        print(arr[i])
        i = i + 1
    print("No fruit.")
    

**_代码-5_**

输出为：

> apple orange watermelon No fruit.

这段代码的代码块结构如下：

![enter image description
here](https://images.gitbook.cn/c6053600-9339-11e9-877d-6fb1f1c96f1d)

Block2 由while语句控制。代码执行到while语句时，先判断当时变量i的值是否小于arr
List的长度，如果满足条件，则进入Block2执行其中的两条代码。

Block2是while循环的循环体。每次Block2执行完之后，都要再跳回到while语句，判断while的循环条件 —— i < len(arr)
是否成立。如果成立，则再进入循环体，否则跳过Block2，继续后面的Block1。

一旦Block2被跳过，说明循环条件的逆条件已经成立了。比如上面的代码，一旦 i的值不再符合i < len(arr)，则说明已经是i >= len(arr)
的情况了。

* * *

有的时候循环体中会用到break或continue关键字，前者用于忽然终止循环跳到循环之外，后者则是跳过当前这轮循环，继续进入下一轮循环。

continue一般影响不大，就是会让循环体少执行一次。但是如果遇到break要注意，在循环体中一旦遇到break语句，就直接跳出，而且不会再回头执行while中的条件判断。

这个时候，如果要保持原图中循环条件的NO分支在确定循环条件为否的时候执行，就要在while循环之后，再加一重判断。用图画出来就是：

![enter image description
here](https://images.gitbook.cn/f7005b20-933b-11e9-877d-6fb1f1c96f1d)
**_循环结构-2_** （其中“！”符号表示取反的意思）

此种结构的例子：

    
    
    arr = ["apple", "orange", "watermelon"]
    i = 0
    while i < len(arr):
        print(arr[i])
        if (arr[i] == "orange"):
            break;
        i = i + 1
    if i == len(arr):
        print("No fruit.")
    

**_代码-6_**

输出是：

> apple orange

#### 不同类型结构的嵌套

不同类型的控制结构是可以嵌套的，比如代码-6中，其实循环结构的循环体中就已经嵌套了一个条件结构了。

嵌套没有更多的限制，只要注意：

  * 每个被嵌入到其他结构中的结构都是一个完整的结构体。

    * 可以是简化的 例如：条件结构只有if，没有elif，也没有else —— 简化的结构体

    * 但不能是残缺的 例如：条件结构有if和elif，却没有else ——残缺的结构体

  * 把每个被嵌入的结构都当作一个语句来对待。

### 用Python实现顺序查找算法

#### 代码实现

我们按照刚才学的不同结构类型的代码表达，来实现一下流程图-2中描述的顺序查找算法。代码如下：

    
    
    tn = 95
    arr = [1,5,8,19,3,2,14,6,8,22,44,95,78]
    i = 0
    
    while (i < len(arr)):
        if (arr[i] == tn):
            print("tn is element", i)
            break
        else:
            i = i + 1
    
    if (i == len(arr)):
        print("failed")
    

**_代码-7_**

输出为：

> tn is element 11

#### 流程图和代码的对照

流程图和代码的对应关系如下：

![enter image description
here](https://images.gitbook.cn/8e7df740-931f-11e9-b9bd-69ad1684fc56)

这个算法有两个大块结构：顺序（浅蓝色框内）结构和循环结构（浅灰色框内）。

顺序结构自不必言，循环结构看一下代码的实现：

  1. 循环条件放在一个while语句里，符合循环条件的YES分支是是while分支下的子代码块（棕色框内）；

  2. 棕色框内部，也就是while循环的子代码块内，正好又有一个条件分支，这个条件分支的YES分支中包含了break语句；

  3. 与循环条件相悖的NO分支放在while语句之后，因为循环体中包含break语句，因此循环条件的NO分支要再用一个if语句来判断是否当时条件正好和while中的条件相反（求反）。

