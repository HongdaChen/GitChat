### 1\. 一行代码生成 [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]

使用列表生成式，创建列表，观察元素出现规律，可得出如下代码：

    
    
    In [97]: a = [2*i+1 for i in range(10)]
    
    In [98]: a
    Out[98]: [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
    

### 2\. 写一个等差数列

产生一个首项为 10，公差为 12，末项不大于 100 的列表。

使用列表生成式创建：

    
    
    In [1]: a = list(range(10,100, 12))
    
    In [2]: a
    Out[2]: [10, 22, 34, 46, 58, 70, 82, 94]
    

### 3\. 一行代码求 1 到 10000 内整数和

提供两种方法：

1\. 使用 Python 内置函数 sum 求和：

    
    
    In [99]: s = sum(range(10000))
    
    In [100]: s
    Out[100]: 49995000
    

2\. 使用 functools 模块中的 reduce 求和：

    
    
    In [101]: s = reduce(lambda x,y: x+y, range(10000))
    
    In [102]: s
    Out[102]: 49995000
    

### 4\. 打乱一个列表

使用 random 模块，shuffle 函数打乱原来列表，值得注意是 in-place 打乱。

    
    
    import random
    a = range(10)
    random.shuffle(a)
    print(a) # [4, 6, 1, 2, 5, 3, 9, 0, 8, 7]
    

### 5\. 字典按 value 排序并返回新字典？

原字典：

    
    
    d= {'a':12,'b':50,'c':1,'d':20} 
    

使用 Python 内置函数 sorted 排序：

    
    
    In [10]: d = dict(sorted(d.items(),key=lambda item: item[1]))
    
    In [11]: d
    Out[11]: {'c': 1, 'a': 12, 'd': 20, 'b': 50} 
    

### 6\. 如何删除 list 里重复元素，并保证元素顺序不变

给定列表：

    
    
    a = [3,2,2,2,1,3]
    

如果只是删除重复元素，直接使用内置 set 函数，去重，但是不能保证原来元素顺序。

不要这么做，列表删除某个元素后，后面的元素整体会向前移动。

    
    
    def del_duplicated(a):
        ac = a.copy()
        b = []
        for index, i in enumerate(ac):
            if i in b:
                del ac[index]
            else:
                b.append(i)
        return ac
    
    In [18]: r = del_duplicated(a)
    
    In [19]: r
    Out[19]: [3, 2, 2, 1] # wrong      
    

正确做法：

    
    
    def del_duplicated(a):
        b = []
        for i in a:
            if i not in b:
                b.append(i)
        return b
    

### 7\. 怎么找出两个列表的相同元素和不同元素？

给定列表 `a = [3,2,2,2,1,3]`，列表 `b = [1,4,3,4,5]`，使用集合，找出相同元素：

    
    
    def ana(a,b):
        aset, bset = set(a), set(b)
        same = aset.intersection(bset)
        differ = aset.difference(bset).union(bset.difference(aset))
        return same, differ
    
    In [28]: ana(a,b)
    Out[28]: ({1, 3}, {2, 4, 5})
    

### 8\. 字符串处理成字典

输入串 `"k0:10|k1:2|k2:11|k3:5"`，输出字典 `{k0:10,k1:2,...}`。

  * 第一层 split，根据分隔符 `|`，分割出 `k0:10,k1:2,k2:11,k3:5`
  * 第二层 split，根据分隔符 `:`，分割出新字典的键值对

使用字典生成式，得到结果，也就是一个新字典：

    
    
    In [30]: m = map(lambda x: x.split(':'),'k0:10|k1:2|k2:11|k3:5'.split('|'))
    
    In [31]: {mi[0]:int(mi[1]) for mi in m}
    Out[31]: {'k0': 10, 'k1': 2, 'k2': 11, 'k3': 5}
    

### 9\. 输入日期，判断这一天是这一年的第几天？

使用 datetime 模块，提取日期 date 对象，调用 timetuple() 方法，返回一个 struct_time 对象，属性 tm_yday
便是这一年的第几天：

    
    
    from datetime import datetime
    
    def get_day_of_year(y,m,d):
        return datetime(y,m,d).date().timetuple().tm_yday
    
    In [45]: get_day_of_year(2020,2,1)
    Out[45]: 32
    In [46]: get_day_of_year(2019,12,31)
    Out[46]: 365
    

### 10\. 遍历目录与子目录，抓取 .py 文件

os 模块、walk 方法实现递归遍历所有文件，os.path.splitext 返回文件的名字和扩展名，如果扩展名匹配到 ext，则添加到 res 中。

    
    
    import os
    
    def get_files(directory,ext):
        res = []
        for root,dirs,files in os.walk(directory):
            for filename in files:
                name,suf = os.path.splitext(filename)
                if suf == ext:
                    res.append(os.path.join(root,filename))
        return res
    
    get_files('D:/source/python-zhuanlan','.py')
    

### 11\. 单机 4G 内存，处理 10G 文件的方法？

假定可以单独处理一行数据，行间数据相关性为零。

方法一：仅使用 Python 内置模板，逐行读取到内存。

使用 yield，好处是 **解耦读取操作和处理操作** ：

    
    
    def python_read(filename):
        with open(filename,'r',encoding='utf-8') as f:
            for line in f:
                yield line
    

以上每次读取一行，逐行迭代，逐行处理数据：

    
    
    if __name__ == '__main__':
        g = python_read('./data/movies.dat')
        for c in g:
            print(c)
            # process c
    

方法二：方法一有缺点，逐行读入，频繁的 IO 操作拖累处理效率。是否有一次 IO，读取多行的方法？

Pandas 包 read_csv 函数，参数有 38 个之多，功能非常强大。

关于单机处理大文件，read_csv 的 chunksize 参数能做到，设置为 5，意味着一次读取 5 行。

    
    
    def pandas_read(filename,sep=',',chunksize=5):
        reader = pd.read_csv(filename,sep=sep,chunksize=chunksize)
        while True:
            try:
                yield reader.get_chunk()
            except StopIteration:
                print('---Done---')
                break
    

使用如同方法一：

    
    
    if __name__ == '__main__':
        g = pandas_read('./data/movies.dat',sep="::")
        for c in g:
            print(c)
            # process c
    

以上就是单机处理大文件的两个方法，推荐使用方法二，更加灵活。

### 12\. 统计一个文本中单词频次最高的 10 个单词

使用 yield 解耦数据读取 python_read 和数据处理 process

  * python_read：逐行读入
  * process：正则替换掉空字符，并使用空格，分隔字符串，保存到 defaultdict 对象中。

    
    
    from collections import Counter, defaultdict
    import re
    
    def python_read(filename):
        with open(filename, 'r', encoding='utf-8') as f:
            for line in f:
                yield line
    
    d = defaultdict(int)
    
    def process(line):
        for word in re.sub('\W+', " ", line).split():
            d[word] += 1
    

使用两个函数，最后，使用 Counter 类统计出频次最高的 10 个单词：

    
    
    for line in python_read('write_file.py'):
        process(line)
    
    most10 = Counter(d).most_common(10)
    print(most10)
    

### 13\. 反转一个整数，例如 -12345 --> -54321

  * 如果 x 位于 (-10,10) 间，直接返回；
  * 然后，将 x 转换为字符串对象 sx；
  * 如果 x 是负数，截取 sx[1:]，并反转字符串；
  * 如果 x 是正数，直接反转字符串；
  * 最后使用内置函数 int() 转化为整数。

    
    
    def reverse_int(x: int):
        if -10 < x < 10:
            return x
        sx = str(x)
    
        def reverse_str(sx):
            return sx[::-1]
    
        if sx[0] == "-":
            sx = reverse_str(sx[1:])
            x = int(sx)
            return -x
        sx = reverse_str(sx)
        return int(sx)
    

### 14\. 以下代码输出结果

此题需要注意，内嵌函数 foo 使用的两个变量 i 和 x，其中 x 为其形参，i 为 enclosing 域内定义的变量。

rtn 添加三个函数 foo，但是并未发生调用。

    
    
    def f():
        i = 0
        def foo(x):
            return i*x
        rtn = []
        while i < 3:
            rtn.append(foo)
            i += 1
        return rtn
    
    # 调用函数 f
    for fs in f():
        print(fs(10))
    

直到执行 fs(10) 时，内嵌函数 foo 才被调用，但是此时的 enclosing 变量 i 取值为 3：

    
    
    for fs in f():
        print(fs(10))
    

所以输出结果为：

    
    
    30
    30
    30
    

### 15\. 如下函数 foo 的调用哪些是正确的？

    
    
    def foo(filename,a=0,b=1,c=2):
        print('filename: %s \n c: %d'%(filename,c))
    

已知 filename 为 '.'，c 为 10，正确为 foo 函数传参的方法，以下哪些是对的，哪些是错误的？

    
    
    A foo('.', 10)
    
    B foo('.', 0,1,10)
    
    C foo('.',0,1,c=10)
    
    D foo('.',a=0,1,10)
    
    E foo(filename='.', c=10)
    
    F foo('.', c=10)
    

分析：

    
    
    A 错误，a 被赋值为 10
    B 正确，c 是位置参数
    C 正确，c 是关键字参数
    D 错误，位置参数不能位于关键字参数后面
    E 正确，filename 和 c 都是关键字参数
    F 正确，filename 位置参数，c 是关键字参数
    

验证测试：

    
    
    In [58]: foo('.', 10)
    filename: .
     c: 2
    
    In [59]: foo('.', 0,1,10)
    filename: .
     c: 10
    
    In [60]: foo('.',0,1,c=10)
    filename: .
     c: 10
    
    In [61]: foo('.',a=0,1,10)
      File "<ipython-input-61-e3909182c523>", line 1
        foo('.',a=0,1,10)
                   ^
    SyntaxError: positional argument follows keyword argument
    
    
    In [62]: foo(filename='.', c=10)
    filename: .
     c: 10
    
    In [63]: foo('.', c=10)
    filename: .
     c: 10
    

### 16\. lambda 函数的形参和返回值

key 值为 lambda 函数，说说 lambda 函数的形参和返回值。

    
    
    def longer(*s):
        return max(*s, key=lambda x: len(x))
    
    longer({1,3,5,7},{1,5,7},{2,4,6,7,8})
    

lambda 函数的形参：s 解包后的元素值，可能取值为：{1,3,5,7}、{1,5,7}、{2,4,6,7,8} 三种。

lambda 函数的返回值为：元素的长度，可能取值为：{1,3,5,7}、{1,5,7}、{2,4,6,7,8} 的长度 4,3,5。

### 17\. 正则匹配负整数

匹配所有负整数，不包括 0。正则表达式：`^-[1-9]\d*$`

  * `^-` 表示字符串以 `-`开头
  * `[1-9]` 表示数字 1 到 9， **注意** 不要写成 `\d`，因为负整数没有以 -0 开头的
  * `\d*` 表示数字 0 到 9 出现 0 次、1 次或多次
  * `$` 表示字符串以数字结尾

以上分布讲解的示意图，如下所示：

![image-20200301170709417](https://images.gitbook.cn/6e1aa380-5bb1-11ea-8155-9d6d04886d5b)

测试字符串：

    
    
    import re
    s = ['-1','-15756','9','-01','10','-']
    pat = r'^-[1-9]\d*$'
    rec = re.compile(pat)
    rs = [i for i in s if rec.match(i)]
    print(rs)
    # 结果
    # ['-1', '-15756']
    

### 18\. 正则匹配负浮点数

正确写出匹配负浮点数的正则表达式，要先思考分析。

考虑到两个实例：-0.12、-111.234，就必须要分为两种情况。

适应实例 -0.12 的正则表达式：`^-0.\d*[1-9]\d*$`，注意要考虑到 -0.0000 这种非浮点数，因此正则表达式必须这样写。

不要想当然地写为：`^-0.\d*$`，或者 `^-0.\d*[1-9]*$`，或者 `^-0.\d*[0-9]*$`，这些都是错误的！

适应实例 -111.234 的正则表达式：`^-[1-9]\d*.\d*$`，使用 `|`，综合两种情况，故正则表达式为：

    
    
    ^-[1-9]\d*\.\d*|-0\.\d*[1-9]\d*$
    

![image-20200301172418055](https://images.gitbook.cn/7e220840-5bb1-11ea-a695-8f4c079b036d)

测试字符串：

    
    
    import re
    s = ['-1','-1.5756','-0.00','-000.1','-1000.10']
    pat = r'^-[1-9]\d*\.\d*|-0\.\d*[1-9]\d*$'
    rec = re.compile(pat)
    rs = [i for i in s if rec.match(i)]
    print(rs)
    # 结果
    # ['-1.5756', '-1000.10']
    

### 19\. 使用 filter() 求出列表中大于 10 的元素

filter 函数使用 lambda 函数，找出满足大于 10 的元素。

    
    
    a = [15,2,7,20,400,10,9,-15,107]
    al = list(filter(lambda x: x > 10, a))
    In [74]: al
    Out[74]: [15, 20, 400, 107]
    

### 20\. 说说下面 map 函数的输出结果

map 函数当含有多个列表时，返回长度为最短列表的长度；

lambda 函数的形参个数等于后面列表的个数。

    
    
    m = map(lambda x,y: min(x,y), [5, 1, 3, 4], [3,4,3,2,1]) 
    
    print(list(m))
    

结果为：

    
    
    [3, 1, 3, 2]
    

### 21\. 说说 reduce 函数的输出结果

reduce 实现对列表的归约化简，规则如下：

    
    
    f(x,y) = x*y + 1
    

因此，下面归约的过程为：

    
    
    f(1,2) = 3
    
    f(3,3) = 3*3 + 1 = 10
    
    f(10,4) = 10*4 + 1 = 41
    
    f(41,5) = 41*5 + 1 = 206
    
    
    
    from functools import reduce
    
    reduce(lambda x,y: x*y+1,[1,2,3,4,5])
    

结果为：

    
    
    206
    

### 22\. x = (i for i in range(5))，x 是什么类型

x 是生成器类型，与 for 等迭代，输出迭代结果：

    
    
    x = (i for i in range(5))
    for i in x:
        print(i)
    

结果为：

    
    
    0
    1
    2
    3
    4
    

### 23\. 可变类型和不可变类型分别列举 3 个

  * 可变类型：mutable type，常见的有：list、dict、set、deque 等
  * 不可变类型：immutable type，常见的有：int、float、str、tuple、frozenset 等

只有不可变类型才能作为字典等的键。

### 24\. is 和 == 有什么区别？

  * is 用来判断两个对象的标识号是否相等；
  * == 用于判断值或内容是否相等，默认是基于两个对象的标识号比较。

也就是说，如果 `a is b` 为 True 且如果按照默认行为，意味着 `a==b` 也为 True。

### 25\. 写一个学生类 Student

添加一个属性 id，并实现若 id 相等，则认为是同一位同学的功能。

重写 __eq__ 方法，若 id 相等，返回 True。

    
    
    class Student:
        def __init__(self,id,name):
            self.id = id
            self.name = name
        def __eq__(self,student):
            return self.id == student.id
    

判断两个 Student 对象，`==` 的取值：

    
    
    s1 = Student(10,'xiaoming')
    s2 = Student(20,'xiaohong')
    s3 = Student(10,'xiaoming2')
    
    In [85]: s1 == s2
    Out[85]: False
    
    In [86]: s1 == s3
    Out[86]: True
    

### 26\. 有什么方法获取类的所有属性和方法？

获取下面类 Student 的所有属性和方法，使用 dir() 内置函数。

    
    
    class Student:
        def __init__(self,id,name):
            self.id = id
            self.name = name
        def __eq__(self,student):
            return self.id == student.id
    

获取类上的所有属性和方法：

    
    
    In [87]: dir(Student)
    Out[87]:
    ['class',
     'delattr',
     'dict',
     'dir',
     'doc',
     'eq',
     'format',
     'ge',
     'getattribute',
     'gt',
     'hash',
     'init',
     'init_subclass',
     'le',
     'lt',
     'module',
     'ne',
     'new',
     'reduce',
     'reduce_ex',
     'repr',
     'setattr',
     'sizeof',
     'str',
     'subclasshook',
     'weakref']
    

获取实例上的属性和方法：

    
    
    s1 = Student(10,'xiaoming')
    In [88]: dir(s1)
    Out[88]:
    ['class',
     'delattr',
     'dict',
     'dir',
     'doc',
     'eq',
     'format',
     'ge',
     'getattribute',
     'gt',
     'hash',
     'init',
     'init_subclass',
     'le',
     'lt',
     'module',
     'ne',
     'new',
     'reduce',
     'reduce_ex',
     'repr',
     'setattr',
     'sizeof',
     'str',
     'subclasshook',
     'weakref',
     'id',
     'name']
    

### 27\. Python 中如何动态获取和设置对象的属性？

如下 Student 类：

    
    
    class Student:
        def __init__(self,id,name):
            self.id = id
            self.name = name
        def __eq__(self,student):
            return self.id == student.id
    

Python 使用 hasattr 方法，判断实例是否有属性 x：

    
    
    s1 = Student(10,'xiaoming')
    In [93]: hasattr(s1,'id')
    Out[93]: True
    In [94]: hasattr(s1,'address')
    Out[94]: False 
    

使用 setattr 动态添加对象的属性，函数原型：

    
    
    <function setattr(obj, name, value, /)>
    

为类对象 Student 添加属性：

    
    
    if not hasattr(Student, 'address'):
        setattr(Student,'address','beijing')
        print(hasattr(s1,'address'))
    

### 28\. 实现一个按照 2*i+1 自增的迭代器

实现类 AutoIncrease，继承于 Iterator 对象，重写两个方法：

  * __iter__
  * __next__

    
    
    from collections.abc import Iterator
    
    class AutoIncrease(Iterator):
        def __init__(self, init, n):
            self.init = init
            self.n = n
            self.__cal = 0
    
        def __iter__(self):
            return self
    
        def __next__(self):
            if self.__cal == 0:
                self.__cal += 1
                return self.init
            while self.__cal < self.n:
                self.init *= 2
                self.init += 1
                self.__cal += 1
                return self.init
            raise StopIteration
    

调用递减迭代器 Decrease：

    
    
    iter = AutoIncrease(1,10)
    for i in iter:
        print(i)
    

打印结果：

    
    
    1
    3
    7
    15
    31
    63
    127
    255
    511
    1023
    

### 29\. 实现文件按行读取和操作数据分离功能

使用 yield 解耦按行读取和操作数据的两步操作：

    
    
    def read_line(filename):
        with open(filename, 'r', encoding='utf-8') as f:
            for line in f:
                yield line
    
    def process_line(line:str):
        pass
    
    for line in read_line('.'):
        process_line(line)
    

### 30\. 使用 Python 锁避免脏数据出现的例子

使用多线程编程，会出现同时修改一个全局变量的情况，创建一把锁 locka：

    
    
    import threading
    import time
    
    locka = threading.Lock()
    a = 0
    def add1():
        global a
        try:
            locka.acquire() # 获得锁
            tmp = a + 1
            time.sleep(0.2) # 模拟 IO 操作
            a = tmp
        finally:
            locka.release() # 释放锁
        print('%s  adds a to 1: %d'%(threading.current_thread().getName(),a))
    
    threads = [threading.Thread(name='t%d'%(i,),target=add1) for i in range(10)]
    [t.start() for t in threads]
    

通过 locka.acquire() 获得锁，通过 locka.release() 释放锁。获得锁和释放锁之间的代码，只能单线程执行。

执行结果，如下：

    
    
    t0  adds a to 1: 1
    t1  adds a to 1: 2
    t2  adds a to 1: 3
    t3  adds a to 1: 4
    t4  adds a to 1: 5
    t5  adds a to 1: 6
    t6  adds a to 1: 7
    t7  adds a to 1: 8
    t8  adds a to 1: 9
    t9  adds a to 1: 10
    

多线程的代码，由于避免脏数据的出现，基本退化为单线程代码，执行效率被拖累。

### 31\. 说说死锁、GIL 锁、协程

多个子线程在系统资源竞争时，都在等待对方解除占用状态。

比如，线程 A 等待着线程 B 释放锁 b，同时，线程 B 等待着线程 A 释放锁 a。在这种局面下，线程 A 和线程 B
都相互等待着，无法执行下去，这就是死锁。

为了避免死锁发生，Cython 使用 GIL 锁，确保同一时刻只有一个线程在执行，所以其实是伪多线程。

所以，Python
里常常使用协程技术来代替多线程。多进程、多线程的切换是由系统决定，而协程由我们自己决定。协程无需使用锁，也就不会发生死锁。同时，利用协程的协作特点，高效的完成了原编程模型只能通过多个线程才能完成的任务。

### 小结

今天，一口气，与大家一起练习了 30 道 Python 高频面试题，如果大家一天一天好好学习专栏，坚持到现在的话，一定能解决这 30 道练习题。

与此同时，大家通过练习这些题目，进一步打牢 Python 基础，提升 Python 水平到一个新的层次，恭喜大家！这是你们辛勤付出的结果。

