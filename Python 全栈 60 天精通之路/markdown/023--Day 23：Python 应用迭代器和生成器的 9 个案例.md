### 列表和迭代器区别

有些读者朋友，区分不开列表、字典、集合等非迭代器对象与迭代器对象，觉得迭代器是多余的。

先探讨它们的区别。首先，创建一个列表 a：

    
    
    a = [1,3,5,7]
    

有没有朋友认为，列表就是迭代器的？注意列表 a 可不是迭代器类型（Iterator）。要想成为迭代器，需要经过内置函数 iter 包装：

    
    
    a_iter = iter(a)
    

此时 a_iter 就是 Iterator，迭代器。可以验证：

    
    
    In [21]: from collections.abc import Iterator
        ...: isinstance(a_iter,Iterator)
    Out[21]: True
    

分别遍历 a、a_iter：

    
    
    In [22]: for i in a:
        ...:     print(i)
        ...:
    1
    3
    5
    7
    
    In [23]: for i in a_iter:
        ...:     print(i)
        ...:
    1
    3
    5
    7
    

打印结果一样，但是，再次遍历 a、a_iter 就会不同，a 正常打印，a_iter 没有打印出任何信息：

    
    
    In [24]: for i in a:
        ...:     print(i)
        ...:
    1
    3
    5
    7
    
    In [25]: for i in a_iter:
        ...:     print(i)
        ...:
    

这就是列表 a 和迭代器 a_iter 的区别：

  * 列表不论遍历多少次，表头位置始终是第一个元素；
  * 迭代器遍历结束后，不再指向原来的表头位置，而是为最后元素的下一个位置。

只有迭代器对象才能与内置函数 next 结合使用，next 一次，迭代器就前进一次，指向一个新的元素。

所以，要想迭代器 a_iter 重新指向 a 的表头，需要重新创建一个新的迭代 a_iter_copy：

    
    
    In [27]: a_iter_copy = iter(a)
    

调用 next，输出迭代器指向 a 的第一个元素：

    
    
    In [28]: next(a_iter_copy)
    Out[28]: 1
    

值得注意，我们无法通过调用 len 获得迭代器的长度，只能迭代到最后一个末尾元素时，才知道其长度。

那么，怎么知道迭代到元素末尾呢？我们不妨一直 next，看看会发生什么：

    
    
    In [30]: next(a_iter_copy)
    Out[30]: 3
    
    In [31]: next(a_iter_copy)
    Out[31]: 5
    
    In [32]: next(a_iter_copy)
    Out[32]: 7
    
    In [33]: next(a_iter_copy)
    
    StopIteration:
    

等迭代到最后一个元素后，再执行 next，会触发 StopIteration 异常。

所以，通过捕获此异常，就能求出迭代器指向 a 的长度，如下：

    
    
    a = [1, 3, 5, 7]
    a_iter_copy2 = iter(a)
    iter_len = 0
    try:
        while True:
            i = next(a_iter_copy2)
            print(i)
            iter_len += 1
    except StopIteration:
        print('iterator stops')
    
    print('length of iterator is %d' % (iter_len,))
    

打印结果：

    
    
    1
    3
    5
    7
    iterator stops
    length of iterator is 4
    

以上总结：遍历列表，表头位置始终不变；遍历迭代器，表头位置相应改变；next 函数执行一次，迭代对象指向就前进一次；StopIteration
触发时，意味着已到迭代器尾部。

认识到迭代器和列表等区别后，我们再来说说生成器。

带 yield
的函数是生成器，而生成器也是一种迭代器。所以，生成器也有上面那些迭代器的特点。前些天已经讨论过生成器的一些基本知识，今天主要讨论生成器带来哪些好处，实际的使用场景在哪里。

### 节省内存案例

求一组数据累积乘，比如：三个数 [1,2,3]，累积乘后返回 [1,2,6]。

一般的方法：

    
    
    def accumulate_div(a):
        if a is None or len(a) == 0:
            return []
        rtn = [a[0]] 
        for i in a[1:]:
            rtn.append(i*rtn[-1])
        return rtn
    
    rtn = accumulate_div([1, 2, 3, 4, 5, 6])
    print(rtn) 
    

打印结果：

    
    
    [1, 2, 6, 24, 120, 720]
    

这个方法开辟一段新内存 rtn，空间复杂度为 O(n)。

更节省内存的写法：

    
    
    def accumulate_div(a):
        if a is None or len(a) == 0:
            return []
        it = iter(a)
        total = next(it)
        yield total
        for i in it:
            total = total * i
            yield total
    

调用生成器函数 accumulate_div，结合 for，输出结果：

    
    
    acc = accumulate_div([1, 2, 3, 4, 5, 6])
    for i in acc:
        print(i)
    

打印结果：

    
    
    1
    2
    6
    24
    120
    720
    

也可以，直接转化为 list，如下：

    
    
    rtn = list(accumulate_div([1, 2, 3, 4, 5, 6]))
    print(rtn)
    

