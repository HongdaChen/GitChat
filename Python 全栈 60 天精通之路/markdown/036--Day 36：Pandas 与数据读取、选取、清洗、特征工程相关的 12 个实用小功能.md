### 读取时抽样 1%

对于动辄就几十或几百个 G 的数据，在读取这么大数据时，我们有没有办法随机选取一小部分数据，然后读入内存，快速了解数据和开展 EDA？

使用 Pandas 的 skiprows 和概率知识，就能做到，下面解释具体怎么做。

如下所示，读取某 100 G 大小的 big_data.csv 数据：

  1. 使用 skiprows 参数
  2. `x>0` 确保首行读入
  3. `np.random.rand()>0.01` 表示 99% 的数据都会被随机过滤掉

言外之意，只有全部数据的 1% 才有机会选入内存中。

    
    
    import pandas as pd
    import numpy as np
    
    df = pd.read_csv("big_data.csv", 
    skiprows = lambda x: x>0 and 
    np.random.rand() > 0.01)
    
    print("The shape of the df is {}. 
    It has been reduced 100 times!".format(df.shape))
    

使用这种方法，读取的数据量迅速缩减到原来的 1%，对于迅速展开数据分析有一定的帮助。

### 生成时间序列的数据集

与时间序列相关的问题，平时挺常见。如何用 Pandas 快速生成时间序列数据？使用 pd.util.testing.makeTimeDataFrame
只需要一行代码，便能生成一个 index 为时间序列的 DataFrame：

    
    
    import pandas as pd
    
    pd.util.testing.makeTimeDataFrame(10)
    

结果：

    
    
    A    B   C   D
    2000-01-03    0.932776    -1.509302   0.285825    0.941729
    2000-01-04    0.565230    -1.598449   -0.786274   -0.221476
    2000-01-05    -0.152743   -0.392053   -0.127415   0.841907
    2000-01-06    1.321998    -0.927537   0.205666    -0.041110
    2000-01-07    0.324359    1.512743    0.553633    0.392068
    2000-01-10    -0.566780   0.201565    -0.801172   -1.165768
    2000-01-11    -0.259348   -0.035893   -1.363496   0.475600
    2000-01-12    -0.341700   -1.438874   -0.260598   -0.283653
    2000-01-13    -1.085183   0.286239    2.475605    -1.068053
    2000-01-14    -0.057128   -0.602625   0.461550    0.033472
    

时间序列的间隔还能配置，默认的 A、B、C、D 四列也支持配置。

    
    
    import numpy as np
    
    df = pd.DataFrame(np.random.randint(1,1000,size=(10,3)),columns = ['商品编码','商品销量','商品库存'])
    df.index = pd.util.testing.makeDateIndex(10,freq='H')
    

结果：

    
    
        商品编码    商品销量    商品库存
    2000-01-01 00:00:00    99  264 98
    2000-01-01 01:00:00    294 406 827
    2000-01-01 02:00:00    89  221 931
    2000-01-01 03:00:00    962 153 956
    2000-01-01 04:00:00    538 46  374
    2000-01-01 05:00:00    226 973 750
    2000-01-01 06:00:00    193 866 7
    2000-01-01 07:00:00    300 129 474
    2000-01-01 08:00:00    966 372 835
    2000-01-01 09:00:00    687 493 910
    

### 使用 apply(type) 做类型检查

有时肉眼所见的列类型，未必就是你预期的类型，如下 DataFrame 销量这一列，看似为浮点型。

![](https://images.gitbook.cn/6713c970-7233-11ea-83c3-675e02f948ee)

实际上，我们是这样生成 DataFrame 的：

    
    
    d = {"商品":["A", "B", "C", "D", "E"], "销量":[100, "100", 50, 550.20, "375.25"]}
    df = pd.DataFrame(d)
    df
    

所以直接在销量这一列上求平均值，就会报错。

    
    
    df['销量'].sum()
    
    
    
    TypeError: unsupported operand type(s) for +: 'int' and 'str'
    

所以在计算前，检查此列每个单元格的取值类型就很重要。

    
    
    df['销量'].apply(type)
    
    
    
    0      <class 'int'>
    1      <class 'str'>
    2      <class 'int'>
    3    <class 'float'>
    4      <class 'str'>
    Name: 销量, dtype: object
    

上面是打印结果，看到取值有 int、str、float。有的读者就问了，这才几行，如果上千上百行，也有一个一个瞅吗？当然不能，使用 value_counts
统计下各种值的频次。

    
    
    df['销量'].apply(type).value_counts()
    
    
    
    <class 'int'>      2
    <class 'str'>      2
    <class 'float'>    1
    Name: 销量, dtype: int64
    

### 标签和位置选择数据

读入数据，本专栏使用到的数据集都会打包发给各位读者。

    
    
    df = pd.read_csv("drinksbycountry.csv", index_col="country")
    df
    

![](https://images.gitbook.cn/874bc5d0-7233-11ea-bf44-ef79e0369ea9)

使用位置，iloc 选择前 10 行数据：

    
    
    df.iloc[:10, :]
    

![](https://images.gitbook.cn/97ba50d0-7233-11ea-964d-61a29639fe46)

标签和 loc 选择两列数据：

    
    
    df.iloc[:10, :].loc[:, "spirit_servings":"wine_servings"]
    

![](https://images.gitbook.cn/a80eeb30-7233-11ea-83c3-675e02f948ee)

也可以直接使用 iloc，得到结果如上面一样。

    
    
    df.iloc[:10, 1:3]
    

### Pandas 空值检查

实际使用的数据，null 值在所难免。如何快速找出 DataFrame 所有列的 null 值个数？

使用 Pandas 能非常方便实现，只需下面一行代码：

    
    
    data.isnull().sum()
    

  * data.isnull()：逐行逐元素查找元素值是否为 null。
  * .sum()：默认在 axis 为 0 上完成一次 reduce 求和。

如下 DataFrame：

![](https://images.gitbook.cn/d0846ea0-7233-11ea-b969-df66a2c2894a)

检查 null 值：

    
    
    data.isnull().sum()
    

结果：

    
    
    PassengerId      0
    Survived         0
    Pclass           0
    Name             0
    Sex              0
    Age            177
    SibSp            0
    Parch            0
    Ticket           0
    Fare             0
    Cabin          687
    Embarked         2
    dtype: int64
    

Age 列 177 个 null 值，Cabin 列 687 个 null 值，Embarked 列 2 个 null 值。

### replace 做清洗

Pandas 的强项在于数据分析，自然就少不了对数据清洗的支持。一个快速清洗数据的小技巧，在某列上使用 replace 方法和正则，快速完成值的清洗。

源数据：

    
    
    d = {"customer": ["A", "B", "C", "D"], 
    "sales":[1100, "950.5", "$400", " $1250.75"]}
    
    df = pd.DataFrame(d)
    df
    

打印结果：

    
    
    customer    sales
    0    A   1100
    1    B   950.5
    2    C   $400
    3    D   $1250.75
    

看到 sales 列的值，有整型，还有美元 + 整型、美元 + 浮点型。

我们的目标：清洗掉 `$` 符号，字符串型转化为浮点型。

一行代码搞定：（点击代码区域，向右滑动，查看完整代码）

    
    
    df["sales"] = df["sales"].replace("[$]", "", regex = True).astype("float")
    

使用正则替换，将要替换的字符放到列表中 `[$]`，替换为空字符，即 `""`；最后使用 astype 转为 float。

打印结果：

    
    
    customer    sales
    0    A   1100.00
    1    B   950.50
    2    C   400.00
    3    D   1250.75
    

如果不放心，再检查下值的类型：

    
    
    df["sales"].apply(type)
    

打印结果：

    
    
    0    <class 'float'>
    1    <class 'float'>
    2    <class 'float'>
    3    <class 'float'>
    

### 转 datetime

已知年和 dayofyear，怎么转 datetime？

原 DataFrame：

    
    
    d = {\
    "year": [2019, 2019, 2020],
    "day_of_year": [350, 365, 1]
    }
    df = pd.DataFrame(d)
    df
    

打印结果：

    
    
      year    day_of_year
    0    2019    350
    1    2019    365
    2    2020    1
    

下面介绍如何转 datetime 的 trick。

**Step 1：创建整数**

    
    
    df["int_number"] = df["year"]*1000 + df["day_of_year"]
    df
    

打印结果：

    
    
    year    day_of_year int_number
    0    2019    350 2019350
    1    2019    365 2019365
    2    2020    1   2020001
    

**Step 2：to_datetime**

    
    
    df["date"] = pd.to_datetime(df["int_number"], format = "%Y%j")
    df
    

注意 `"%Y%j"` 中转化格式 j，正是对应 dayofyear 参数。

打印结果：

    
    
        year    day_of_year int_number  date
    0    2019    350 2019350 2019-12-16
    1    2019    365 2019365 2019-12-31
    2    2020    1   2020001 2020-01-01
    

### 小分类值的替换

分类中出现次数较少的值，如何统一归为 others，该怎么做到？这也是我们在数据清洗、特征构造中经常会面临的一个任务。

如下 DataFrame：

    
    
    d = {"name":['Jone','Alica','Emily','Robert','Tomas','Zhang','Liu','Wang','Jack','Wsx','Guo'],
         "categories": ["A", "C", "A", "D", "A", "B", "B", "C", "A", "E", "F"]}
    df = pd.DataFrame(d)
    df
    

结果：

    
    
        name    categories
    0    Jone    A
    1    Alica   C
    2    Emily   A
    3    Robert  D
    4    Tomas   A
    5    Zhang   B
    6    Liu B
    7    Wang    C
    8    Jack    A
    9    Wsx E
    10    Guo F
    

D、E、F 仅在分类中出现一次，A 出现次数较多。

**步骤 1：统计频次，并归一化**

    
    
    frequencies = df["categories"].value_counts(normalize = True)
    frequencies
    

结果：

    
    
    A    0.363636
    B    0.181818
    C    0.181818
    F    0.090909
    E    0.090909
    D    0.090909
    Name: categories, dtype: float64
    

**步骤 2：设定阈值，过滤出频次较少的值**

    
    
    threshold = 0.1
    small_categories = frequencies[frequencies < threshold].index
    small_categories
    

结果：

    
    
    Index(['F', 'E', 'D'], dtype='object')
    

**步骤 3：替换值**

    
    
    df["categories"] = df["categories"].replace(small_categories, "Others")
    

替换后的 DataFrame：

    
    
        name    categories
    0    Jone    A
    1    Alica   C
    2    Emily   A
    3    Robert  Others
    4    Tomas   A
    5    Zhang   B
    6    Liu B
    7    Wang    C
    8    Jack    A
    9    Wsx Others
    10    Guo Others
    

### 重新排序列

某些场景需要重新排序 DataFrame 的列，如下 DataFrame：

![](https://images.gitbook.cn/6cab6860-7234-11ea-bf44-ef79e0369ea9)

如何将列快速变为：

![](https://images.gitbook.cn/79421060-7234-11ea-88af-f7ef8761fe69)

先构造数据：

    
    
    df = pd.DataFrame(np.random.randint(0,20,size=(5,7)),columns=list('ABCDEFG'))
    df
    

![](https://images.gitbook.cn/86cc6730-7234-11ea-bf44-ef79e0369ea9)

方法 1，直接了当：

    
    
    df2 = df[["A", "C", "D", "F", "E", "G", "B"]]
    df2
    

结果：

![](https://images.gitbook.cn/a12c1cb0-7234-11ea-833b-93fabc8068c9)

方法 2，也了解下：

    
    
    cols = df.columns[[0, 2 , 3, 5, 4, 6, 1]]
    df3 = df[cols]
    df3
    

也能得到方法 1 的结果。

### 时间数据下采样

步长为小时的时间序列数据，有没有小技巧，快速完成下采样，采集成按天的数据呢？

先生成测试数据：

    
    
    import pandas as pd
    import numpy as np
    
    df = pd.DataFrame(np.random.randint(1,10,
    size=(240,3)),
    columns = ['商品编码','商品销量','商品库存'])
    
    df.index = pd.util.testing.makeDateIndex(240,freq='H')
    df
    

生成 240 行步长为小时间隔的数据：

![](https://images.gitbook.cn/bee86ab0-7234-11ea-af73-378397689c85)

使用 resample 方法，合并为天（D）：

    
    
    day_df = df.resample("D")["商品销量"].sum().to_frame()
    day_df
    

结果如下，10 行、240 小时，正好为 10 天：

![](https://images.gitbook.cn/d8934f20-7234-11ea-bf44-ef79e0369ea9)

### 11 map 做特征工程

DataFrame 上快速对某些列展开特征工程，如何做到？

先生成数据：

    
    
    d = {
    "gender":["male", "female", "male","female"], 
    "color":["red", "green", "blue","green"], 
    "age":[25, 30, 15, 32]
    }
    
    df = pd.DataFrame(d)
    df
    

![](https://images.gitbook.cn/40efdbe0-db96-11ea-9369-b5210af59199)

在 gender 列上做如下映射：

    
    
    d = {"male": 0, "female": 1}
    df["gender2"] = df["gender"].map(d) 
    

![](https://images.gitbook.cn/5160b0d0-db96-11ea-b89a-ffa3ff92b2c1)

### 结合使用 where 和 isin

如下 DataFrame:

    
    
    d = {"genre": ["A", "C", "A", "A", "A", "B", "B", "C", "D", "E", "F"]}
    df = pd.DataFrame(d)
    df
    
    
    
    genre
    0    A
    1    C
    2    A
    3    A
    4    A
    5    B
    6    B
    7    C
    8    D
    9    E
    10    F
    

除了 3个出现频率最高的值外，统一替换其他值为 others.

首先找到出现频率最高的 3 个值：

    
    
    top3 = df["genre"].value_counts().nlargest(3).index
    top3
    

结果：

    
    
    Index(['A', 'C', 'B'], dtype='object')
    

使用 isin:

    
    
    df_update = df.where(df["genre"].isin(top3), other = "others")
    df_update
    

使用 where, 不满足第一个参数条件的，都会被替换为 others.

结果：

    
    
        genre
    0    A
    1    C
    2    A
    3    A
    4    A
    5    B
    6    B
    7    C
    8    others
    9    others
    10    others
    

结合使用 where, isin 做数据工程，特征清洗非常方便。

### 小结

今天介绍了 12 个 Pandas 实用的小功能：

  * 读取数据使用 `skiprows`
  * 生成时间序列的简便方法，apply(type) 做类型检查
  * 标签和位置选择数据
  * 8 个关于数据清洗，特征工程的实用小功能

aaaaaaaaaaaaaa

