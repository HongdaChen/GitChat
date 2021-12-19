基于 Python 和 NumPy 开发的 Pandas，在数据分析领域，应用非常广泛。而使用 Pandas 处理数据的第一步往往就是读入数据，比如读写
CSV 文件，Pandas 提供了强劲的读取支持，参数有 38 个之多。这些参数中，有的容易被忽略，但却在实际工作中用处很大。比如：

  * 文件读取时设置某些列为时间类型
  * 导入文件，含有重复列
  * 过滤某些列
  * 每次迭代读取 10 行

挑选其中 30 个常用参数，分为 6 大类详细讨论数据读取那些事。以下讲解使用的 Pandas 版本为：0.25.1。

### 基本参数

#### **filepath _or_ buffer**

数据输入路径，可以是文件路径，也可以是 URL，或者实现 read 方法的任意对象。

如下经典的数据集 iris，直接通过 URL 获取。

    
    
    In [160]: pd.read_csv('https://archive.ics.uci.edu/ml/machine-learning-databases/iris/iris.data')
    Out[160]:
         5.1  3.5  1.4  0.2     Iris-setosa
    0    4.9  3.0  1.4  0.2     Iris-setosa
    1    4.7  3.2  1.3  0.2     Iris-setosa
    2    4.6  3.1  1.5  0.2     Iris-setosa
    3    5.0  3.6  1.4  0.2     Iris-setosa
    4    5.4  3.9  1.7  0.4     Iris-setosa
    ..   ...  ...  ...  ...             ...
    144  6.7  3.0  5.2  2.3  Iris-virginica
    145  6.3  2.5  5.0  1.9  Iris-virginica
    146  6.5  3.0  5.2  2.0  Iris-virginica
    147  6.2  3.4  5.4  2.3  Iris-virginica
    148  5.9  3.0  5.1  1.8  Iris-virginica
    
    [149 rows x 5 columns]
    

#### **sep**

数据文件的分隔符，默认为逗号。

注意：如果分割字符长度大于 1，且不是 `\s+`，启动 Python 引擎解析。

举例：test.csv 文件分割符为 `\t`，如果使用 sep 默认的逗号分隔符，读入后的数据混为一体。

    
    
    # 创建并保存数据
    In [1]: d = {'id':[1,2],'name':['gz','lh'],'age':[10,12]}
    In [2]: df = pd.DataFrame(d)
    In [3]: df.to_csv('test.csv',sep='\t')
    
    #读取数据
    In [4]: df = pd.read_csv('test.csv')
    In [5]: df
    Out[5]:
      \tid\tname\tage
    0    0\t1\tgz\t10
    1    1\t2\tlh\t12
    

sep 必须设置为 `'\t'`，数据分割才会正常。

    
    
    In [6]: df = pd.read_csv('test.csv',sep='\t')
    In [6]: df
    Out[6]:
       Unnamed: 0  id name  age
    0           0   1   gz   10
    1           1   2   lh   12
    

#### **delimiter**

分隔符的另一个名字，与 sep 功能相似。

#### **delim_whitespace**

0.18 版本后新加参数，默认为 False，设置为 True 时，表示分割符为空白字符，可以是一个空格、两个空格，或 `\t` 等。

如下 test.csv 文件分隔符为空格时，设置 delim_whitespace 为 True：

    
    
    In [1]: d = {'id':[1,2],'name':['gz','lh'],'age':[10,12]}
    In [2]: df = pd.DataFrame(d)
    In [3]: df.to_csv('test.csv',sep=' ') # 两个空格的分隔符
    
    In [4]: df = pd.read_csv('test.csv',delim_whitespace=True)
    
    In [5]: df
    Out[5]:
       id name  age
    0   1   gz   10
    1   2   lh   12
    

#### **header**

设置导入 DataFrame 的列名称，默认为 `'infer'`，注意它与 names 参数的微妙关系。

#### **names**

没被赋值时，header 会被 infer 为 0，即选取数据文件的第一行作为列名。

当 names 被赋值，header 没被赋值时会被 infer 为 None。如果都赋值，就会实现两个参数的组合功能。

分别看下这几种情况。

**1\. names 没有被赋值，header 也没赋值：**

    
    
    In [1]: df = pd.read_csv('test.csv',delim_whitespace=True)
    
    In [2]: df
    Out[2]:
       id name  age
    0   1   gz   10
    1   2   lh   12
    

