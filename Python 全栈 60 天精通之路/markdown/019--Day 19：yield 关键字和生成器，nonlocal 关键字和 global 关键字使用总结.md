### yield 关键字

很多 Python 开发者，尤其是新手，对 yield 关键字总是望而生畏，据 Stack Overflow 网站统计，yield
关键字的被提问频次高居Python类首位。那么 yield 关键字到是干什么的？从以下几个方面论述。

#### **直观理解 yield**

要想通俗理解 yield，可结合函数的返回值关键字 return，yield 是一种特殊的 return。说是特殊的 return，是因为执行遇到
yield 时，立即返回，这是与 return 的相似之处。

不同之处在于：下次进入函数时直接到 yield 的下一个语句，而 return 后再进入函数，还是从函数体的第一行代码开始执行。

带 yield 的函数是生成器，通常与 next 函数结合用。下次进入函数，意思是使用 next 函数进入到函数体内。

举个例子说明 yield 的基本用法。下面是被熟知的普通函数 f，f() 就会立即执行函数：

    
    
    In [1]: def f():
       ...:     print('enter f...')
       ...:     return 'hello'
    
    In [2]: ret = f()
    enter g...
    
    In [3]: ret
    Out[3]: 'hello'
    

但是，注意观察下面新定义的函数 f，因为带有 yield，所以是生成器函数 f：

    
    
    def f():
        print('enter f...')
        yield 4
        print('i am next sentence of yield')
    

执行 f，并未打印任何信息：

    
    
    g = f()
    

只得到一个生成器对象 g。再执行 next(g) 时，输出下面信息：

    
    
    In [18]: next(g)
    enter f...
    Out[18]: 4
    

再执行 next(g) 时，输出下面信息：

    
    
    In [23]: next(g)
    i am next sentence of yield
    ---------------------------------------------------------------------------
    StopIteration                             Traceback (most recent call last)
    <ipython-input-23-e734f8aca5ac> in <module>
    ----> 1 next(g)
    
    StopIteration:
    

输出信息 `i am next sentence of yield`，表明它直接进入 yield 的下一句。

同时，抛出一个异常 StopIteration，意思是已经迭代结束。

此处确实已经迭代结束，所以此异常无需理会。还可以捕获此异常，以此为依据判断迭代结束。

#### **yield 与生成器**

函数带有 yield，就是一个生成器，英文 generator，它的重要优点之一 **节省内存** 。可能有些读者不理解怎么能节省的？下面看一个例子。

首先定义一个函数 myrange：

    
    
    def myrange(stop):
        start = 0
        while start < stop:
            yield start
            start += 1
    

使用生成器 myrange：

    
    
    for i in myrange(10):
        print(i)
    

返回结果：

    
    
    0
    1
    2
    3
    4
    5
    6
    7
    8
    9
    

整个过程空间复杂度都是 O(1)，这得益于 yield 关键字， **遇到就返回且再进入执行下一句** 的机制。

以上完整过程，动画演示如下：

