  *     *       * 内置函数 type() 和 isinstance()
        * type()
          * 1\. 查看一个对象的类型
          * 2\. 创建一个类
        * isinstance()
      * Metaclass
        * type 与 object 的关系
      * 自定义 Metaclass

### 内置函数 type() 和 isinstance()

在讲 Metaclass 之前，我们先了解一下几个相关的内置函数：type() 和 isinstance()。

#### type()

type() 有两个主要的功能：（1）查看一个变量（对象）的类型；（2）创建一个类（class）。

##### **1\. 查看一个对象的类型**

当传入一个参数时，返回这个对象的类型。

    
    
    class ClassA:
        name = "type test"
    
    a = ClassA()
    b = 3.0
    
    print(type(a))
    print(type(b))
    print(type("This is string"))
    print()
    
    print(a.__class__)
    print(b.__class__)
    

输出结果：

    
    
    <class 'advanced_programming.Metaclass.ClassA'>
    <class 'float'>
    <class 'str'>
    
    <class 'advanced_programming.Metaclass.ClassA'>
    <class 'float'>
    

这个时候，通常与 `object.__class__` 的功能相同，都是返回对象的类型。

##### **2\. 创建一个类**

当传入三个参数时，用来创建一个类。

    
    
    class type(name, bases, dict)
    

  * name：要创建的类的类型
  * bases：要创建的类的基类，Python 中允许多继承，因此这是一个 tuple 元组的类型
  * dict：要创建的类的属性，是一个 dict 字典类型

    
    
    ClassVariable = type('ClassA', (object,), dict(name="type test"))
    a = ClassVariable()
    print(type(a))
    print(a.name)
    

输出结果：

    
    
    <class 'advanced_programming.Metaclass.ClassA'>
    type test
    

在这段代码中，通过 type('ClassA', (object,), dict(name="type test")) 创建一个类
ClassVariable，再通过 ClassVariable() 创建一个实例 a。通过 type 创建的类 ClassA 和 class ClassA
这种定义创建的类是一样的。

    
    
    class ClassA:
        name = "type test"
    
    a = ClassA()
    print(type(a))
    print(a.name)
    

正常情况下，我们都是用 class Xxx… 来定义一个类；但是，type() 函数也允许我们动态地创建一个类。Python
是一个解释型的动态语言，这是与静态语言（如 Java、C++）的最大区别之处：动态语言可以很方便地在运行期间动态地创建类。

#### isinstance()

isinstance() 的作用是判断一个对象是不是某个类型的实例，函数原型：

    
    
    isinstance(object, classinfo)
    

  * object：要判断的对象
  * classinfo：期望的类型

> 如果 object 是 classinfo 的一个实例，或 classinfo 子类的一个实例，则返回 True，否则返回 False。

看一个示例：

    
    
    class BaseClass:
        name = "Base"
    
    class SubClass(BaseClass):
        pass
    
    
    base = BaseClass()
    sub = SubClass()
    
    print(isinstance(base, BaseClass))
    print(isinstance(base, SubClass))
    print()
    
    print(isinstance(sub, SubClass))
    print(isinstance(sub, BaseClass))
    

输出结果：

    
    
    True
    False
    
    True
    True
    

如果要知道子类与父类之间的继承关系，可用 issubclass() 方法，或 `object.__bases__` 。

    
    
    print(issubclass(SubClass, BaseClass))
    print(issubclass(BaseClass, SubClass))
    print(SubClass.__bases__)
    

输出结果：

    
    
    True
    False
    (<class 'advanced_programming.Metaclass.BaseClass'>,)
    

### Metaclass

metaclas 直译为 **元类** （我还是更喜欢原名，后文也将不再翻译），可控制类的属性和类实例的创建过程。

在 Python 中，一切都是对象：一个整数是对象，一串字符是对象，一个类实例也一个对象，类本身也是一个对象。
一个类也是一个对象，和其他对象一样，它是元类（Metaclass）的一个实例。我们用一个图来表示对象（obj，或叫实例）、类（class）、元类（Metaclass）的关系。