**2\. names 没有赋值，header 被赋值：**

    
    
    In [1]: df=pd.read_csv('test.csv',delim_whitespace=True,header=1)
    
    In [2]: df
    Out[2]:
       0  1  gz  10
    0  1  2  lh  12
    

**3\. names 被赋值，header 没有被赋值：**

    
    
    In [1]: pd.read_csv('test.csv',delim_whitespace=True,names=['id','name','age'])
    Out[1]:
       id  name  age
    0  id  name  age
    1   0     1   gz
    2   1     2   lh
    

**4\. names 和 header 都被设置：**

    
    
    In [1]: pd.read_csv('test.csv',delim_whitespace=True,names=['id','name','age'],header=0)
    Out[1]:
       id name  age
    0   1   gz   10
    1   2   lh   12
    

#### **index_col**

index_col 参数表示使用哪个或哪些列作为 index，这个参数也是非常实用的。

当设置 index_col 为 id 列时，index 变为 id 值。

    
    
    In [1]: df = pd.read_csv('test.csv',delim_whitespace=True,index_col=1)
    
    In [2]: df
    Out[2]:
       id name  age
    1   0   gz   10
    2   1   lh   12
    

#### **usecols**

usecols 参数用于选取数据文件的哪些列到 DataFrame 中。

如下所示，我们只想使用源数据文件的 id 和 age 两列，那么可以为 usecols 参数赋值为 `['id','name']`：

    
    
    In [1]: df = pd.read_csv('test.csv',delim_whitespace=True,usecols=['id','name'])
    
    In [2]: df
    Out[2]:
       id name
    0   1   gz
    1   2   lh
    

#### **mangle_dupe_cols**

实际生产用的数据会很复杂，有时导入的数据会含有重名的列。参数 mangle_dupe_cols 默认为 True，重名的列导入后面多一个 `.1`
如果设置为 False，会抛出不支持的异常：

    
    
    ValueError: Setting mangle_dupe_cols=False is not supported yet
    

#### **prefix**

prefix 参数，当导入的数据没有 header 时，设置此参数会自动加一个前缀。比如，设置参数为 col 时，列名变为如下：

    
    
    In [1]: pd.read_csv('test.csv',sep=' ',prefix='col',header=None)
    Out[1]:
       col0 col1  col2 col3
    0   NaN   id  name  age
    1   0.0    1    gz   10
    2   1.0    2    lh   12
    

### 通用解析参数

#### **dtype**

dtype 查看每一列的数据类型，如下：

    
    
    In [1]: d = {'id':[1,2],'name':['gz','lh'],'age':[10,12]}
    In [2]: df = pd.DataFrame(d)
    In [3]: df.to_csv('test.csv',sep=' ',index=False)
    
    In [4]: df = pd.read_csv('test.csv',sep='\s+')    
    
    In [15]: df.dtypes
    Out[15]:
    id       int64
    name    object
    age      int64
    dtype: object
    

如果想修改 age 列的数据类型为 float，在 read_csv 时可以使用 dtype 调整，如下：

    
    
    In [16]: df = pd.read_csv('test.csv',sep='\s+',dtype={'age':float})
    
    In [17]: df
    Out[17]:
       id name   age
    0   1   gz  10.0
    1   2   lh  12.0
    
    In [18]: df.dtypes
    Out[18]:
    id        int64
    name     object
    age     float64
    dtype: object
    

这个参数有用之处可能体现在如下这个例子，就是某列的数据：

    
    
    label
    01
    02
    

如果不显示的指定此列的类型 str，read_csv 解析引擎会自动判断此列为整形，如下在原 test.csv 文件中增加上面一列，如果不指定
dtype，读入后 label 列自动解析为整型。如果现实的指定为 str，则读入的类型自然就是 str。

    
    
    In [51]: df = pd.read_csv('test.csv',sep='\s+',dtype={'label':str})             
    

#### **engine**

engine 参数，Pandas 目前的解析引擎提供两种：C、Python，默认为 C，因为 C 引擎解析速度更快，但是特性没有 Python
引擎高。如果使用 C 引擎没有的特性时，会自动退化为 Python 引擎。

#### **converters**

converters 实现对列数据的变化操作，如下所示：

    
    
    In [21]: df = pd.read_csv('test.csv',sep='\s+')
    
    In [22]: df
    Out[22]:
       id name  age
    0   1   gz   10
    1   2   lh   12
    
    
    In [19]: df = pd.read_csv('test.csv',sep='\s+',converters={'age':lambda x:1+int(x)})
    
    In [20]: df
    Out[20]:
       id name  age
    0   1   gz   11
    1   2   lh   13
    

