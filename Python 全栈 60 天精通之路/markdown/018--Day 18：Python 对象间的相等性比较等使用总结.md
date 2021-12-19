Python 中，对象相等性比较相关关键字包括 is、in，比较运算符有 `==`。

  * is 判断两个对象的标识号是否相等；
  * in 用于成员检测
  * `==` 用于判断值或内容是否相等，默认是基于两个对象的标识号比较。

也就是说，如果 `a is b` 为 True 且如果按照默认行为，意味着 `a==b` 也为 True。

下面详细介绍它们的用法。

### is 判断标识号是否相等

is 比较的是两个对象的标识号是否相等，Python 中使用 id() 函数获取对象的标识号。

    
    
    In [1]: a = [1,2,3]
    In [2]: id(a) 
    Out[2]: 95219592
    
    In [5]: b = [1,2,3] #再创建一个列表实例，元素取值也为 1,2,3
    In [6]: id(b) 
    Out[6]: 95165640
    

创建的两个列表实例位于不同的内存地址，所以它们的标识号不等。

    
    
    In [7]: a is b
    Out[7]: False
    

即便对于两个空列表实例，它们 is 比较的结果也是 False：

    
    
    In [17]: a, b = [], []
    
    In [18]: a is b
    Out[18]: False
    

对于序列型、字典型、集合型对象，一个对象实例指向另一个对象实例，is 比较才返回真值。

    
    
    In [19]: a, b = {'a':[1,2,3]},{'id':'book id', 'price':'book price'}
    
    In [20]: a = b
    
    In [21]: a is b
    Out[21]: True
    

对于值类型而言，不同的编译器可能会做不同的优化。从性能角度考虑，它们会缓存一些值类型的对象实例。所以，使用 is
比较时，返回的结果看起来会有些不太符合预期。注意观察下面两种情况，同样的整数值，使用 is 得到不同结果。

    
    
    In [100]: a = 123
    In [101]: b = 123
    In [102]: a is b
    Out[102]: True
    
    
    
    In [103]: c = 123456
    In [104]: d = 123456
    
    In [105]: c is d
    Out[105]: False
    

Python 解释器，对位于区间 [-5,256] 内的小整数，会进行缓存，不在该范围内的不会缓存，所以才出现上面的现象。

Python 中 None 对象是一个单例类的实例，具有唯一的标识号，如下所示：

    
    
    In [23]: id(None)
    Out[23]: 140722146270432
    

在判断某个对象是否为 None 时，最便捷的做法：`variable is None`，如下所示：

    
    
    In [24]: a = None
    
    In [25]: a is None
    Out[25]: True
    
    In [26]: id(a)
    Out[26]: 140722146270432
    

### in 用于成员检测

  * 如果元素 i 是 s 的成员，则 `i in s` 为 True； 
  * 若不是 s 的成员，则返回 False，也就是 `i not in s` 为 True。 

对于字符串类型，`i in s` 为 True，意味着 i 是 s 的子串，也就是 s.find(i) 返回大于 - 的值。举例如下：

    
    
    In [70]: 'ab' in 'abc'
    Out[70]: True
    In [27]: 'abc'.find('ab')
    Out[27]: 0
    
    In [71]: 'ab' in 'acb'
    Out[71]: False
    In [29]: 'abc'.find('ac')
    Out[29]: -1
    

内置的序列类型、字典类型和集合类型，都支持 in 操作。对于字典类型，in 操作判断 i 是否是字典的键。

    
    
    In [30]: [1,2] in [[1,2],'str']
    Out[30]: True
    
    In [31]: 'apple' in {'orange':1.5,'banana':2.3,'apple':5.2}
    Out[31]: True
    

对于自定义类型，判断是否位于序列类型中，需要重写序列类型的 魔法方法 __contains__。

具体操作步骤如下：

  * 自定义 Student 类，无特殊之处
  * Students 类继承 list，并重写 __contains__ 方法

