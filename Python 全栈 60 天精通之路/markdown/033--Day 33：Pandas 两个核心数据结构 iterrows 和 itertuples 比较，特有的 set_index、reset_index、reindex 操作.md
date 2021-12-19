今天介绍 Pandas 的 2 个基础对象，以及 Pandas 常用的几个进阶功能。

### 一维数组 Series

Series 是 Pandas 两大数据结构中（DataFrame、Series）的一种，先从 Series 的定义说起。Series
是一种类似于一维数组的对象，它由一组数据域以及一组与之相关联的数据标签和索引组成。

Series 对象也是一个 NumPy 的数组，因此 NumPy 的数组处理函数可以直接对 Series 进行处理。

与此同时，Series 除了可以使用位置索引作为下标存取元素之外，还可以使用“标签下标”存取元素，这一点和字典相似，每个 Series
对象都由两个数组组成：

  * index：它是从 NumPy 数组继承的 Index 对象，保存标签信息。
  * values：保存值的 NumPy 数组。

接下来，分别介绍 Series 内元素的增加、删除、修改、访问。

#### **创建 Series**

Series 的标准构造函数，列举常用的几个参数：

    
    
    Series(data=None, index=None, dtype=None, name=None)
    

其中，data 为数据部分，index 为标签部分，省略下默认为自增整数索引，dtype 为 str、numpy.dtype 或
ExtensionDtype。

创建一个 series，如下：

    
    
    In [85]: ps = pd.Series(data=[-3,2,1],index=['a','f','b'],dtype=np.float32)     
    
    In [86]: ps                                                                     
    Out[86]: 
    a   -3.0
    f    2.0
    b    1.0
    dtype: float32
    

#### **增加元素**

在 ps 基础上增加一个元素，使用 append，如下：

    
    
    In [112]: ps.append(pd.Series(data=[-8.0],index=['f']))                         
    Out[112]: 
    a    4.0
    f    2.0
    b    1.0
    f   -8.0
    dtype: float64
    

可以看到，Pandas 允许包含重复的标签：

    
    
    In [114]: psn = ps.append(pd.Series(data=[-8.0],index=['f']))                   
    
    In [115]: psn                                                                   
    Out[115]: 
    a    4.0
    f    2.0
    b    1.0
    f   -8.0
    dtype: float64
    
    In [116]: psn['f']                                                              
    Out[116]: 
    f    2.0
    f   -8.0
    dtype: float64
    

利用标签访问元素，返回所有带标签的数据。

#### **删除元素**

使用 drop 删除指定标签的数据，如下：

    
    
    In [119]: ps                                                                    
    Out[119]: 
    a    4.0
    f    2.0
    b    1.0
    dtype: float32
    
    In [120]: psd = ps.drop('f')                                                    
    In [121]: psd                                                                  
    Out[121]: 
    a    4.0
    b    1.0
    dtype: float32
    

注意不管是 append 操作，还是 drop 操作，都是发生在原数据的副本上，不是原数据上。

#### **修改元素**

通过标签修改对应数据，如下所示：

    
    
    In [123]: psn                                                                   
    Out[123]: 
    a    4.0
    f    2.0
    b    1.0
    f   -8.0
    dtype: float64
    
    In [124]: psn['f'] = 10.0                                                       
    In [125]: psn                                                                   
    Out[125]: 
    a     4.0
    f    10.0
    b     1.0
    f    10.0
    dtype: float64
    

标签相同的数据，都会被修改。

#### **访问元素**

访问元素，Pandas 提供两种方法：

  * 一种通过默认的整数索引，在 Series 对象未被显示的指定 label 时，都是通过索引访问；
  * 另一种方式是通过标签访问。

    
    
    In [126]: ps                                                                    
    Out[126]: 
    a    4.0
    f    2.0
    b    1.0
    dtype: float32
    
    In [128]: ps[2] # 索引访问                              
    Out[128]: 1.0
    
    In [127]: ps['b']  # 标签访问                                                             
    Out[127]: 1.0
    

### 二维数组 DataFrame

DataFrame，Pandas 两个重要数据结构中的另一个，可以看做是 Series 的容器。

DataFrame 同时具有行、列标签，是二维的数组，行方向轴 axis 为 0，列方向 axis 为 1，如下：

    
    
    axis : {0 or 'index', 1 or 'columns'}
    

#### **创建 DataFrame**

DataFrame 构造函数如下：

    
    
    DataFrame(data=None, index=None, columns=None, dtype=None, copy=False)
    

参数意义与 Series 相似，不再赘述。

