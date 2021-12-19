Python 自带许多很好的模块（libraries），能够非常方便的解决一些实际问题。

而选择对合适的函数，使用好某些模块，能帮助我们少些很多行代码。

今天，与大家一起学习一个非常有用的模块——collections。

这个模块，提供与容器相关的、更高性能的数据类型，它们比通用容器 dict、list、set 和 tuple 更强大。

主要介绍，collections 模块常用的 3 种数据类型。

### NamedTuple

对于数据分析或机器学习领域，用好 NamedTuples 会写出可读性更强、更易于维护的代码。

大家回忆这种熟悉的场景。

> 你正在做特征工程，比较偏爱 list，所以把特征都放到一个 list 中。
>
> 然后，喂到机器学习模型中。
>
> 很快，你将会意识到数百个特征位于此 list 中，这就是事情变糟糕的开始。

如下，把如下 14 个特征，装到一个 list 类型的变量 features 中：

    
    
    In [5]: features = ['id','age','height','name','address','province','city','town','country','birth_address','father_name', 'monther_name','telephone','emergency_telephone']
    

假设，我负责维护某乡村，上千行居民信息。

现在有个新任务，刚刚调查过户口，有了一份新数据，现在要比较下，哪些居民的居住地址（对应字段 address）、联系电话（对应字段
telephone）、出生地信息（对应字段 birth address）发生了变化，统计出这些居民。

先确定，三个字段在 features
中的索引；然后，导入老数据，刚统计的居民数据；比较三个字段，只要一个有不同，就认为有变化，并装入到信息变化的居民列表中。

很快，写出下面的代码：

    
    
    def update_persons_info(old_data,new_data):
        # 假定老数据，新数据 id 按照行都卡对好。
        changed_list = []
        for line in new_data:
            new_props = line.split() # 使用空格分隔新版行数据
            for old in old_data: 
                old_props =  old.split() # 使用空格分隔老版行数据
                if old_props[11] != new_props[10]: # 假定新版数据中 telephone 的索引为 10；
                    changed_list.append(old.props[11])
                elif old_props[4] != new_props[6]: # 假定新版数据中 address 的索引为 6；
                    changed_list.append(old.props[11])
                elif old_props[9] != new_props[3]: # 假定新版数据中 birth_address 的索引为 3；
                    changed_list.append(old.props[11])
        return changed_list
    

以上代码，出现整数索引 0、3、4、6、10、11，代码可读性比较差，如果没有注释，可能日后我都不知道这些索引代表什么意思。

但是，如果使用 NamedTuple 去处理，乱为一团的事情，将会迅速变得井然有序。

再也不会为一系列的整数索引而犯愁！

先了解 NamedTuple 的基本使用，导入 NamedTuple 包，如下：

    
    
    In [14]: from collections import namedtuple
    

创建一个带有 14 个属性，名字为 Person 的 NamedTuple 实例 Person，如下：

    
    
    In [18]: Person = namedtuple('Person',['id','age','height','name','address','province','city','town','country','birth_address','father_name', 'monther_name','telephone','emergency_telephone'])
    

如下，调用实例 Person，创建一个 id=3 的 Person 对象：

    
    
    In [29]: a = ['']*11
    
    In [30]: Person(3,19,'xiaoming',*a)
    Out[30]: Person(id=3, age=19, height='xiaoming', name='', address='', province='', city='', town='', country='', birth_address='', father_name='', monther_name='', telephone='', emergency_telephone='')
    

使用 NamedTuple 重新改写上面的任务：

    
    
    def update_persons_info(old_data,new_data):
        changed_list = []
        for line in new_data:
            new_props = line.split() 
            new_person = Person(new_props) # new_props 与 Person 参数卡对好
            for old in old_data: 
                old_props =  old.split() 
                old_person = Person(old_props)
                if old_person.id != new_person.id: 
                    changed_list.append(old_person.id)
                elif old_person.address != new_person.address:
                    changed_list.append(old_person.address)
                elif old_person.birth_address != new_person.birth_address: 
                    changed_list.append(old_person.birth_address)
        return changed_list
    

效果对比明显，改后的代码，3 处条件比较地方，没有用到一个整数索引，提高了代码可读性。

同时，也增强了代码的可维护性。当新导入的文件，特征列的顺序与原来不一致时，无需改动那 3 处条件比较之处，但是原来版本就必须要修改，相对更繁琐，不好被维护。

以上所述，NamedTuple 优点明显，但是同样缺点也较为明显，一个 NamedTuple 创建后，它的属性取值不允许被修改，也就是属性只能是可读的。