根据 Student 类的 name 属性，判断某 Student 是否在 Students 序列对象中。

    
    
    class Student():
        def __init__(self,name):
            self._name = name
    
        @property
        def name(self):
            return self._name
    
        @name.setter
        def name(self,val):
            self._name = val
    
    class Students(list):
            def __contains__(self,stu):
                for s in self:
                    if s.name ==stu.name:
                        return True
                return False
    

Student、Students 类的示意图：

![](https://images.gitbook.cn/72f20180-5695-11ea-8ccf-d385e6cc64ec)

使用自定义类，s3 的名字与列表 a 中的第一个元素 s1 重名，所以 `s3 in a` 返回 True。

s4 不在列表 a 中，所以 in 返回 False。

    
    
    s1 = Student('xiaoming')
    s2 = Student('xiaohong')
    
    a = Students()
    a.extend([s1,s2])
    
    s3 = Student('xiaoming')
    print(s3 in a) # True
    
    s4 = Student('xiaoli')
    print(s4 in a) # False
    

### == 判断值是否相等

对于数值型、字符串、列表、字典、集合，默认只要元素值相等，`==` 比较结果是 True。

如下所示：

    
    
    In [46]: str1 = "alg-channel"
    
    In [47]: str2 = "alg-channel"
    
    In [48]: str1 == str2
    Out[48]: True
    
    In [49]: a = [1, 2, 3]
    
    In [50]: b = [1, 2, 3]
    
    In [51]: a==b
    Out[51]: True
    
    In [52]: c = [1, 3, 2]
    
    In [53]: a==c
    Out[53]: False
    
    In [54]: a = {'a':1.0,'b':2.0}
    
    In [55]: b = {'a':1.0,'b':2.0}
    
    In [56]: a==b
    Out[56]: True
    
    In [57]: c = (1,2)
    
    In [58]: d = (1,2)
    
    In [59]: c==d
    Out[59]: True
    
    In [60]: c={1,2,3}
    
    In [61]: d={1,3,2}
    
    In [62]: c==d
    Out[62]: True
    

对于自定义类型，当所有属性取值完全相同的两个实例，判断 `==` 时，返回 False。

但是，大部分场景下，我们希望这两个对象是相等的，这样不用重复添加到列表中。

比如，判断用户是否已经登入时，只要用户所有属性与登入列表中某个用户完全一致时，就认为已经登入。

如下所示，需要重写方法 __eq__，使用 __dict__ 获取实例的所有属性。

    
    
    class Student():
        def __init__(self,name,age):
            self._name = name
            self._age = age
    
        @property
        def name(self):
            return self._name    
        @name.setter
        def name(self,val):
            self._name = val
    
        @property
        def age(self):
            return self._age    
        @age.setter
        def age(self,val):
            self._age = val
    
        def __eq__(self,val):
            print(self.__dict__)
            return self.__dict__ == val.__dict__
    

Student 类的示意图：

![](https://images.gitbook.cn/d1e71680-5695-11ea-ad74-0783a5f0ad5e)

如下，第三个实例 xiaoming2 与已添加到列表 a 中的 xiaoming 属性完全一致，所以 `==` 比较或 in 时，都会返回
True，这正是我们想要的结果。

    
    
    a = []
    xiaoming = Student('xiaoming',29)
    if xiaoming not in a:
        a.append(xiaoming)
    
    xiaohong = Student('xiaohong',30)
    if xiaohong not in a:
        a.append(xiaohong)
    
    xiaoming2 = Student('xiaoming',29)
    if xiaoming2 == xiaoming:
        print('对象完全一致，相等')
    
    if xiaoming2 not in a:
        a.append(xiaoming2)
    
    print(len(a))
    

![](https://images.gitbook.cn/3810f520-5696-11ea-9dfa-cf3c87b75c44)

### 小结

今天学习了 Python 中 is、in、==：

  * is 比较内存地址是否相等，id 获取内存地址。
  * in 成员属于某个序列类型中的检测
  * == 判断值是否相等