创建 DataFrame 的常用方法：

    
    
    In [134]: df = pd.DataFrame([['gz',4.0,'2019-01-01'],['lg',1.2,'2019-06-01']],index = ['a','f'], columns = ['nm', 'y','da'])                          
    
    In [135]: df                                                                    
    Out[135]: 
       nm    y          da
    a  gz  4.0  2019-01-01
    f  lg  1.2  2019-06-01
    

也可以通过字典传入，得到一样的 DataFrame，如下：

    
    
    In [136]: df2 = pd.DataFrame({'nm':['gz','lg'],'y':[4.0,1.2], 'da':['2019-01-01', '2019-06-01']},index = ['a','f'])                                   
    In [137]: df2                                                                   
    Out[137]: 
       nm    y          da
    a  gz  4.0  2019-01-01
    f  lg  1.2  2019-06-01
    

#### **增加数据**

通过增加一个 Series，扩充到 DataFrame 中，如下所示：

    
    
    In [143]: dfn = df.append(pd.Series(data=['zx',3.6,'2019-05-01'],index=['nm','y','da'],name='b'))                                                     
    In [144]: dfn                                                                   
    Out[144]: 
       nm    y          da
    a  gz  4.0  2019-01-01
    f  lg  1.2  2019-06-01
    b  zx  3.6  2019-05-01
    

Series 的 index 与 DataFrame 的 column 对齐，name 与 DataFrame 的 index 对齐。

#### **删除数据**

与 Series 删除类似，也是使用 drop 删除指定索引或标签的数据。如下，注意删除仍然是在 dfn 的副本上进行，像下面这样删除对 dfn
没有任何影响。

    
    
    In [145]: dfn.drop('b')                                                         
    Out[145]: 
       nm    y          da
    a  gz  4.0  2019-01-01
    f  lg  1.2  2019-06-01
    

如果要删除某列，需要设定 axis 为 1，如下所示：

    
    
    In [147]: dfn.drop('y',axis=1)                                                  
    Out[147]: 
       nm          da
    a  gz  2019-01-01
    f  lg  2019-06-01
    b  zx  2019-05-01
    

#### **修改数据**

修改依然是先通过索引或标签定位到数据，然后修改，如下所示：

    
    
    In [151]: dfn.loc['a','da']='2019-04-01'                                        
    In [152]: dfn                                                                   
    Out[152]: 
       nm    y          da
    a  gz  4.0  2019-04-01
    f  lg  1.2  2019-06-01
    b  zx  3.6  2019-05-01
    

作为入门或许理解到这里就够了，但要想进阶，还必须要了解一个关键点：链式赋值（chained assignment）。

使用 Pandas 链式赋值，经常会触发一个警告——SettingWithCopyWarning，类似下面一串警告：

    
    
    SettingWithCopyWarning:
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    See the caveats in the documentation
    

归根结底，是因为代码中出现链式操作，那么什么是链式操作？如下，就是一次链式赋值：

    
    
    tmp = df[df.a<4]
    tmp['c'] = 200
    

出现此 Warning，需要理会它吗？还是需要！

    
    
    import pandas as pd
    
    df = pd.DataFrame({'a':[1,3,5],'b':[4,2,7],'c':[0,3,1]})
    print(df)
    
    tmp = df[df.a<4]
    tmp['c'] = 200
    print('-----原 df 没有改变------')
    print(df)
    

输出结果：

    
    
       a  b  c
    0  1  4  0
    1  3  2  3
    2  5  7  1
    SettingWithCopyWarning:
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the caveats in the documentation: http://pandas.pydata.org/pandas-docs/stable/user_guide/indexing.html#returning-a-view-versus-a-copy
      tmp['c'] = 200
    -----原 df 没有改变------
       a  b  c
    0  1  4  0
    1  3  2  3
    2  5  7  1
    

以上发生链式赋值，本意想改变 df，但是 df 未发生任何变化。因为 tmp 是 df 的 copy，而并非 view。

因此，如何创建 df 的 view，而确保不发生链式操作，建议使用 .loc，如下调用：

    
    
    import pandas as pd
    
    df = pd.DataFrame({'a':[1,3,5],'b':[4,2,7],'c':[0,3,1]})
    print(df)
    
    df.loc[df.a<4,'c'] = 100
    print(df)
    

输出结果：

    
    
       a  b  c
    0  1  4  0
    1  3  2  3
    2  5  7  1
       a  b    c
    0  1  4  100
    1  3  2  100
    2  5  7    1
    

#### **访问数据**