完成对 age 列的数据加 1，注意 int(x)，此处解析器默认所有列的类型为 str，所以需要显示类型转换。

#### **true_values**

true_values 参数指定数据中哪些字符应该被清洗为 True，false_values 参数指定哪些字符被清洗为 False。

如下所示，修改原数据文件 label 列的值为：

    
    
    In [1]: import pandas as pd
    
    In [2]: d = {'id':[1,2],'name':['gz','lh'],'age':[10,12],'label':['YES','NO']}
    
    In [3]: df = pd.DataFrame(d)
    
    In [4]: df.to_csv('test_label.csv',index=False)
    In [6]: df = pd.read_csv('test_label.csv')
    Out[6]:
       id name  age label
    0   1   gz   10   YES
    1   2   lh   12    NO
    

现在，想转化 YES 为 True，NO 为 False。这样使用参数：

    
    
    In [9]: df2 = pd.read_csv('test_label.csv',true_values=['YES'],false_values=['NO'])
    
    In [10]: df2
    Out[10]:
       id name  age  label
    0   1   gz   10   True
    1   2   lh   12 False
    

#### **skip_rows**

skip_rows 过滤行，数据文件如下。过滤掉 index 为 0 的行，使用 skip_rows：

    
    
    In [17]: df = pd.read_csv('test.csv',sep='\s+')
    
    In [18]: df
    Out[18]:
       id name  age
    0   1   gz   10
    1   2   lh   12
    
    In [19]: df = pd.read_csv('test.csv',sep='\s+',skiprows=[0])
    
    In [20]: df
    Out[20]:
       1  gz  10
    0  2  lh  12
    
    In [21]: df = pd.read_csv('test.csv',sep='\s+',header=None, skiprows=[0])
    
    In [22]: df
    Out[22]:
       0   1   2
    0  1  gz  10
    1  2  lh  12
    

但是，我们一般不想过滤掉列名称这一行，所以过滤掉一般从 1 开始：

    
    
    In [23]: df = pd.read_csv('test.csv',sep='\s+',skiprows=[1])
    
    In [24]: df
    Out[24]:
       id name  age
    0   2   lh   12
    

#### **skip_footer**

skip_footer 从文件末尾过滤行，解析器退化为 Python。这是因为 C 解析器没有这个特性。

    
    
    In [25]: df = pd.read_csv('test.csv',sep='\s+',skipfooter=1)
    
    In [26]: df
    Out[26]:
       id name  age
    0   1   gz   10
    

#### **nrows**

nrows 参数设置一次性读入的文件行数，它在读入大文件时很有用，比如 16G 内存的PC无法容纳几百 G 的大文件。

此参数可以结合 skiprows 使用，比如我想从原始文件的第 2 行（文件第一行为列名）开始一次读入 500 行：

    
    
    df = pd.read_csv('test.csv',sep='\s+',nrows=500) 
    

这样每次读取一个文件片（chunk），直到处理完成整个文件。

解析框架的其他两个参数 low_memory、memory_map 是布尔型变量，不再详细解释。

### 空值处理相关参数

#### **na_values**

na_values 参数可以配置哪些值需要处理成 Na/NaN，类型为字典，键指明哪一列，值为看做 Na/NaN 的字符。

假设我们的数据文件如下，date 列中有一个 `#` 值，我们想把它处理成 NaN 值。

    
    
    In [1]: d = {'id':[1,2],'name':['gz','lh'],'age':[10,12],'date':['2020-03-10','#']}
    In [2]: df = pd.DataFrame(d)
    In [3]: df.to_csv('test_date.csv',sep=' ',index=False)
    
    In [4]: df = pd.read_csv('test_date.csv',sep='\s+')    
    

可以使用 na_values 实现：

    
    
    In [37]:  df = pd.read_csv('test_date.csv',sep='\s+',na_values=['#'])
    
    In [38]: df
    Out[38]:
       id name  age        date
    0   1   gz   10  2020-03-10
    1   2   lh   12         NaN
    

keep_default_na 是和 na_values 搭配的，如果前者为 True，则 na_values 被解析为 Na/NaN
的字符除了用户设置外，还包括默认值。

#### **skip_blank_lines**

skip_blank_lines 默认为 True，则过滤掉空行，如为 False 则解析为 NaN。如下：

    
    
    In [39]: df = pd.read_csv('test_date.csv',sep='\s+',skip_blank_lines=False)
    
    In [40]: df
    Out[40]:
       id name  age        date
    0   1   gz   10  2020-03-10
    1   2   lh   12           #
    

