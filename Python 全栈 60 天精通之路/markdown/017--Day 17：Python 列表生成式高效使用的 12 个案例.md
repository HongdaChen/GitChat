Python 里使用 `[]` 创建一个列表。容器类型的数据进行运算和操作，生成新的列表最高效的办法——列表生成式。

列表生成式优雅、简洁，值得大家多多使用！

今天盘点列表生成式在工作中的主要使用案例和场景。

### 1\. 数据再运算

如下，实现对每个元素的乘方操作后，利用列表生成式返回一个新的列表。

    
    
    In [1]: a = range(0,11)
    
    In [2]: b = [x**2 for x in a] # 利用列表生成式创建列表
    
    In [3]: b
    Out[3]: [0, 1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
    

![](https://images.gitbook.cn/15353c50-560f-11ea-a714-09515afb96e2)

数值型的元素列表，转换为字符串类型的列表：

    
    
    In [1]: a = range(0,10)
    
    In [2]: b = [str(i) for i in a]
    
    In [3]: b
    Out[3]: ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9']
    

![](https://images.gitbook.cn/2a115aa0-560f-11ea-908b-b5332759803b)

### 2\. 一串随机数

生成 10 个 0 到 1 的随机浮点数，保留小数点后两位：

    
    
    In [6]: from random import random
    
    In [7]: a = [round(random(),2) for _ in range(10)]
    

![](https://images.gitbook.cn/3e982d00-560f-11ea-ac19-374315264b53)

生成 10 个 0 到 10 的满足均匀分布的浮点数，保留小数点后两位：

    
    
    In [16]: from random import uniform
    
    In [10]: a = [round(uniform(0,10),2) for _ in range(10)]
    

![](https://images.gitbook.cn/56f1d950-560f-11ea-ad74-0783a5f0ad5e)

### 3\. if 和嵌套 for

对一个列表里面的数据筛选，只计算 [0,11) 中偶数的平方：

    
    
    In [10]: a = range(11)
    
    In [11]: c = [x**2 for x in a if x%2==0]
    
    In [12]: c
    Out[12]: [0, 4, 16, 36, 64, 100]
    

![](https://images.gitbook.cn/831856d0-560f-11ea-908b-b5332759803b)

**列表生成式中嵌套 for**

如下使用嵌套的列表，一行代码生成 99 乘法表的所有 45 个元素：

    
    
    In [9]: a = [i*j for i in range(10) for j in range(1,i+1)]  
    

### 4\. zip 和列表

    
    
    In [13]: a = range(5)
    
    In [14]: b = ['a','b','c','d','e']
    In [20]: c = [str(y) + str(x) for x, y in zip(a,b)]
    In [21]: c
    Out[21]: ['a0', 'b1', 'c2', 'd3', 'e4']
    

![](https://images.gitbook.cn/a2044b30-560f-11ea-91c3-41dcada8ff66)

### 5\. 打印键值对

    
    
    In [22]: a = {'a':1,'b':2,'c':3}
    In [23]: b = [k+ '=' + v for k, v in a.items()]
    In [24]: b = [k+ '=' + str(v) for k, v in a.items()]
    In [25]: b
    Out[25]: ['a=1', 'b=2', 'c=3']
    

![](https://images.gitbook.cn/bbf5fca0-560f-11ea-a714-09515afb96e2)

### 6\. 文件列表

    
    
    import os
    a = [d for d in os.listdir('D:/source/test')]
    

再结合 if，只查找出文件夹，如下找出三个文件夹：

    
    
    In [23]: import os
        ...: dirs = [d for d in os.listdir('D:/source/test') if os.path.isdir(d)]
    
    In [24]: dirs
    Out[24]: ['.vscode', 'templates', '__pycache__']
    

只查找出文件，如下找出 .sql、.py、.db 和 .json 文件。

    
    
    In [20]: import os
        ...: files = [d for d in os.listdir('D:/source/test') if os.path.isfile(d)]
    
    In [21]: files
    Out[21]:
    ['a.sql',
     'barchart.py',
     'barstack.py',
     'bar_numpy.py',
     'decorator.py',
     'flask_form_operate.py',
     'flask_hello.py',
     'intest.py',
     'pyecharts_server.py',
     'pytorchtest.py',
     'requests_started.py',
     'sqlite3_started.py',
     'test.db',
     'timlibe.json',
     'uniform_points.py']
    

### 7\. 转为小写

    
    
    In [34]: a = ['Hello', 'World', '2019Python']
    
    In [35]: [w.lower() for w in a]
    Out[35]: ['hello', 'world', '2019python']
    

但是以上写法可能会有问题，因为 Python 的列表类型可以不同，如果列表 a：

    
    
    In [25]: a = ['Hello', 'World',2020,'Python']
    
    In [26]: [w.lower() for w in a]
    ---------------------------------------------------------------------------
    AttributeError: 'int' object has no attribute 'lower'
    

如上就会出现 int 对象没有方法 lower 的问题，先转化元素为 str 后再操作：

    
    
    In [27]: [str(w).lower() for w in a]
    Out[27]: ['hello', 'world', '2020', 'python']
    

更友好的做法，使用 isinstance，判断元素是否为 str 类型，如果是，再调用 lower 做转化：

    
    
    In [34]: [w.lower() for w in a if isinstance(w,str) ]
    Out[34]: ['hello', 'world', 'python']
    

### 8\. 保留唯一值

    
    
    In [64]: def filter_non_unique(lst):
        ...:   return [item for item in lst if lst.count(item) == 1]
    
    In [65]: filter_non_unique([1, 2, 2, 3, 4, 4, 5])
    Out[65]: [1, 3, 5]
    

有了上面这些基础后，再来看几个难度大点的案例。

### 9\. 筛选分组

    
    
    In [36]: def bifurcate(lst, filter):
        ...:   return [
        ...:     [x for i,x in enumerate(lst) if filter[i] == True],
        ...:     [x for i,x in enumerate(lst) if filter[i] == False]
        ...:   ]
        ...:
    In [37]: bifurcate(['beep', 'boop', 'foo', 'bar'], [True, True, False, True])
    
    Out[37]: [['beep', 'boop', 'bar'], ['foo']]
    

![](https://images.gitbook.cn/48c4bd60-5610-11ea-afc1-9b7d03147fc6)

### 10\. 函数分组

    
    
    In [38]: def bifurcate_by(lst, fn):
        ...:   return [
        ...:     [x for x in lst if fn(x)],
        ...:     [x for x in lst if not fn(x)]
        ...:   ]
    
    In [39]: bifurcate_by(['Python3', 'up', 'users', 'people'], lambda x: x[0] == 'u')
    Out[39]: [['up', 'users'], ['Python3','people']]
    

![](https://images.gitbook.cn/5f355e60-5610-11ea-b227-d5a95658c855)

### 11\. 差集

    
    
    In [53]: def difference(a, b):
        ...:   _a, _b =set(a),set(b)
        ...:   return [item for item in _a if item not in _b]
    
    In [54]: difference([1,1,2,3,3], [1, 2, 4])
    Out[54]: [3]
    

![](https://images.gitbook.cn/12bb02a0-5611-11ea-908b-b5332759803b)

### 12\. 函数差集

列表 a、b 中元素经过 fn 映射后，返回在 a 不在 b 中的元素。

    
    
    In [14]: def difference_by(a, b, fn):
        ...:     _b = set(map(fn, b))
        ...:     return [item for item in a if fn(item) not in _b]
    

列表元素为单个元素：

    
    
    In [15]: from math import floor
    
    In [16]: difference_by([2.1, 1.2], [2.3, 3.4],floor)
    Out[16]: [1.2]
    

![](https://images.gitbook.cn/32e3bcc0-5611-11ea-9dfa-cf3c87b75c44)

列表元素为字典：

    
    
    In [63]: difference_by([{ 'x': 2 }, { 'x': 1 }], [{ 'x': 1 }], lambda v : v['x'])
    Out[63]: [{'x': 2}]
    

![](https://images.gitbook.cn/501f9020-5611-11ea-ac19-374315264b53)

### 小结

熟练操作以上 12 个例子，就会进一步强化对 Python 中有用的列表生成式的使用。

列表生成式常与 if、for、嵌套 for、map、lambda 等结合使用。