Pandas 推荐使用访问接口 iloc、loc 访问数据，详细使用如下：

    
    
    In [153]: df                                                                    
    Out[153]: 
       nm    y          da
    a  gz  4.0  2019-01-01
    f  lg  1.2  2019-06-01
    
    In [154]: df.iloc[1,:]                                                          
    Out[154]: 
    nm            lg
    y            1.2
    da    2019-06-01
    Name: f, dtype: object
    
    In [155]: df.loc['f',:]                                                         
    Out[155]: 
    nm            lg
    y            1.2
    da    2019-06-01
    Name: f, dtype: object
    
    In [156]: df.loc['f','nm']                                                      
    Out[156]: 'lg'
    

iloc 索引访问，loc 标签访问。

### 强大的 []

原生 Python 中，`[]` 操作符常见的是与 list 搭配使用，并且 `[]` 操作符支持的对象只能是整型、切片，例如：

    
    
    a= [1, 3, 5]
    # 以下OK：
    a[1]
    a[True]
    a[:2]
    a[:6]
    

list 等可迭代对象是禁止的，如下：

    
    
    # 不OK：
    a[[1,2]]
    

为了更好地发挥 Python 的强大威力，Pandas 的 `[]` 操作符支持 list 对象。

为了演示，生成一个 DataFrame 实例 df1 ：

    
    
    import pandas as pd
    import numpy as  np
    
    np.random.seed(1)
    df1=pd.DataFrame(np.random.randint(1,10,(4,3)),
    index=['r1','r2','r3','r4'],
    columns=['c1','c2','c3'])
    
    #结果
      c1  c2  c3
    r1  6  9  6
    r2  1  1  2
    r3  8  7  3
    r4  5  6  3
    

`[]` 操作符支持 list 对象，对 c1 和 c3 列索引：

    
    
    df1[['c1','c3']]
    #测试支持列索引
      c1  c3
    r1  6  6
    r2  1  2
    r3  8  3
    r4  5  3
    

注意这种支持也仅仅是对列索引，对于行索引依然不支持，如下：

    
    
    df1[['r1', 'r4']]
    #测试结果，会出现以下报错：
    KeyError: "None of [Index(['r1', 'r4'], dtype='object')] are in the [columns]"
    

利用行切片整数索引，获取 DataFrame，Pandas 依然延续了原生 Python 的风格，如下：

    
    
    In [7]: df1[1:3]
    Out[7]:
        c1  c2  c3
    r2   1   1   2
    r3   8   7   3
    

Pandas 还对此做了增强，同时支持行切片标签索引获取 DataFrame， **注意包括终止标签** 。

    
    
    In [9]: df1['r1':'r3']
    Out[9]:
        c1  c2  c3
    r1   6   9   6
    r2   1   1   2
    r3   8   7   3
    

`[]` 操作符除了支持以上对象类型外，还支持哪些对象类型？

**DataFrame 与标量结合**

仅仅一行 `df1>4` 就能实现逐元素与 4 比较，大于 4 为 True，否则为 False。

然后返回一个新的 DataFrame，如下所示：

    
    
    In [10]: df1 > 4
    Out[10]:
           c1     c2     c3
    r1   True   True   True
    r2  False  False  False
    r3   True   True  False
    r4   True   True  False
    

结合 Pandas 的方括号 `[]` 后，便能实现元素级的操作，不大于 4 的元素被赋值为 NaN：

    
    
    In [11]: df1[df1>4]
    Out[11]:
         c1   c2   c3
    r1  6.0  9.0  6.0
    r2  NaN  NaN  NaN
    r3  8.0  7.0  NaN
    r4  5.0  6.0  NaN
    

`df1>4` 操作返回一个元素值为 True 或 False 的 DataFrame，`(df1>4).values.tolist()` 得到原生的
python list 对象：

    
    
    In [15]: inner = ((df1>4).values).tolist()
    
    In [16]: inner
    Out[16]:
    [[True, True, True],
     [False, False, False],
     [True, True, False],
     [True, True, False]]
    
    In [17]: type(inner)
    Out[17]: list
    

DataFrame、`[]` 和 list 实例结合会出现如下的结果：

    
    
    In [18]: df1[inner]
    Out[18]:
        c1  c2  c3
    r1   6   9   6
    r1   6   9   6
    r1   6   9   6
    r3   8   7   3
    r3   8   7   3
    r4   5   6   3
    r4   5   6   3
    

仅仅改变为 `df1[python list]`，而不是原来的 `df1[DataFrame]` 后，返回的结果迥异。

### iterrows、itertuples