![enter image description
here](https://images.gitbook.cn/4d86dd10-c3ae-11e8-b424-4761250c06f8)

**附图3-1 对象、类、元类的关系**

我们来看下面的实例：

    
    
    class MyClass:
        pass
    
    m = MyClass()
    print(type(MyClass))
    print(type(m))
    print()
    
    print(isinstance(m, MyClass))
    print(isinstance(MyClass, type))
    

输出结果：

    
    
    <class 'type'>
    <class 'advanced_programming.Metaclass.MyClass'>
    
    True
    True
    

默认的 metaclas 是 type，所以上面的代码中我们看到 MyClass 的类型是 type 。不幸的是，为了向后兼容，type
这个类型总是让人困惑：type 也可以当作函数来使用，返回一个对象的类型。

造成这种种困扰的始作俑者就是 type，type 在 Python 中是一个极为特殊的类型。为了彻底理解 Metaclass，我们先要搞清楚 type 和
object 的关系。

#### type 与 object 的关系

在 Python 3 中，object 是所有类的基类，内置的类、自定义的类都直接或间接地继承自 object 类。如果你去看源码，会发现 type
也继承自 object。这就对我们的理解造成极大的困扰，主要表现在以下三点。

  * type 是一个 Metaclass，而且是一个默认的 Metaclass。也就是说，type 是 object 的类型，object 是 type 的一个实例。
  * type 是 object 的一个子类，继承 object 的所以属性和行为。
  * type 还是一个 callable，即实现了 `__call__` 方法，可以当成一个函数来使用。

我们用一张图来解释 type 和 object 的关系：

![enter image description
here](https://images.gitbook.cn/8e6e76d0-c3ae-11e8-b424-4761250c06f8)

**附图3-2 type 和 object 的关系**

type 和 object 有点像“蛋生鸡”与“鸡生蛋”的关系：type 是 object 的子类，同时 object 又是 type 的一个实例（type
是 object 的类型）；二者是不可分离的。

type 的类型也是 type，这个估计更难理解，先这么记着吧！

我们可以自定义 Metaclass，自定义的 Metaclass 必须继承自 type。自定义的 Metaclass 通常以 Metaclass（或
Meta）作为后缀进行取名以示区分（如附图3-2中的 CustomMetaclass），CustomMetaclass 和 type 都是
Metaclass 类型。

所有的类都继承自 object，包括内置的类和用户自定义的类。一般来说类 Class 的类型为 type（即一般的类的 Metaclass 是 type，是
type 的一个实例）。如果要改变类的 Metaclass，必须在类定义时显示地指定它的 Metaclass，如下面的示例：

    
    
    class CustomMetaclass(type):
        pass
    
    class CustomClass(metaclass=CustomMetaclass):
        pass
    
    print(type(object))
    print(type(type))
    print()
    
    obj = CustomClass()
    print(type(CustomClass))
    print(type(obj))
    
    print()
    print(isinstance(obj, CustomClass))
    print(isinstance(obj, object))
    

输出结果：

    
    
    <class 'type'>
    <class 'type'>
    
    <class 'advanced_programming.Metaclass.CustomMetaclass'>
    <class 'advanced_programming.Metaclass.CustomClass'>
    
    True
    True
    

### 自定义 Metaclass

自定义 Metaclass 时，要注意以下几个点。

**（1）object 的`__init__` 方法只有 1 参数，但自定义 metaclass 的 `__init__` 有 4 个参数。**

    
    
    def __init__(self)
    

但 type 重写了 `__init__` 方法，有 4 个参数：

    
    
    def __init__(cls, what, bases=None, dict=None)
    

因为自定义 Metaclass 继承自 type，所以重写 `__init__` 方法时也要有四个参数。

（2） **普通的类，重写`__call__` 方法说明对象是 callable 的。**

在 Metaclass 中 `__call__` 方法还负责对象的创建，一个对象的创建过程大致是附图 3-3 这样的：

![enter image description
here](https://images.gitbook.cn/feffc160-c3ae-11e8-b424-4761250c06f8)

**附图 3-3 对象实例的创建过程**

我们结合实例代码一起看一下：

    
    
    class CustomMetaclass(type):
    
        def __init__(cls, what, bases=None, dict=None):
            print("CustomMetaclass.__init__ cls:", cls)
            super().__init__(what, bases, dict)
    
        def __call__(cls, *args, **kwargs):
            print("CustomMetaclass.__call__ args:", args, kwargs)
            self = super(CustomMetaclass, cls).__call__(*args, **kwargs)
            print("CustomMetaclass.__call__ self:", self)
            return self
    
    class CustomClass(metaclass=CustomMetaclass):
    
        def __init__(self, *args, **kwargs):
            print("CustomClass.__init__ self:", self)
            super().__init__()
    
        def __new__(cls, *args, **kwargs):
            self = super().__new__(cls)
            print("CustomClass.__new__, self:", self)
            return self
    
        def __call__(self, *args, **kwargs):
            print("CustomClass.__call__ args:", args)
    
    obj = CustomClass("Meta arg1", "Meta arg2", kwarg1=1, kwarg2=2)
    print(type(CustomClass))
    print(obj)
    obj("arg1", "arg2")
    

输出结果：

    
    
    CustomMetaclass.__init__ cls: <class 'advanced_programming.Metaclass.CustomClass'>
    CustomMetaclass.__call__ args: ('Meta arg1', 'Meta arg2') {'kwarg1': 1, 'kwarg2': 2}
    CustomClass.__new__, self: <advanced_programming.Metaclass.CustomClass object at 0x02B921B0>
    CustomClass.__init__ self: <advanced_programming.Metaclass.CustomClass object at 0x02B921B0>
    CustomMetaclass.__call__ self: <advanced_programming.Metaclass.CustomClass object at 0x02B921B0>
    <class 'advanced_programming.Metaclass.CustomMetaclass'>
    <advanced_programming.Metaclass.CustomClass object at 0x02B921B0>
    CustomClass.__call__ args: ('arg1', 'arg2')
    

图中每一条实线表示具体操作，每一条虚线表示返回的过程，实例对象的整个创建过程大致是以下这样的。

  1. `metaclass.__init__` 进行一些初始化的操作，如一些全局变量的初始化。
  2. `metaclass.__call__` 进行创建实例，创建的过程中会调用 class 的 `__new__` 和 `__init__` 方法。
  3. `class.__new__` 进行具体的实例化的操作，并返回一个实例对象 obj(0x02B921B0)。
  4. `class.__init__` 对返回的实例对象 obj（0x02B921B0）进行初始化，如一些状态和属性的设置。
  5. 返回一个用户真正需要使用的对象 obj（0x02B921B0）。

到这里我们应该知道，通过
Metaclass，我们几乎可以自定义一个对象生命周期的各个过程。现在再回去看一个[《第04课：生活中的单例模式》](https://gitbook.cn/gitchat/column/5b26040ac81ac568fcf64ea3/topic/5b2605a7c81ac568fcf64f41)第二个实现方式，应该能更深刻地理解其中的原理了。