![](https://images.gitbook.cn/03d3e410-5697-11ea-987f-4debc547abef)

如果不使用 yield，也就是使用普通方法，如下定义 myrange：

    
    
    def myrange(stop):
        start = 0
        result = []
        while start < stop:
            result.append(start)
            start += 1
        return result
    
    for i in myrange(10):
        print(i)
    

如果不使用 yield 关键字，用于创建一个序列的 myrange 函数，空间复杂度都是 O(n)。

以上完整过程，动画演示如下：

![](https://images.gitbook.cn/03d3e410-5697-11ea-987f-4debc547abef)

#### **send 函数**

带 yield 的生成器对象里还封装了一个 send 方法。下面的例子：

    
    
    def f():
        print('enter f...')
        while True:
            result = yield 4
            if result:
                print('send me a value is:%d'%(result,))
                return
            else:
                print('no send')
    

按照如下调用：

    
    
    g = f()
    print(next(g))
    print('ready to send')
    print(g.send(10))
    

输出结果：

    
    
    enter f...
    4
    ready to send
    send me a value is:10
    ---------------------------------------------------------------------------
    StopIteration                             Traceback (most recent call last)
    <ipython-input-43-0f6c10a3ae1e> in <module>
          2 print(next(g))
          3 print('ready to send')
    ----> 4 print(g.send(10))
    
    StopIteration:
    

以上完整过程，动画演示如下：

![](https://images.gitbook.cn/f468c9e0-5697-11ea-b227-d5a95658c855)

分析输出的结果：

  * `g = f()`：创建生成器对象，什么都不打印
  * `print(next(g))`：进入 f，打印`enter f...`，并 yield 后返回值 4，并打印 4
  * `print('ready to send')`
  * `print(g.send(10))`：send 值 10 赋给 result，执行到上一次 yield 语句的后一句打印出 `send me a value is:10`
  * 遇到 return 后返回，因为 f 是生成器，同时提示 StopIteration

通过以上分析，能体会到 send 函数的用法：它传递给 yield 左侧的 result 变量。

return 后抛出迭代终止的异常，此处可看作是正常的提示。

理解以上后，再去理解 Python 高效的“协程”机制就会容易多了。

#### **更多使用 yield 案例**

**1\. 完全展开 list**

下面的函数 deep_flatten 定义中使用了 yield 关键字，实现嵌套 list 的完全展开。

    
    
    def deep_flatten(lst):
        for i in lst:
            if type(i)==list:
                yield from deep_flatten(i)
            else:
                yield i
    

deep_flatten 函数，返回结果为生成器类型，如下所示：

    
    
    In [10]: gen = deep_flatten([1,['s',3],4,5])
    
    In [11]: gen
    Out[11]: <generator object deep_flatten at 0x000002410D1C66C8>
    

返回的 gen 生成器，与 for 结合打印出结果：

    
    
    In [14]: for i in gen:
        ...:     print(i)
    1
    s
    3
    4
    5
    

`yield from` 表示再进入到 deep_flatten 生成器。

下面为返回值 s 的帧示意图：

![](https://images.gitbook.cn/2a3876f0-5699-11ea-8ccf-d385e6cc64ec)

**2\. 列表分组**

    
    
    from math import ceil
    
    def divide_iter(lst, n):
        if n <= 0:
            yield lst
            return
        i, div = 0, ceil(len(lst) / n)
        while i < n:
            yield lst[i * div: (i + 1) * div]
            i += 1
    
    list(divide_iter([1, 2, 3, 4, 5], 0))  # [[1, 2, 3, 4, 5]]
    list(divide_iter([1, 2, 3, 4, 5], 2))  # [[1, 2, 3], [4, 5]]
    

完整过程，演示动画如下：

![](https://images.gitbook.cn/70b1d8b0-5699-11ea-908b-b5332759803b)

这也是一个很经典的 yield 使用案例，优雅的节省内存，做到 O(1) 空间复杂度。

以上就是 yield 的主要用法和案例，下面介绍另外两个有用的关键字。

### nonlocal 关键字

关键词 nonlocal 常用于函数嵌套中，声明变量为 **非局部变量** 。

如下，函数 f 里嵌套一个函数auto_increase。实现功能：不大于 10 时自增，否则置零后，再从零自增。

    
    
    In [4]: def f():
       ...:     i=0
       ...:     def auto_increase():
       ...:         if i>=10:
       ...:             i = 0
       ...:         i+=1
       ...:     ret = []
       ...:     for _ in range(28):
       ...:         auto_increase()
       ...:         ret.append(i)
       ...:     print(ret)
    

调用函数 f，会报出如下异常：

    
    
    <ipython-input-9-5ca6794fdb70> in auto_increase()
          2     i=0
          3     def auto_increase():
    ----> 4         if i>=10:
          5             i = 0
          6         i+=1
    UnboundLocalError: local variable 'i' referenced before assignment
    

`if i>=10` 这行报错，i 引用前未被赋值。

为什么会这样？明明 i 一开始已经就被定义！

原来，最靠近变量 i 的函数是 auto _increase，不是 f，i 没有在 auto_ increase 中先赋值，所以报错。

解决方法：使用 nonlocal 声明 i 不是 auto_increase 内的局部变量。

修改方法：

    
    
    In [11]: def f():
        ...:     i=0
        ...:     def auto_increase():
        ...:         nonlocal i # 使用 nonlocal 告诉编译器，i 不是局部变量
        ...:         if i>=10:
        ...:             i = 0
        ...:         i+=1
        ...:     ret = []
        ...:     for _ in range(28):
        ...:         auto_increase()
        ...:         ret.append(i)
        ...:     print(ret)
    
    In [12]: f()
    [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 1, 2, 3, 4, 5, 6, 7, 8]
    

调用 f()，正常输出结果。

### global 关键字

先回答为什么要有 global。一个变量被多个函数引用，想让全局变量被所有函数共享。有的伙伴可能会想这还不简单，这样写：

    
    
    i = 5
    def f():
        print(i)
    
    def g():
        print(i)
        pass
    
    f()
    g()
    

f 和 g 两个函数都能共享变量 i，程序没有报错，所以，至此依然没有真正解释global 存在的价值。

但是，如果某个函数要修改 i，实现递增，这样：

    
    
    def h():
        i += 1
    
    h()
    

此时执行程序，就会出错，抛出异常 UnboundLocalError，原来编译器在解释 `i+=1` 时，会解析 i 为函数 h()
内的局部变量。很显然，在此函数内，解释器找不到对变量 i 的定义，所以报错。

global 在此种场景下，会大显身手。

在函数 h 内，显示地告诉解释器 i 为全局变量，然后，解释器会在函数外面寻找 i 的定义，执行完 `i+=1` 后，i 还为全局变量，值加 1：

    
    
    i = 0
    def h():
        global i
        i += 1
    
    h()
    print(i)
    

### 小结

今天一起与大家总结有用的三个关键字：

  * yield
  * nonlocal
  * global

yield 关键字和生成器用法四个方面总结及三个例子，nonlocal 关键字、global 关键字各在哪些场景下，发挥重要作用。

**动画对应短视频下载链接：**

> <https://pan.baidu.com/s/1MrnrXEnl54XDYYwkZbSqSA>
>
> 提取码：634k