尽管 Pandas 已经尽可能向量化，让使用者尽可能避免 for 循环，但是有时不得已，还得要遍历 DataFrame。Pandas 提供
iterrows、itertuples 两种行级遍历。

这两种遍历效率上有什么不同，下面使用 Kaggle 上泰坦尼克号数据集的训练部分，测试遍历时的效率。

    
    
    import pandas as pd
    df = pd.read_csv('dataset/titanic_train_data.csv',sep=',')
    

df 一共有 891 行。

使用 iterrows 遍历打印所有行，在 IPython 里输入以下行：

    
    
    def iterrows_time(df):
        for i,row in df.iterrows():
            print(row)
    

然后使用魔法指令 `%time`，直接打印出此行的运行时间：

    
    
    In [77]: %time iterrows_time(df)
    

实验 10 次，平均耗时 920 ms。

然后，使用 itertuples 遍历打印每行：

    
    
    def itertuples_time(df):
        for nt in df.itertuples():
            print(nt)
    

操作平均耗时 132 ms。

也就是说打印 891 行，itertuples 遍历能节省 6 倍时间，这对于处理几十万、几百万级别的数据时，效率优势会更加凸显。

分析 iterrows 慢的原因，通过分析源码，我们看到每次遍历时，Pandas 都会把 v 包装为一个 klass 对象，`klass(v,
index=columns, name=k)` 操作耗时。

    
    
    def iterrows(self):
            columns = self.columns
            klass = self._constructor_sliced
            for k, v in zip(self.index, self.values):
                s = klass(v, index=columns, name=k)
                yield k, s
    

因此，平时涉及遍历时，建议使用 itertuples。

### set_index、reset_index、reindex

Pandas 的 index 可以是整数或字符串。字符串型的行标签可以说是 pandas 的灵魂一笔，支撑了 pandas
很多强大的业务功能，比如多个数据框的 join、merge 操作，自动对齐等。下面总结几个平时常用的 index 操作。

#### **列转 index**

有时，我们想把现有的数据框的某些列转化为 index，为之后的更多操作做准备。

某列转为 index 的实现方法如下：

    
    
    import pandas as pd
    
    In [19]: df1 = pd.DataFrame({'a':[1,3,5],'b':[9,4,12]})
    
    In [20]: df1
    Out[20]:
       a   b
    0  1   9
    1  3   4
    2  5  12
    

把 a 列转为 index，分别不保留、保留原来的 a 列：

    
    
    In [21]: df1.set_index('a',drop=False)
    Out[21]:
       a   b
    a
    1  1   9
    3  3   4
    5  5  12
    
    In [22]: df1.set_index('a',drop=True)
    Out[22]:
        b
    a
    1   9
    3   4
    5  12
    

对于频繁使用的列，建议转化为索引，效率会更高。

#### **index 转列**

使用 reset_index，将 index 转化为 DataFrame 的列，操作如下：

    
    
    In [92]: df1 = pd.DataFrame({'a':[1,3,5],'b':[9,4,12]})
    
    In [93]: df2 = df1.set_index('a',drop=True)
    
    In [94]: df2.reset_index('a',drop=True)
    Out[94]:
        b
    0   9
    1   4
    2  12
    
    In [95]: df2.reset_index('a',drop=False)
    Out[95]:
       a   b
    0  1   9
    1  3   4
    2  5  12
    

#### **reindex**

如果想按照某种规则，按照行重新排序数据，靠 reindex 函数可以实现：

    
    
    In [96]: df1 = pd.DataFrame({'a':[1,3,5],'b':[9,4,12]})
    
    In [97]: df1
    Out[97]:
       a   b
    0  1   9
    1  3   4
    2  5  12
    
    
    In [26]: df1.reindex([0,3,2,1]) 
    Out[26]:
         a     b
    0  1.0   9.0
    3  NaN   NaN
    2  5.0  12.0
    1  3.0   4.0
    

df1 原来有的行索引会重新按照最新的索引 [0,3,2,1] 重新对齐，原来没有的行索引 3，默认数据都填充为 NaN。

列数据的调整，也一样通过 reindex 实现，如下：

    
    
    In [27]: df1.reindex(columns=['b','a','c'])
    Out[27]:
        b  a   c
    0   9  1 NaN
    1   4  3 NaN
    2  12  5 NaN
    

以上是关于 index 调整的方法，在实际中很有用，希望读者们能掌握。

### 小结

今天一起学习：

  * Pandas 基本的数据操作，一维数组 Series，二维数组 DataFrame；
  * 更强大的 `[]` 操作；
  * iterrows、itertuples的效率比较；
  * Pandas 特有的 set_index、reset_index、reindex 操作。 