如下，xiaoming 一旦创建后，所有属性都不允许被修改。

    
    
    In [33]: Person = namedtuple('Person',['id','age','height','name','address','province','ci
        ...: ty','town','country','birth_address','father_name', 'monther_name','telephone','e
        ...: mergency_telephone'])
    
    In [34]:  a = ['']*11
    
    In [35]: xiaoming = Person(3,19,'xiaoming',*a)
    
    In [36]: xiaoming.age = 20
    ---------------------------------------------------------------------------
    AttributeError                            Traceback (most recent call last)
    <ipython-input-36-14208077a3a9> in <module>
    ----> 1 xiaoming.age = 20
    
    AttributeError: can't set attribute
    

### Counter

Counter 正如名字那样，它的主要功能就是计数。我们在分析数据时，基本都会涉及计数，真的家常便饭。

习惯使用 list 的朋友，往往会这样统计：

    
    
    sku_purchase = [3, 8, 3, 10, 3, 3, 1, 3, 7, 6, 1, 2, 7, 0, 7, 9, 1, 5, 1, 0]
    
    d = {}
    for i in sku_purchase:
        if d.get(i) is None:
            d[i] = 1
        else:
            d[i] += 1
    
    d_most = dict(sorted(d.items(), key=lambda item: item[1], reverse=True))
    print(d_most)
    
    # 最受欢迎的商品（键为商品编号），排序结果：
    # {3: 5, 1: 4, 7: 3, 0: 2, 8: 1, 10: 1, 6: 1, 2: 1, 9: 1, 5: 1}
    

但是，如果使用 Counter，能写出更加简化的代码。

首先，导入 Counter 类：

    
    
    In [35]: from collections import Counter
    

然后，使用一行代码搞定：

    
    
    In [42]: Counter(sku_purchase).most_common()
    Out[42]:
    [(3, 5),(1, 4),(7, 3),(0, 2),(8, 1),(10, 1),(6, 1),(2, 1),(9, 1),(5, 1)]
    

仅仅一行代码，便输出统计结果。并且，输出按照购买次数的由大到小排序好的列表，比如，编号为3 的商品最受欢迎，一共购买了 5 次。

除此之外，使用 Counter 能快速统计，一句话中单词出现次数，一个单词中字符出现次数。如下所示：

    
    
    In [41]: Counter('i love python so much').most_common()
    Out[41]:
    [(' ', 4),
     ('o', 3),
     ('h', 2),
     ('i', 1),
     ('l', 1),
     ('v', 1),
     ('e', 1),
     ('p', 1),
     ('y', 1),
     ('t', 1),
     ('n', 1),
     ('s', 1),
     ('m', 1),
     ('u', 1),
     ('c', 1)]
    

### DefaultDict

DefaultDict 能自动创建一个被初始化的字典，也就是每个键都已经被访问过一次。

首先，导入 DefaultDict：

    
    
    In [44]: from collections import defaultdict
    

创建一个字典值类型为 int 的默认字典：

    
    
    In [45]: d = defaultdict(int)
    

创建一个字典值类型为 list 的默认字典：

    
    
    In [46]: d = defaultdict(list)
    
    In [47]: d
    Out[47]: defaultdict(list, {})
    

统计下面字符串：

    
    
    from collections import defaultdict
    

每个字符出现的位置索引：

    
    
    d = defaultdict(list)
    s = 'from collections import defaultdict'
    for index,i in enumerate(s):
        d[i].append(index)
    print(d)
    
    defaultdict(<class 'list'>, {'f': [0, 26], 'r': [1, 21], 'o': [2, 6, 13, 20], 'm': [3, 18], ' ': [4, 16, 23], 'c': [5, 10, 33], 'l': [7, 8, 29], 'e': [9, 25], 't': [11, 22, 30, 34], 'i': [12, 17, 32], 'n': [14], 's': [15], 'p': [19], 'd': [24, 31], 'a': [27], 'u': [28]})
    

当尝试访问一个不在字典中的键时，将会抛出一个异常。但是，使用 DefaultDict 帮助我们初始化。

如果不使用 DefaultDict，就需要写 `if -else` 逻辑。

如果键不在字典中，手动初始化一个列表 []，并放入第一个元素——字符的索引 index。就像下面这样：

    
    
    d = {}
    s = 'from collections import defaultdict'
    for index,i in enumerate(s):
        if i in d:
            d[i].append(index)
        else:
            d[i] = [index]
    print(d)
    # 结果如下：
    {'f': [0, 26], 'r': [1, 21], 'o': [2, 6, 13, 20], 'm': [3, 18], ' ': [4, 16, 23], 'c': [5, 10, 33], 'l': [7, 8, 29], 'e': [9, 25], 't': [11, 22, 30, 34], 'i': [12, 17, 32], 'n': [14], 's': [15], 'p': [19], 'd': [24, 31], 'a': [27], 'u': [28]}
    

虽然也能得到同样结果，但是很显然，使用 DefaultDict，代码更加简洁。

### 排序词

排序词（permutation）：两个字符串含有相同字符，但字符顺序不同。

    
    
    from collections import defaultdict
    
    def is_permutation(str1, str2):
        if str1 is None or str2 is None:
            return False
        if len(str1) != len(str2):
            return False
        unq_s1 = defaultdict(int)
        unq_s2 = defaultdict(int)
        for c1 in str1:
            unq_s1[c1] += 1
        for c2 in str2:
            unq_s2[c2] += 1
    
        return unq_s1 == unq_s2
    

defaultdict，字典值默认类型初始化为 int，计数默次数都为 0。

统计出的两个 defaultdict：unq_s1、unq_s2，如果相等，就表明 str1、str2 互为排序词。

下面，测试：

    
    
    r = is_permutation('nice', 'cine')
    print(r)  # True
    
    r = is_permutation('', '')
    print(r)  # True
    
    r = is_permutation('', None)
    print(r)  # False
    
    r = is_permutation('work', 'woo')
    print(r)  # False
    

### 单词频次

使用 yield 解耦数据读取 python_read 和数据处理 process。

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
    

调用两个函数，使用 Counter 类统计出频次的排序：

    
    
    for line in python_read('test.txt'):
        process(line)
    
    frequency = Counter(d).most_common()
    print(frequency)
    

### 小结

list、tuple、set、dict 等，对大家来说都比较熟知。

今天，与大家一起学习，比它们更强大的三种数据类型：

  * NamedTuple：替换整数索引，使用可读性更好的字符串
  * Counter：快速计数
  * DefaultDict：默认初始化某类型的字典值