#### **verbose**

打印一些重要信息，如下：

    
    
    In [55]: df = pd.read_csv('test.csv',sep='\s+',header=0,verbose=True)           
    Tokenization took: 0.02 ms
    Type conversion took: 0.88 ms
    Parser memory cleanup took: 0.01 ms
    

### 时间处理相关参数

#### **parse_dates**

如果导入的某些列为时间类型，但是导入时没有为此参数赋值，导入后就不是时间类型，如下：

    
    
    In [41]: df = pd.read_csv('test_date.csv',sep='\s+',header=0,na_values=['#'])
    
    In [42]: df
    Out[42]:
       id name  age        date
    0   1   gz   10  2020-03-10
    1   2   lh   12         NaN
    
    In [43]: df.dtypes
    Out[43]:
    id       int64
    name    object
    age      int64
    date    object
    dtype: object
    

date 列此时类型为 object，想办法转化为时间型：

    
    
    In [50]: df = pd.read_csv('test_date.csv',sep='\s+',header=0,na_values=['#'],parse_dates=['date'])
    
    In [51]: df
    Out[51]:
       id name  age       date
    0   1   gz   10 2020-03-10
    1   2   lh   12        NaT
    
    In [52]: df.dtypes
    Out[52]:
    id               int64
    name            object
    age              int64
    date    datetime64[ns]
    dtype: object
    

#### **date_parser**

date_parser 参数定制某种时间类型，详细使用过程总结如下。如果时间格式转化成标准的年月日，操作如下：

    
    
    In [58]: d = {'id':[1,2],'name':['gz','lh'],'age':[10,12],'date':['2020-03-10','2020-03-12']}
    
    In [59]: df = pd.DataFrame(d)
    
    In [60]: df.to_csv('test_date2.csv',sep=' ',index=False)
    
    In [61]: df = pd.read_csv('test_date2.csv',sep='\s+',parse_dates=['date'],date_parser= lambda dates: pd.datetime.strpti
        ...: me(dates,'%Y-%m-%d'))
    
    In [62]: df
    Out[62]:
       id name  age       date
    0   1   gz   10 2020-03-10
    1   2   lh   12 2020-03-12
    
    In [63]: df.dtypes
    Out[63]:
    id               int64
    name            object
    age              int64
    date    datetime64[ns]
    dtype: object
    

#### **infer_datetime_format**

infer_datetime_format 参数默认为 False。

如果设定为 True 并且 parse_dates 可用，那么 Pandas 将尝试转换为日期类型，如果可以转换，转换方法并解析，在某些情况下会快 5~10
倍。

### 分块读入相关参数

分块读入内存，尤其单机处理大文件时会很有用。

#### **iterator**

iterator 取值 boolean，default False，返回一个 TextFileReader 对象，以便逐块处理文件。

这个在文件很大时，内存无法容纳所有数据文件，此时分批读入，依次处理。

具体操作演示如下，我们的文件数据域一共有 2 行。

先读入一行，get_chunk 设置为 1 表示一次读入一行：

    
    
    In [64]: chunk = pd.read_csv('test.csv',sep='\s+',iterator=True)
    
    In [65]: chunk.get_chunk(1)
    Out[65]:
       id name  age
    0   1   gz   10
    

再读入下一行：

    
    
    In [66]: chunk.get_chunk(1)
    Out[66]:
       id name  age
    1   2   lh   12
    

此时已到文件末尾，再次读入会报异常：

    
    
    In [108]: chunk.get_chunk(1)  
    
    StopIteration
    

#### **chunksize**

chunksize 整型，默认为 None，设置文件块的大小。

如下，一次读入 2 行：

    
    
    In [68]: chunk = pd.read_csv('test.csv',sep='\s+',chunksize=2)
    
    In [69]: chunk
    Out[69]: <pandas.io.parsers.TextFileReader at 0x1ca02c6c608>
    
    In [70]: chunk.get_chunk()
    Out[70]:
       id name  age
    0   1   gz   10
    1   2   lh   12
    

### 格式和压缩相关参数

#### **compression**

compression 参数取值为 {‘infer’, ‘gzip’, ‘bz2’, ‘zip’, ‘xz’, None}，默认
‘infer’，直接使用磁盘上的压缩文件。