打印结果：

    
    
    [1, 2, 6, 24, 120, 720]
    

这种使用 yield 生成器函数的方法，占用内存空间为 O(1)。

当输入的数组 [1, 2, 3, 4, 5, 6]，只有 6 个元素时，这种内存浪费可以忽视，但是当处理几个 G
的数据时，这种内存空间的浪费就是致命的，尤其对于单机处理。

所以，要多学会使用 yield，写出更加节省内存的代码，这是真正迈向 Python 中高阶的必经之路。

Python 内置的 itertools  模块就是很好的使用 yield 生成器的案例，今天我们再来学习几个。

### 拼接迭代器

chain 函数实现元素拼接，原型如下，参数 `*` 表示可变的参数：

    
    
    chain(*iterables)
    

应用如下：

    
    
    In [9]: from itertools import *
    
    In [10]: chain_iterator = chain(['I','love'],['python'],['very', 'much'])
    
    In [11]: for i in chain_iterator:
        ...:     print(i)
    
    I
    love
    python
    very
    much
    

它有些 join 串联字符串的感觉，join 只是一次串联一个序列对象。而 chain 能串联多个可迭代对象，形成一个更大的可迭代对象。

查看函数返回值 chain_iterator，它是一个迭代器（Iterator）。

    
    
    In [6]: from collections.abc import Iterator
    In [9]: isinstance(chain_iterator,Iterator)
    Out[9]: True    
    

chain 如何做到高效节省内存？chain 主要实现代码如下：

    
    
    def chain(*iterables):
        for it in iterables:
            for element in it:
                yield element
    

chain 是一个生成器函数，在迭代时，每次吐出一个元素，所以做到最高效的节省内存。

### 累积迭代器

返回可迭代对象的累积迭代器，函数原型：

    
    
    accumulate(iterable[, func, *, initial=None])
    

应用如下，返回的是迭代器，通过结合 for 打印出来。

如果 func 不提供，默认求累积和：

    
    
    In [15]: accu_iterator = accumulate([1,2,3,4,5,6])
    
    In [16]: for i in accu_iterator:
        ...:     print(i)
    1
    3
    6
    10
    15
    21
    

如果 func 提供，func 的参数个数要求为 2，根据 func 的累积行为返回结果。

    
    
    In [13]: accu_iterator = accumulate([1,2,3,4,5,6],lambda x,y: x*y)
    
    In [14]: for i in accu_iterator:
        ...:     print(i)
    1
    2
    6
    24
    120
    720
    

accumulate 主要的实现代码，如下：

    
    
    def accumulate(iterable, func=operator.add, *, initial=None):
        it = iter(iterable)
        total = initial
        if initial is None:
            try:
                total = next(it)
            except StopIteration:
                return
        yield total
        for element in it:
            total = func(total, element)
            yield total
    

accumulate 函数工作过程如下：

  1. 包装 iterable 为迭代器；
  2. 初始值 initial 很重要；

  * 如果它的初始值为 None，迭代器向前移动求出下一个元素，并赋值给 total，然后 yield；
  * 如果初始值被赋值，直接 yield。

  1. 此时迭代器 it 已经指向 iterable 的第二个元素，遍历迭代器 it，`func(total, element)` 后，求出 total 的第二个取值，yield 后，得到返回结果的第二个元素。
  2. 直到 it 迭代结束。

### 漏斗迭代器

compress 函数，功能类似于漏斗功能，所以称它为漏斗迭代器，原型：

    
    
    compress(data, selectors)
    

经过 selectors 过滤后，返回一个更小的迭代器。

    
    
    In [20]: compress_iter = compress('abcdefg',[1,1,0,1])
    
    In [21]: for i in compress_iter:
        ...:     print(i)
    
    a
    b
    d
    

compress 返回元素个数，等于两个参数中较短序列的长度。

它的主要实现代码，如下：

    
    
    def compress(data, selectors):
        return (d for d, s in zip(data, selectors) if s)
    

### drop 迭代器

扫描可迭代对象 iterable，从不满足条件处往后全部保留，返回一个更小的迭代器。

原型如下：

    
    
    dropwhile(predicate, iterable)
    

应用举例：

    
    
    In [48]: drop_iterator = dropwhile(lambda x: x<3,[1,0,2,4,1,1,3,5,-5])
    
    In [49]: for i in drop_iterator:
        ...:     print(i)
        ...:
    4
    1
    1
    3
    5
    -5
    

主要实现逻辑，如下：

    
    
    def dropwhile(predicate, iterable):
        iterable = iter(iterable)
        for x in iterable:
            if not predicate(x):
                yield x
                break
        for x in iterable:
            yield x
    

函数工作过程，如下：

  * iterable 包装为迭代器
  * 迭代 iterable
    * 如果不满足条件 predicate，`yield x`，然后跳出迭代，迭代完 iterable 剩余所有元素。
    * 如果满足条件 predicate，就继续迭代，如果所有都满足，则返回空的迭代器。

### take 迭代器

