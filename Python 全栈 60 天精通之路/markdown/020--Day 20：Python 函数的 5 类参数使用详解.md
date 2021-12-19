Python 五类参数：

  * 位置参数
  * 关键字参数
  * 默认参数
  * 可变位置参数
  * 可变关键字参数

理解它们，有助于写出更加灵活多变的函数。

下面，详细解释每类参数的使用规则及注意事项。

### 五类参数

定义函数 f，只有一个参数 a，a 既可能为位置参数，也可能为关键字参数，这取决于调用函数 f 的传参。

    
    
    def f(a):
      print(f'a:{a}')
    

下面这样调用 f，a 就是位置参数，英文 positional argument，并且 a 被赋值为 1：

    
    
    In [8]: f(1)
    a:1
    

如果这样调用 f，a 就是关键字参数，英文 keyword argument，a 同样被赋值为 1：

    
    
    In [9]: f(a=1)
    a:1
    

如果函数 f 定义为如下，有一个默认值 0：

    
    
    def f(a=0):
        print(f'a:{a}')
    

那么，a 就是默认参数，且默认值为 0。

有以下两种不同的调用方式：

**1\. 按照 a 的默认值调用：**

    
    
    In [11]: f()
    a:0
    

**2\. 默认参数 a 值为 1：**

    
    
    In [12]: f(1)
    a:1
    

如果函数 f 定义为，如下：

    
    
    def f(a,*b,**c):
        print(f'a:{a},b:{b},c:{c}')
    

函数 f 的参数稍显复杂，但也是最常用的函数参数定义结构。

  * 出现带一个星号的参数 b，这是可变位置参数；
  * 带两个星号的参数 c，这是可变关键字参数。

可变表示函数被赋值的变量个数是变化的。例如，可以这样调用函数：

    
    
    f(1,2,3,w=4,h=5)
    

参数 b 被传递 2 个值，参数 c 也被传递 2 个值。可变位置参数 b 被解析为元组，可变关键字参数 c 被解析为字典。

    
    
    In [17]: f(1,2,3,w=4,h=5)
    a:1,b:(2, 3),c:{'w': 4, 'h': 5}
    

如果这样调用函数：

    
    
    f(1,2,w=4)
    

参数 b 被传递 1 个值，参数 c 被传递 1 个值。

    
    
    In [18]: f(1,2,w=4)
    a:1,b:(2,),c:{'w': 4}
    

所以，称带星号的变量为可变的。

### 查看参数

借助 Python 的 inspect 模块查看参数的类型。

首先，导入 inspect 模块：

    
    
    from inspect import signature
    

函数 f 定义为：

    
    
    def f(a,*b):
      print(f'a:{a},b:{b}')
    

查看参数类型：

    
    
    In [24]: for name,val in signature(f).parameters.items():
        ...:     print(name,val.kind)
    
    a POSITIONAL_OR_KEYWORD
    b VAR_POSITIONAL
    

看到，参数 a 既可能是位置参数，也可能是关键字参数。参数 b 为可变位置参数。

但是，如果函数 f 定义为：

    
    
    def f(*,a,**b):
      print(f'a:{a},b:{b}')
    

查看参数类型：

    
    
    In [22]: for name,val in signature(f).parameters.items():
        ...:     print(name,val.kind)
        ...:
    a KEYWORD_ONLY
    b VAR_KEYWORD
    

看到参数 a 只可能为 KEYWORD_ONLY 关键字参数。因为参数 a 前面有个星号，星号后面的参数 a 只可能是关键字参数，而不可能是位置参数。

不能使用如下方法调用 f：

    
    
    f(1, b=2, c=3)
    
    TypeError: f() takes 0 positional arguments but 1 was given
    

下面总结，Python 中五类参数的传递赋值规则。

### 传递规则

Python 强大多变，原因之一在于函数参数的多样化。

方便的同时，也要求开发者遵守更多的约束规则。如果不了解这些规则，函数调用时，可能会出现各种各样的调用异常。

常见的有以下六类：

  * `SyntaxError: positional argument follows keyword argument`，位置参数位于关键字参数后面
  * `TypeError: f() missing 1 required keyword-only argument: 'b'`，必须传递的关键字参数缺失
  * `SyntaxError: keyword argument repeated`，关键字参数重复
  * `TypeError: f() missing 1 required positional argument: 'b'`，必须传递的位置参数缺失
  * `TypeError: f() got an unexpected keyword argument 'a'`，没有这个关键字参数
  * `TypeError: f() takes 0 positional arguments but 1 was given`，不需要位置参数但却传递 1 个

下面总结，6 个主要的参数使用规则。

**规则 1：不带默认值的位置参数缺一不可**

函数调用时，根据函数定义的参数位置来传递参数，是最常见的参数类型。

    
    
    def f(a):
      return a
    
    f(1) # 这样传递值，参数 a 为位置参数
    

如下带有两个参数，传值时必须两个都赋值。

    
    
    def f(a,b):
        pass
    
    In [107]: f(1)
    
    TypeError: f() missing 1 required positional argument: 'b'
    

**规则 2：关键字参数必须在位置参数右边**

在调用 f 时，通过 **键--值** 方式为函数形参传值。

    
    
    def f(a):
      print(f'a:{a}')
    

参数 a 变为关键字参数：

    
    
    f(a=1)
    

但是，下面调用，就会出现：位置参数位于关键字参数后面的异常。

    
    
    def f(a,b):
        pass
    
    In [111]: f(a=1,20.)
    
    SyntaxError: positional argument follows keyword argument
    

**规则 3：对同一个形参不能重复传值**

下面调用也不 OK：

    
    
    def f(a,**b):
        pass
    
    In [113]: f(1,width=20.,width=30.)
    
    SyntaxError: keyword argument repeated
    

**规则 4：默认参数的定义应该在位置形参右面**

在定义函数时，可以为形参提供默认值。

对于有默认值的形参，调用函数时，如果为该参数传值，则使用传入的值，否则使用默认值。

如下 b 是默认参数：

    
    
    def f(a,b=1):
      print(f'a:{a}, b:{b}')
    

默认参数通常应该定义成不可变类型。

**规则 5：可变位置参数不能传入关键字参数**

如下定义的参数 a 为可变位置参数：

    
    
    def f(*a):
      print(a)
    

调用方法：

    
    
    In [115]: f(1)
    (1,)
    
    In [116]: f(1,2,3)
    (1, 2, 3)
    

但是，不能这么调用：

    
    
    In [117]: f(a=1)
    
    TypeError: f() got an unexpected keyword argument 'a'
    

**规则 6：可变关键字参数不能传入位置参数**

如下，a 是可变关键字参数：

    
    
    def f(**a):
      print(a)
    

调用方法：

    
    
    In [119]: f(a=1)
    {'a': 1}
    
    In [120]: f(a=1,b=2,width=3)
    {'a': 1, 'b': 2, 'width': 3}
    

但是，不能这么调用：

    
    
    In [121]: f(1)
    
    TypeError: f() takes 0 positional arguments but 1 was given
    

### 小结

今天与大家一起学习：

  * 函数的五类参数：位置参数、关键字参数、默认参数、可变位置参数、可变关键字参数
  * 查看参数类型的 inspect 模块
  * 六个参数的基本传值规则