如果使用 infer 参数，则使用 gzip、bz2、zip 或者解压文件名中以 ‘.gz’、‘.bz2’、‘.zip’ 或 ‘xz’
这些为后缀的文件，否则不解压。

如果使用 zip，那么 ZIP 包中必须只包含一个文件。设置为 None 则不解压。

手动压缩本文一直使用的 test.csv 为 test.zip 文件，然后打开：

    
    
    In [73]:  df = pd.read_csv('test.zip',sep='\s+',compression='zip')
    
    In [74]: df
    Out[74]:
       id name  age
    0   1   gz   10
    1   2   lh   12
    

#### **thousands**

str，default None，千分位分割符，如 `,` 或者 `.`。

如下，显示数据文件 test.csv：

    
    
    In [122]: cat test.csv                                                          
    id  id  age  label  date
    1  'gz'  10  YES  1,090,001
    2  'lh'  12  NO  20,010
    

如果显示指定 thousands 为 `,`，则读入后 date 列显示为正常的整型。

    
    
    In [128]: df = pd.read_csv('***.csv',sep='\s+',thousands=',') 
    
    In [132]: df                                                                    
    Out[132]: 
       id  id.1  age label     date
    0   1  'gz'   10   YES  1090001
    1   2  'lh'   12    NO    20010
    
    
    In [130]: df['date'].dtypes                                                     
    Out[130]: dtype('int64')
    

#### **encoding**

encoding 指定字符集类型，通常指定为 'utf-8'。List of Python standard encodings。

#### **error_bad_lines**

如果一行包含过多的列，那么默认不会返回 DataFrame，如果设置成 False，那么会将该行剔除（只能在C解析器下使用）。

我们有意修改 test.csv 文件某个单元格的取值（带有两个空格，因为我们的数据文件默认分隔符为两个空格）：

    
    
    In [148]: cat test.csv                                                          
    id  id  age  label  date
    1  'gz'  10.8  YES  1,090,001
    2  'lh'  12.31  NO  O  20,010
    

此时，读入数据文件，会报异常：

    
    
    ParserError: Error tokenizing data. C error: Expected 5 fields in line 3, saw 6
    

在小样本读取时，这个错误很快就能发现。

但是在读取大数据文件时，假如读取 3 个小时，最后几行出现了这类错误，就很闹心！

所以稳妥起见，一般读取大文件时，会将 error_bad_lines 设置为 False，也就是剔除此行，同时使用 warn_bad_lines
设置为True，打印剔除的这行。

    
    
    In [150]: df = pd.read_csv('***.csv',sep='\s+',error_bad_lines=False)          
    b'Skipping line 3: expected 5 fields, saw 6\n'
    
    In [151]: df               
    Out[151]: 
       id  id.1   age label       date
    0   1  'gz'  10.8   YES  1,090,001
    

可以看到输出的警告信息：

> Skipping line 3: expected 5 fields, saw 6

#### **warn_bad_lines**

如果 error_bad_lines 设置为 False，并且 warn_bad_lines 设置为 True，那么所有的“bad
lines”将会被输出（只能在 C 解析器下使用）。

以上就是读 CSV 文件的所有参数及对应演示。

### 案例分析

对于动辄就几十或几百个 G 的数据，在读取的这么大数据的时候，我们有没有办法随机选取一小部分数据，然后读入内存，快速了解数据和开展 EDA？

使用 Pandas 的 skiprows 和概率知识，就能做到。下面解释具体怎么做。

如下所示，读取某 100G 大小的 big_data.csv 数据：

  1. 使用 skiprows 参数；
  2. x>0 确保首行读入； 
  3. np.random.rand()>0.01 表示 99% 的数据都会被随机过滤掉。

言外之意，只有全部数据的 1% 才有机会选入内存中。

    
    
    import pandas as pd
    import numpy as np
    
    df = pd.read_csv("big_data.csv", 
    skiprows = 
    lambda x: x>0 and np.random.rand() > 0.01)
    
    print("The shape of the df is {}. 
    It has been reduced 100 times!".format(df.shape))
    

使用这种方法，读取的数据量迅速缩减到原来的 1%，对于迅速展开数据分析有一定的帮助。

### 小结

今天与大家，一起探索 Pandas 读取 CSV 文件的 30 个实用参数，尤其在处理大数据文件时，有些参数的价值就会更加显现。

参数分为六大类，做了详细解读：

  * 基本参数
  * 通用解析类参数
  * 空值处理类参数
  * 时间处理类参数
  * 分块读入类参数
  * 格式和压缩类参数