扫描列表，只要满足条件就从可迭代对象中返回元素，直到不满足条件为止，原型如下：

    
    
    takewhile(predicate, iterable)
    

应用举例：

    
    
    In [50]: take_iterator = takewhile(lambda x: x<5, [1,4,6,4,1])
    
    In [51]: for i in take_iterator:
        ...:     print(i)
        ...:
    1
    4
    

主要实现代码，如下：

    
    
    def takewhile(predicate, iterable):
        for x in iterable:
            if predicate(x):
                yield x
            else:
                break #立即返回
    

函数工作过程：

  * 遍历 iterable
    * 符合条件 predicate，`yield x`
    * 否则跳出循环

### 克隆迭代器

tee 实现对原迭代器的复制，原型如下：

    
    
    tee(iterable, n=2)
    

应用如下，克隆出 2 个迭代器，以元组结构返回：

    
    
    In [52]: a = tee([1,4,6,4,1],2)
    
    In [53]: a
    Out[53]: (<itertools._tee at 0x223ec30cfc8>, <itertools._tee at 0x223ec30c548>)
    
    In [54]: a[0]
    Out[54]: <itertools._tee at 0x223ec30cfc8>
    
    In [55]: a[1]
    Out[55]: <itertools._tee at 0x223ec30c548>
    

并且，复制出的两个迭代器，相互独立，如下，迭代器 a[0] 向前迭代一次：

    
    
    In [56]: next(a[0])
    Out[56]: 1
    

克隆出的迭代器 a[1] 向前迭代一次，还是输出元素 1，表明它们之间是相互独立的：

    
    
    In [57]: next(a[1])
    Out[57]: 1
    

这种应用场景，需要用到迭代器至少两次的场合，一次迭代器用完后，再使用另一个克隆出的迭代器。

实现它的主要逻辑，如下：

    
    
    from collections import deque
    
    def tee(iterable, n=2):
        it = iter(iterable)
        deques = [deque() for i in range(n)]
        def gen(mydeque):
            while True:
                if not mydeque:            
                    try:
                        newval = next(it)   
                    except StopIteration:
                        return
                    for d in deques:     
                        d.append(newval)
                yield mydeque.popleft()
        return tuple(gen(d) for d in deques)
    

### 复制元素

repeat 实现复制元素 n 次，原型如下：

    
    
    repeat(object[, times])
    

应用如下：

    
    
    In [66]: list(repeat(6,3))
    Out[66]: [6, 6, 6]
    
    In [67]: list(repeat([1,2,3],2))
    Out[67]: [[1, 2, 3], [1, 2, 3]]
    

它的主要实现逻辑，如下，有趣的是 repeat 函数如果 times 不设置，将会一直 repeat 下去。

    
    
    def repeat(object, times=None):
        if times is None:
            while True: 
                yield object
        else:
            for i in range(times):
                yield object
    

### 笛卡尔积

笛卡尔积实现的效果，同下：

    
    
     ((x,y) for x in A for y in B)
    

举例，如下：

    
    
    In [68]: list(product('ABCD', 'xy'))
    Out[68]:
    [('A', 'x'),
     ('A', 'y'),
     ('B', 'x'),
     ('B', 'y'),
     ('C', 'x'),
     ('C', 'y'),
     ('D', 'x'),
     ('D', 'y')]
    

它的主要实现代码，包括：

    
    
    def product(*args, repeat=1):
        pools = [tuple(pool) for pool in args] * repeat
        result = [[]]
        for pool in pools:
            result = [x+[y] for x in result for y in pool]
        for prod in result:
            yield tuple(prod)
    

### 加强版 zip

若可迭代对象的长度未对齐，将根据 fillvalue 填充缺失值，返回结果的长度等于更长的序列长度。

    
    
    In [69]: list(zip_longest('ABCD', 'xy', fillvalue='-'))
    Out[69]: [('A', 'x'), ('B', 'y'), ('C', '-'), ('D', '-')]
    

它的主要实现逻辑：

    
    
    def zip_longest(*args, fillvalue=None):
        iterators = [iter(it) for it in args]
        num_active = len(iterators)
        if not num_active:
            return
        while True:
            values = []
            for i, it in enumerate(iterators):
                try:
                    value = next(it)
                except StopIteration:
                    num_active -= 1
                    if not num_active:
                        return
                    iterators[i] = repeat(fillvalue)
                    value = fillvalue
                values.append(value)
            yield tuple(values)
    

它里面使用 repeat，也就是在可迭代对象的长度未对齐时，根据 fillvalue 填充缺失值。

### 小结

今天，一起与大家学习了 Python 进阶重要的知识和案例，包括：

  * 分清列表和迭代器的区别，进而弄明白迭代器的基本知识
  * 生成器节省内存的一个案例
  * 内置模块 itertools 中使用生成器的 9 个节省内存的案例，进一步帮助大家加深生成器的应用。

值得注意，生成器也是一种迭代器，输出结果需要结合 for 或 next 和捕获 StopIteration。

